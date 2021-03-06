# Using the PostgreSQL Database Secrets Engine with Vaultron

The following mini-guide shows how to set up Vaultron with a PostgreSQL Docker
container to use the Vault PostgreSQL secrets engine.

The guide presumes that you have formed Vaultron, initialized and unsealed
your Vault, and logged in with the initial root token.


## Run PostgreSQL Docker Container

Use the official PostgreSQL Docker container:

```
$ docker run \
  -p5432:5432 \
  --name vaultron_postgres \
  -e POSTGRES_PASSWORD=vaultron \
  -d postgres
```

Determine the PostgreSQL Docker container's internal IP address:

```
$ docker inspect \
    --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
    vaultron_postgres
172.17.0.2
```

## Configure Vault

Vaultron enables the database secrets engine at `vaultron_database` if using `blazing sword`; if you set up manually, you'll need to enable it:

```
$ vault secrets enable -path=vaultron_database database
```

Next, configure a simple PostgreSQL connection without SSL:

```
$ vault write vaultron-database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="postgresql-readonly" \
    connection_url="postgresql://postgres:vaultron@172.17.0.2:5432?sslmode=disable"


The following warnings were returned from the Vault server:
* Read access to this endpoint should be controlled via ACLs as it will return the connection details as is, including passwords, if any.
```

Write an initial PostgreSQL read only user role:

```
$ vault write vaultron-database/roles/postgresql-readonly \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
Success! Data written to: database/roles/postgresql-readonly
```

Retrieve a read only PostgreSQL database credential:

```
$ vault read vaultron-database/creds/postgresql-readonly
Key             Value
---             -----
lease_id        database/creds/readonly/ddc27039-ef66-a22b-c2f4-61fbfbbefd8a
lease_duration  1h0m0s
lease_renewable true
password        A1a-1p2p9yxwzsp51047
username        v-root-readonly-1r9s3w2qwzx3t2r0rzr0-1513092218
```

Log in to PostgreSQL container with read-only credential:

```
$ psql \
  --host=127.0.0.1 \
  --dbname=postgres \
  --username=v-root-readonly-1r9s3w2qwzx3t2r0rzr0-1513092218 \
  --password

# Use password 'A1a-1p2p9yxwzsp51047' from above
Password for user v-root-readonly-1r9s3w2qwzx3t2r0rzr0-1513092218:
psql (10.1)
Type "help" for help.

postgres=
```
