# app-kimai
Application stack to host [Kimai](https://www.kimai.org/)

## Getting started
Clone this repository and create a `docker-compose.override.yml`.

Configure the secrets via environment variables.

``` yaml
services:
  sqldb:
    environment:
      - MYSQL_PASSWORD=my-kimai-user-password
      - MYSQL_ROOT_PASSWORD=another-secret
  kimai:
    environment:
      - APP_SECRET=arandomgeneratedstringthatisverylong
      - "DATABASE_URL=mysql://kimai:my-kimai-user-password@sqldb/kimai?charset=utf8mb4&serverVersion=5.7.40"
```

Start the stack

``` bash
docker compose up -d
```

When deploying the app on a server, make sure to update the `TRUSTED_HOSTS` environment variable with the domain the app is hosted on.

E.g.

``` yaml
services:
  kimai:
    environment:
      - TRUSTED_HOSTS=kimai.redpencil.io,nginx,localhost,127.0.0.1
```

## How-to guides

### How to configure a backup using Borgmatic
To backup the stack, add the following configuration to Borgmatic:

``` yaml
archive_name_format: 'rpio-timesheet-app-kimai-{now}'

repositories:
    - path: "ssh://u1234-sub1@u1234.your-storagebox.de:23/./rpio-timesheet-app-kimai.borg"
      label: app-kimai

encryption_passphrase: "secretpassphrase"
ssh_command: ssh -i /root/.ssh/id_borgmatic

source_directories:
    - /data/app-kimai/docker-compose*.yml
    - /data/app-kimai/config
    - /data/app-kimai/data/kimai
    - /data/app-kimai/data/kimai-plugins

mysql_databases:
    - name: kimai
      hostname: localhost # command is executed via exec in app-kimai-sqldb-1 container
      username: root
      password: my-sql-root-password
      mysql_dump_command: docker exec --user=root app-kimai-sqldb-1 mysqldump --password=my-sql-root-password

skip_actions:
    - compact
    - pruneroot
```

Mount a shared volume in the `sqldb` service for Borgmatic to store the database dump:

``` yaml
services:
  sqldb:
    volumes:
      - /data/app-borgmatic/data/.borgmatic:/root/.borgmatic # shared with app-borgmatic to store mysqldump
```

### How to restore the SQL db from a backup
Extract the dump file from the Borgmatic archive

``` bash
docker compose exec borgmatic-restore borgmatic extract --archive latest --repository rpio-timesheet-app-kimai --destination /restore --path /root/.borgmatic/mysql_databases/localhost/kimai

```

Next, copy the file to the `./data/mysql-dump` folder and restore the dump using the `mysql` command in the `sqldb` container

``` bash
docker compose exec sqldb /bin/bash
> mysql -u root --password=my-sql-root-password -D kimai < /data/kimai
```

