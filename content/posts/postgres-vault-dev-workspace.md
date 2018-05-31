---
title: "Setting up a development workspace for Postgres and vault"
date: 2018-05-30
---

My current line of work has led me to start getting into docker more and more. One of the neat things I've been getting to play with is setting up and running a Postgres server locally for development, and connecting a vault instance to it for secrets management. I figured it was time to showcase what the basics of a development workspace looks like to help get someone get setup and running quickly.

I've hopefully boiled this down into the smallest amount of steps needed to make things work. This allows a new developer on a project to run `docker-compose up` and things will just be working in the background.

# Setting up Postgres with SSL

One of requirements I've had to fill is that I need a running Postgres instance with SSL turned on. To create these SSL files I've created this script to create all the necessary files:

```bash
#!/bin/bash
# Creates SSL certificates need for postgres

OUTPUT_FOLDER="$(dirname "$0")/../run/ssl-files"

mkdir -p $OUTPUT_FOLDER

OUTPUT_FOLDER="$(cd "$(dirname "$OUTPUT_FOLDER")"; pwd)/$(basename "$OUTPUT_FOLDER")"

echo "SSL files output folder [$OUTPUT_FOLDER]"

rm -rf $OUTPUT_FOLDER/*

OPENSSL_COMMAND="docker run --rm -it -w /out -v $OUTPUT_FOLDER:/out svagi/openssl"

echo -e "\n\n### Create CA ###"
$OPENSSL_COMMAND req -new -x509 -nodes -newkey rsa -out ca.pem -keyout ca-key.pem -set_serial 01 -batch -subj "/CN=ssl-server"
$OPENSSL_COMMAND rsa -in ca-key.pem -out ca-key.pem

echo -e "\n\n### Create postgres server key ###"
$OPENSSL_COMMAND req -new -newkey rsa -keyout server-key.pem -out server-req.pem -passout pass:abcd -subj "/CN=postgres-ssl-server"
$OPENSSL_COMMAND rsa -in server-key.pem -out server-key.pem -passin pass:abcd
$OPENSSL_COMMAND x509 -req -in server-req.pem -CA ca.pem -CAkey ca-key.pem -set_serial 02 -out server-cert.pem

echo -e "\n\n### Create client key ###"
$OPENSSL_COMMAND req -new -newkey rsa -keyout client-key.pem -out client-req.pem -passout pass:abcd -subj "/CN=postgres-ssl-server"
$OPENSSL_COMMAND rsa -in client-key.pem -out client-key.pem -passin pass:abcd
$OPENSSL_COMMAND x509 -req -in client-req.pem -CA ca.pem -CAkey ca-key.pem -set_serial 03 -out client-cert.pem

chmod 0600 $OUTPUT_FOLDER/*

```

This file assumes we are in a subfolder and will be creating a `run/ssl-files` folder above it and drops all the commands in there. I'm using Docker as the tool to bring in openssl command instead of getting it installed locally. 

# Configuring Postgres

The next script I have is to configure postgres correctly. This will add those SSL files and make sure only SSL connections are accepted. This looks like this:

```bash
#!/bin/sh

echo "Setting up postgres hba settings"
cat <<EOM > ${PGDATA}/pg_hba.conf
local     all  all       trust
hostnossl all  all  all  reject
hostssl   all  all  all  password
EOM

echo "Adding SSL settings to postgresql.conf file"
cat <<EOM >> ${PGDATA}/postgresql.conf
ssl = on
ssl_cert_file = '/var/ssl/server-cert.pem'
ssl_key_file = '/var/ssl/server-key.pem'
ssl_ca_file = '/var/ssl/ca.pem'
EOM
``` 

This script is obliviously intended to be run inside of the docker setup routine which we will get to in a minute. The `pg_hba.conf` file is to ensure that only ssl connections are accepted and the `postgresql.conf` file is adding the SSL settings needed for that.

Fairly straight forward so far, so let's see how the docker-compose settings are built. 

# Seeding vault

The script to seed vault looks like this:

```bash
#!/bin/sh

while ! vault status; do
  echo "Vault not up, sleeping for 1 second"
  sleep 1
done

echo "Enabling database plugin"
vault secrets enable -path=dbs database

echo "Adding postgres connection information"
vault write dbs/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url='postgresql://{{username}}:{{password}}@database:5432/mydb' \
  allowed_roles=mydb-admin \
  username="admin" \
  password="secret" \
  verify_connection=false

echo "Adding admin role"
vault write dbs/roles/mydb-admin \
  db_name=mydb \
  default_ttl=5m \
  max_ttl=1h \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' \
                         VALID UNTIL '{{expiration}}'; \
                         GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
```

# Docker-compose setup

The `docker-compose.yml` file that binds all of these together looks like this:

{{< highlight docker "linenos=table" >}}
version: '3'
services:
  database:
    image: postgres:9.6
    volumes:
      - ./run/ssl-files:/var/ssl/
      - ./scripts/setup-postgres:/docker-entrypoint-initdb.d/setup.sh
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydb
  vault:
      image: vault:0.10.1
      ports:
        - 8200:8200
      environment:
        - VAULT_DEV_ROOT_TOKEN_ID=secret-token
      depends_on:
        - database
  vault-seed:
      image: vault:0.10.1
      volumes:
        - ./scripts/seed-vault:/app/run.sh
      environment:
        - VAULT_ADDR=http://vault:8200/
        - VAULT_TOKEN=secret-token
      command: /app/run.sh
      depends_on:
        - vault
{{< / highlight >}}

Let's step through this line by line.

Line 1 & 2 are just normal Docker compose file format stuff.

The next 3 sections are the 3 different services we will be running. Postgres the database is defined in lines 3 - 13, the actual vault service is defined in 14 - 21 and the special vault seeding is last in line 22 - 31. 

The database section is fairly straight forward postgres setup. We add the SSL files to the docker container (line 6) and the script to be run on startup (line 7). We also expose the ports to other apps that I am developing can actually hit this when it is running (line 9). The last bits are environment variables to setup the initial database (line 11 - 13). Those last ones you can tweak to what ever you desire, these are just some defaults I've picked.

The vault section is fairly straight forward as well. The two things to point out are first, the environment variable with our dev token defined (line 19). This is the key to get into our vault instance to grab secrets out of it. And we make sure to say that we need the database up and running (line 21) for this.

The vault-seed section is needed to seed the vault with all of our settings. Vault, unlike the postgres image, doesn't provide a completely sane way of allowing a default configuration to be provided. You have to execute commands against the running instance to set things up correctly. So this seed image configured to point back to our vault image (line 27 - 28, note: did you know that there are DNS names based upon the definition of the docker container). And then we execute our seed command on line 29.
