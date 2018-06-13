---
title: "Setting up a development workspace for Postgres and vault"
date: 2018-05-30
---

My current line of work has led me to start getting into docker more and more. One of the neat things I've been getting to play with is using [Postgres](https://www.postgresql.org/) as my database and [Vault](https://www.vaultproject.io/) for secrets management. I figured it was time to showcase what a basic development workspace looks like to help get someone up and running quickly.

I've hopefully boiled this down into the smallest pieces needed to make things work. This allows a new developer on a project to run `docker-compose up` which will bring up a working development environment.

I will guide you first through some setup scripts that are needed, then tie them all together in the `docker-compose.yml` file.

# Setting up Postgres with SSL

One of requirements I've had to fill is that I need a running Postgres instance with SSL turned on. To create the needed SSL files I've created this script to create all the necessary files:

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

This file assumes we are in a subfolder and will be creating a `run/ssl-files` folder above it and drops all the commands in there. To make things easier and more portable, I'm using Docker as the tool to bring in openssl command instead of getting it installed locally. 

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

This script is intended to be run inside of the docker setup routine which we will get to in a minute. The `pg_hba.conf` file is to ensure that only ssl connections are accepted and the `postgresql.conf` file is adding the SSL settings needed for that.

This script will be run when we start up postgres and will be configured in our `docker-compose.yml` file. We will get to that in just a minute. 

# Seeding vault

The next script is related to Vault. This script will start vault then configure vault to be able to create secrets for our database. The script looks like this:

```bash
#!/bin/sh
# Run a vault dev server and configure that instance for the database

# Capture the token defined in the environment and make it our dev server token so they match
export VAULT_DEV_ROOT_TOKEN_ID=$VAULT_TOKEN

# Start vault server and place in background
vault server -dev &

# Wait for server to start
sleep 2

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
                         GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="REASSIGN OWNED BY \"{{name}}\" TO "admin"; DROP OWNED BY \"{{name}}\"; DROP ROLE \"{{name}}\";"

# Wait for vault server to finish
wait
```

This script is intended to be run when the vault container starts. This means that this script is responsible to ensure the server starts and stays running. Once that is complete, we then write out our configuration using the `vault` command. After that is all done, we let the script hang with the `vault server` portion with the `wait` command. This is because the script is what actually started the server and when this script ends, so will the docker container.  

One of the first bits in the script is to take `VAULT_TOKEN` variable that is passed and make that our dev server token. This token is how you access your vault instance. And this makes it easy to define one token and make sure that the running server has that same key for access later on. And just to be clear, we are running vault in dev mode. This allows us to define that token up front. When vault is ran in production, these tokens are generated on startup for you. 

The next interesting bits are the connection and role setup. One quick thing I want to point out that the paths follow a pattern. The name `dbs` comes from where we enabled the database plugin. There are then 3 sub paths of that, `config`, `roles` and `creds`. The `config` path is where the configuration to our database is stored. Next we add to the `roles` part which defines what roles can create credentials to our database with the config. Then there is a `creds` key that actually creates the credentials when it is read, more on using that later. 

To reiterate we store configuration in `dbs/config/mydb`, we store role information in `dbs/roles/mydb-admin`. This automatically creates a path at `dbs/creds/mydb-admin` that we will use later to get our credentials.

The postgres credentials and postgres is worth a blog post all together but let me talk about this really quick. Postgres roles can be tricky if you don't get them right. With vault, we are creating temporary roles that will come and go at will. When we use these credentials then to also manage the schema, we need to pay extra close attention on who actually owns the tables. When you create a table, who ever the role is that you are currently working as is the role that will be the owner of said table. Then, your next role will have issues actually accessing that table unless permissions and ownership have been setup correctly. I have a [dba.stackexchange](https://dba.stackexchange.com/q/208649/16141) question related this that will have a longer blog post later.

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
        - VAULT_TOKEN=secret-token
        - VAULT_ADDR=http://127.0.0.1:8200/
      volumes:
        - ./scripts/setup-and-run-vault:/setup-and-run.sh
      command: /setup-and-run.sh
{{< / highlight >}}

Let's step through this line by line.

Line 1 & 2 are just normal Docker compose file format stuff.

The next 3 sections are the 3 different services we will be running. Postgres the database is defined in lines 3 - 13 and the vault service is defined in 14 - 24.

The database section is a fairly straight forward postgres docker setup. We add the SSL files to the docker container (line 6) and the script to be run on startup (line 7). We also expose the ports to other apps that I am developing can actually hit this when it is running (line 9). The last bits are environment variables to setup the initial database (line 11 - 13). Those last ones you can tweak to what ever you desire, these are just some defaults I've picked.

The vault section is mostly straight forward. In the environment variables (starting line 20) we have two definitions, one for our token, another one for the address of the running vault instance. The first is the token we use for our dev server as well as accessing that server (remember: we define the token and use it in our script for the dev server). The address needs to be defined because by default vault will try and use https. Looks kind of silly that you need to define the local server, but that's alright. The next part is adding our script that will run and configure the Vault server. 

# Actually using this

Alright, we now have all of this crazy stuff, how exactly do I use any of this? Well, let's demonstrate that with running inside of this local project, not connecting any code. Just making sure things are running and that we can demonstrate they are running.

First, how do I start this up? Well run `docker-compose up`. This brings the whole system up. The whole point of what we did before with the seeding and such is that one command will give you a working system. If you go read through all of the output that gives you, you will see that the database was started and initialized, vault was started and initialized, and vault was seeded with our initial configuration. 

Next, let's go ahead and see how to connect to the database. One neat way is we can actually get connected to our running instance and execute `psql` through that. You can do that by `docker-compose exec database psql -U admin -d mydb`. If you ran that, you should now be inside of `psql` prompt that is connect to the database.  

However, that isn't the fun part with vault. What if I wanted to try out using the whole pipeline of vault creating a credential, then getting into the database with those credentials? Well, its a bit more complicated but here is a script that goes through that process:

```bash
VAULT_SETTINGS=$(docker-compose exec vault vault read --format=json dbs/creds/mydb-admin)
VAULT_USERNAME=$(echo $VAULT_SETTINGS | docker run --rm -i colstrom/jq -r '.data.username')
docker-compose exec database psql -U $VAULT_USERNAME -d mydb
``` 
The first part is capturing a vault token by executing `vault read` inside of one of the `vault-seed` container. This will create a JSON object that looks something like this:

```json
{
  "request_id": "92b63ef7-9ed1-c669-cb36-16ac8110f7c2",
  "lease_id": "dbs/creds/mydb-admin/af6bcdaf-b20e-4927-6eff-c5c93aef3e2e",
  "lease_duration": 300,
  "renewable": true,
  "data": {
    "password": "A1a-0684tz5r1zpz3vys",
    "username": "v-token-mydb-adm-29r3ryxwxs3ryrvy9010-1528801793"
  },
  "warnings": null
}
```

As you can see, this gives you a username and password that you could use to login to your postgres instance. This is a temporary credential that will expire after a default of 5 minutes. 

We take that `VAULT_SETTINGS` then (using Docker of course!) we extract the username from that JSON Object using the `jq` command. Notice how you can pipe data into a docker container. This makes docker an extremely useful tool in your toolkit because you can now use it like you would use it for normal Linux commands and create a nice data pipeline.

From there we then login with that username, and we can verify that by running `\c`.

So there you have it, a vault and postgres development workspace. I have the code over on [github](https://github.com/baens/docker-vault-postgres-dev-workspace) if you just want to download it and use it as you need it.
