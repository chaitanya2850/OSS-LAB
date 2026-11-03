# MantisBT -- Simple Docker Compose Setup

This setup keeps the MantisBT application files on the host and runs
Apache/PHP and MariaDB in Docker Compose.

## Architecture

-   **MantisBT:** application files remain on the host in `./mantis`.
-   **Apache + PHP:** runs in the `mantisbt` container and serves the
    mounted application directory.
-   **MariaDB:** runs in the `mariadb` container and stores MantisBT's
    data.
-   **Browser URL:** `http://localhost:8080/`

## Project structure

From `~/BOOTCAMP/mantisbt`:

``` text
mantisbt/
├── Dockerfile
├── docker-compose.yml
├── .env
└── mantis/
    ├── admin/
    ├── config/
    ├── core/
    ├── index.php
    └── ...
```

The `mantis/` directory must contain the MantisBT application files,
including `index.php`.

## 1. Dockerfile

Create `Dockerfile` in the project root:

``` dockerfile
FROM php:8.3-apache

# Install the PHP extensions needed by MantisBT
RUN apt-get update \
    && apt-get install -y --no-install-recommends libonig-dev \
    && docker-php-ext-install mysqli mbstring \
    && a2enmod rewrite \
    && rm -rf /var/lib/apt/lists/*
```

This installs the MySQLi and mbstring PHP extensions and enables
Apache's rewrite module.

## 2. Environment file

Create `.env` in the project root:

``` env
MARIADB_ROOT_PASSWORD=REPLACE_WITH_A_STRONG_ROOT_PASSWORD
MARIADB_PASSWORD=REPLACE_WITH_A_STRONG_APPLICATION_PASSWORD
```

Generate strong passwords, for example:

``` bash
openssl rand -hex 24
openssl rand -hex 24
```

Put the generated values into `.env`. The `MARIADB_PASSWORD` value must
also be used in MantisBT's `config_inc.php`.

Restrict access to the file:

``` bash
chmod 600 .env
```

Do not commit `.env` to source control.

## 3. Docker Compose

Create `docker-compose.yml`:

``` yaml
services:
  mantisbt:
    build: .
    container_name: mantisbt
    restart: unless-stopped

    ports:
      - "8080:80"

    volumes:
      - ./mantis:/var/www/html

    depends_on:
      mariadb:
        condition: service_healthy

  mariadb:
    image: mariadb:11.4
    container_name: mantisbt-db
    restart: unless-stopped

    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MARIADB_DATABASE: mantisbt
      MARIADB_USER: mantisuser
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}

    volumes:
      - mantisbt_db:/var/lib/mysql

    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mantisbt_db:
```

MariaDB is not published on a host port. MantisBT reaches it over the
Compose network using the hostname `mariadb`.

> **Existing database warning:** This Compose file declares a named
> volume called `mantisbt_db`. If you already have a database with data,
> retain the existing volume or bind mount configuration. Do not switch
> to a new volume unless you intend to start with an empty database. Do
> not run `docker compose down -v` unless you intentionally want to
> delete the database volume.

## 4. Start the containers

Run these commands from `~/BOOTCAMP/mantisbt`:

``` bash
docker compose up -d --build
docker compose ps
```

Check the PHP extensions:

``` bash
docker compose exec mantisbt php -m
```

Confirm that `mysqli` and `mbstring` appear in the output.

View logs:

``` bash
docker compose logs -f mantisbt
docker compose logs -f mariadb
```

## 5. MantisBT configuration file

The configuration file is:

``` text
mantis/config/config_inc.php
```

The installer may report that it cannot write this file because the
application directory is bind-mounted from the host. In that case,
create it on the host:

``` bash
nano ./mantis/config/config_inc.php
```

Preferably use the configuration which the web installer displays while the installation (it'll tell that it cannot create the files, so you need to create the above file using the config displayed. If now then and then only use the below)

Use this as the configuration, replacing the password and salt
placeholders with the values generated for your installation:

``` php
<?php

$g_hostname = 'mariadb';
$g_db_type = 'mysqli';
$g_database_name = 'mantisbt';
$g_db_username = 'mantisuser';
$g_db_password = 'REPLACE_WITH_THE_MARIADB_PASSWORD_FROM_ENV';

$g_default_timezone = 'Asia/Kolkata';

$g_crypto_master_salt = 'REPLACE_WITH_THE_CRYPTO_SALT_GENERATED_BY_THE_INSTALLER';

$g_path = 'http://localhost:8080/';
```

The database password must exactly match `MARIADB_PASSWORD` in `.env`.

If the installer displayed a generated crypto master salt, copy that
exact value into `$g_crypto_master_salt`. If you need to generate one
yourself, use a cryptographically secure random value, for example:

``` bash
openssl rand -base64 32
```

Then make the configuration file readable by the web server:

``` bash
chmod 644 ./mantis/config/config_inc.php
```

Keep this file private: it contains database credentials and the crypto
master salt. Back it up securely.

## 6. Run the web installer

Open the installer in your browser:

-   Installation check: <http://localhost:8080/admin/check/index.php>
-   Database installer: <http://localhost:8080/admin/install.php>

Use these database settings:

  -----------------------------------------------------------------------
  Installer field                     Value
  ----------------------------------- -----------------------------------
  Database type                       MySQL Improved

  Hostname                            `mariadb`

  Database username                   `mantisuser`

  Database password                   Value of `MARIADB_PASSWORD` in
                                      `.env`

  Database name                       `mantisbt`

  Admin username/password for         Leave blank; Compose has already
  database creation                   created the database and
                                      application user

  Default time zone                   Kolkata

  Installation URL                    `http://localhost:8080/`
  -----------------------------------------------------------------------

The installation URL is the address users enter in their browsers. It is
not the internal Docker hostname. Keep `localhost:8080` when using the
browser on the Docker host. If other computers will access MantisBT, use
the host machine's reachable IP address or DNS name instead.

The installer may warn that it cannot retrieve `http://localhost:8080/`
from inside the container. That check can fail because `localhost`
inside a container refers to that container itself. Keep the
browser-facing installation URL correct.

After installation, verify that the login page opens at
<http://localhost:8080/>.

If this is an existing installation with an existing database, do not
run the installer again.

## 7. Log in

For a standard fresh MantisBT installation, the initial administrator
login is commonly:

-   **Username:** `administrator`
-   **Password:** `root`

If you selected or created different credentials during setup, use those
instead. Change the default password immediately after the first login.

## 8. Secure the installation

After the installation is complete and verified, remove the
installer/admin directory from the host:

``` bash
rm -rf ./mantis/admin
```

Do not remove it before the installation is complete.

Also: - Keep `.env` and `config_inc.php` private. - Do not expose the
MariaDB port publicly. - Back up the database volume and MantisBT
configuration. - Use a strong administrator password.

## 9. Useful commands

  Task                   Command
  ---------------------- ---------------------------------------
  Start services         `docker compose up -d`
  Build and start        `docker compose up -d --build`
  Stop services          `docker compose down`
  Restart MantisBT       `docker compose restart mantisbt`
  Show service status    `docker compose ps`
  MantisBT logs          `docker compose logs -f mantisbt`
  MariaDB logs           `docker compose logs -f mariadb`
  Check PHP extensions   `docker compose exec mantisbt php -m`

**Database caution:** `docker compose down` keeps the named database
volume. `docker compose down -v` removes it and can permanently delete
the database data.
