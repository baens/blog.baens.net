---
title: "Setting a development environment with Vault and Postgres on Google Cloud SQL"
date: 2018-06-13
---

This blog post is going to build off [my previous one]() on setting up a development environment with [Postgres]() and [Vault](). This time though, we are going to tweak the environment for the Postgres to better match the behavior in [Google Cloud SQL version of Postgres](). Along with the basic tweaks to Postgres, we will also setup Vault to provide 3 different roles. 

# Phiolsphy of this all

Explanation of why we are setting up vault users and these layers. Also maybe touch on why docker-compose s

# Initial workspace 

Taking from my previous blog post, we will not walk through what was provided by the [last repository]. But instead we will add tweaks and improvements of things. So let's get started with Postgres and add what is needed there, then work with Vault.

# Initial postgres SQL

The first thing we are going to add is setting up the database with a few roles that Google SQL has. These roles will mimic what is given to use in Google SQL. We will also make sure the schema has the correct permissions and roles setup so we can simulate what it would be like if we were actually working against it.

This is what that initial SQL looks like:

```sql
ALTER ROLE postgres RENAME TO cloudsqladmin;
CREATE ROLE cloudsqlsuperuser WITH CREATEDB CREATEROLE;
ALTER DATABASE mydb OWNER TO cloudsqlsuperuser;
CREATE ROLE "postgres" WITH LOGIN CREATEDB CREATEROLE IN ROLE cloudsqlsuperuser;
``` 

Take note, I am using the database name of "mydb". This is not set in stone and can be changed. In my images that I use on a day to day basis, I actually have this configurable. It starts as "mydb" then I rename it as the last operation.  

Looking over this we will notice the default `postgres` role is renamed to `cloudsqladmin`. That is because the only super user role is that `cloudsqladmin` in Google's Postgres version and we want to simulate that. Next is the common role for everything, which is `cloudsqlsuperuser`. This is the role we are going to reassign back to everything. More on that later.

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

One thing I want to point out is the TTLs of these users. These are just for demonstration purposes. In production, these roles should last a few days and then have to be renewed after a few weeks. This is just to allow testing locally if the credential rotation works. 

# Migrations

Now that you have all of these fancy temporary roles to play with, let's investigate migrations. We need to pay extra close attention to this in Postgres. When a table, or any other object in the database for that matter, is created. The owner of that object is the role of that created it. Now that we have a temporary role that we are creating those objects, we need to pay attention to where the ownership of the objects ends up. What's nice about this is Postgres actually has a very clean way of cleaning this up. It actually looks like this:

```sql
REASSIGN OWNED BY current_user TO "cloudsqlsuperuser";
```

This command will reassign all of the objects just created back to the common role of `cloudsqlsuperuser`. This way, after the temporary admin role is gone, the objects are in the correct permissions. 

# Demo

Now that we have all of that together, I have put together a demo repository with all of the pieces together for demonstration purposes. This demo will setup all of the needed services, use a migration (using Flyway) on the database to get the schema where it needs to be, and allow you to connect with a Node service to demonstrate what is needed.

