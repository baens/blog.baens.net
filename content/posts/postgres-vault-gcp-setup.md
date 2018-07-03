---
title: "Setting a development environment with Vault and Postgres on Google Cloud SQL"
date: 2018-06-13
---

This blog post is going to build off my previous one of building up a development envionrment with vault and postgres. This time though, we are going to get a better targted environment for the postgres flavor that is on GCP. There are a few things we will need to tweak, as well as we are going to setup 3 roles that we can retrieve from vault. 

# Initial workspace 

Taking from my previous blog post, I will assume that all the settings are there to get a postgres and vault instance up and running. This means that vault and postgres are already talking and postgres has SSL setup.

# Initial postgres SQL

The first thing we are going to add is setting up the database with a few roles. These roles will mimic what is given to use in GCP. We will also make sure the schema has the correct permissions and roles setup so we can simulate what it would be like if we were actually working against it.

This is what that inital SQL looks like:

```sql
ALTER ROLE postgres RENAME TO cloudsqladmin;
CREATE ROLE cloudsqlsuperuser WITH CREATEDB CREATEROLE;
ALTER DATABASE mydb OWNER TO cloudsqlsuperuser;
CREATE ROLE "postgres" WITH LOGIN CREATEDB CREATEROLE IN ROLE cloudsqlsuperuser;
``` 

Now remember, I am using the database name of "mydb". This is not set in stone and can be changed. 


Looking over this we will notice the default `postgres` role is renamed to `cloudsqladmin`. That is because the only super user role is that `cloudsqladmin` in Google's Postgres versnion and we want to simulate that. Next is the common role for everything, which is `cloudsqlsuperuser`. This is the role we are going to reassign back to everything. More on that later.

# Vault Roles

Now we are going to setup our roles. Instead of just having one we are going to have three. The first role is `admin`. This role is used when tables and other database objects will need to be created. This role will primarily be used by the system we use to do our DDL changes. Next will be the `app` role. This role will be used with the actual application. It will only have select, insert, update, and delete permissions. Third, there is a `readonly` role. Certain aspects of an application may not need insert, update, or delete. So we create this 3rd role for parts of the application we know are only reading from the database.

Let's see what that looks like in our vault setup script

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
cat <<EOM | vault write dbs/config/mydb -
{
  "plugin_name": "postgresql-database-plugin",
  "allowed_roles": ["mydb-admin", "mydb-app", "mydb-readonly"],
  "connection_url": "postgresql://{{username}}:{{password}}@database:5432/mydb",
  "username": "admin",
  "password": "secret",
  "verify_connection": false
}
EOM

cat <<EOM | vault write dbs/roles/mydb-admin -
{
  "db_name":"mydb",
  "default_ttl":"5m",
  "max_ttl":"1h",
  "creation_statements":[
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'",
    "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\""
  ]
}
EOM

cat <<EOM | vault write dbs/roles/mydb-app -
{
  "db_name":"mydb",
  "default_ttl":"5m",
  "max_ttl":"1h",
  "creation_statements":[
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'",
    "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";",
    "GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\""
  ]
}
EOM

cat <<EOM | vault write dbs/roles/mydb-readonly -
{
  "db_name":"mydb",
  "default_ttl":"5m",
  "max_ttl":"1h",
  "creation_statements":[
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'",
    "GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\""
  ]
}
EOM

# Wait for vault server to finish
wait
```

# Migrations

Now that you have all of these fancy temporary roles to play with, the next big hurdle to cross in Postgres land is the playing of roles when creating tables. Because the admin role is temporary we have to take extra pre-caution to permissions. By default (at least in Postgres 9.6) what ever role creates the table is what is assigned to that table. That can create problems down the road because of our temporary role assignment. To mitigate that, what ever is doing our migrations needs to run this command:

```sql
REASSIGN OWNED BY current_user TO "cloudsqlsuperuser";
```

This command will reassign all of the objects just created back to the common role of `cloudsqlsuperuser`. This way, after the temporary admin role is gone, the objects are in the correct permissions. 

# Demo

Now that we have all of that together, I have put together a demo repository with all of the pieces together for demonstration purposes. This demo will setup all of the needed services, use a migration on the database to get the schema where it needs to be, and allow you to connect with a node service to demonstrate what is needed.
