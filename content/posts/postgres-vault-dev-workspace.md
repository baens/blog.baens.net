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

The next script we have is one that can seed vault with our configuration. This script will be setup to hit our docker-compose instance and write the needed configuration. The script to seed vault looks like this:

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
                         GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"\
  revocation_statements="REASSIGN OWNED BY \"{{name}}\" TO "admin"; DROP OWNED BY \"{{name}}\"; DROP ROLE \"{{name}}\";"
```

This script will go through 4 things: first it will verify that vault is up and running, then it will enable the database secrets plugin, then write the database plugin configuration, then the database role.

We need to verify that the database is up and running and I took the approach that we will use the vault command itself to verify that we can communicate with vault correctly.

The interesting bits for this are the connection and role setup. One quick thing I want to point out that the paths follow a pattern. The name `dbs` comes from where we enabled the database plugin. There are then 3 sub paths of that, `config`, `roles` and `creds`. The `config` part is fairly obvious. This is where the configuration to our database is stored. Next we add to the `roles` part which defines what roles can create credentials to our database with the config. Then there is a `creds` key that actually creates the credentials when it is read, more on using that later. 

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

The `vault` section is fairly straight forward as well. The two things to point out are first, the environment variable with our dev token defined (line 19). This is the key to get into our vault instance to grab secrets out of it. And we make sure to say that we need the database up and running (line 21) for this.

The `vault-seed` section is needed to seed the vault with all of our settings. Vault, unlike the postgres image, doesn't provide a completely sane way of allowing a default configuration to be provided. You have to execute commands against the running instance to set things up correctly. So this seed image configured to point back to our vault image (line 27 - 28, note: did you know that there are DNS names based upon the definition of the docker container). And then we execute our seed command on line 29.

# Actually using this

Alright, we now have all of this crazy stuff, how exactly do I use any of this? Well, let's demonstrate that with running inside of this local project (tying code to this will be an exercise for later blog posts). 

First, how do I start this up? Well run `docker-compose up`. This brings the whole system up. The whole point of what we did before with the seeding and such is that one command will give you a working system. If you go read through all of the output that gives you, you will see that the database was stood up, vault was stood up, and vault was seeded with our initial configuration. 

Next, let's go ahead and see how to connect to the database. One neat way is we can actually get connected to our running instance and execute `psql` through that. You can do that by `docker-compose exec database bash`. This will drop you into a bash shell running in your database container. From there you can run `psql -U admin -d mydb` and viola you are in the database. 

However, that isn't the fun part with vault. What if I wanted to try out using the whole pipeline of vault creating a credential, then getting into the database with those credentials? Well, its a bit more complicated but here is a script that goes through that process:

```bash
VAULT_SETTINGS=$(docker-compose run --no-deps --rm vault-seed vault read --format=json dbs/creds/mydb-admin)
VAULT_USERNAME=$(echo $VAULT_SETTINGS | docker run --rm -i colstrom/jq -r '.data.username')
docker-compose exec database psql -U $VAULT_USERNAME -d mydb
``` 
