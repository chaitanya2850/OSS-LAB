# MantisBT with Docker Compose and Mailpit

This guide documents the local setup with MantisBT source files on the host and Apache/PHP, MariaDB, and Mailpit running in Docker Compose.

## Architecture

- **MantisBT:** application files remain on the host in `./mantis`.
- **Apache + PHP:** serves MantisBT from the `mantisbt` container.
- **MariaDB:** stores MantisBT's database.
- **Mailpit:** captures outgoing email from MantisBT so you can test notifications and password-reset emails without sending real email.
- **MantisBT URL:** `http://localhost:8080/`
- **Mailpit inbox:** `http://localhost:8025/`

Mailpit is for local testing. Emails sent to it are captured in its web inbox; they are not delivered to real email addresses.

## Project structure

From `~/BOOTCAMP/mantisbt`:

```text
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

The `mantis/` directory must contain the MantisBT application files, including `index.php`.

## 1. Dockerfile

Create `Dockerfile` in the project root:

```dockerfile
FROM php:8.3-apache

# Install the PHP extensions needed by MantisBT
RUN apt-get update \
    && apt-get install -y --no-install-recommends libonig-dev \
    && docker-php-ext-install mysqli mbstring \
    && a2enmod rewrite \
    && rm -rf /var/lib/apt/lists/*
```

This installs the MySQLi and mbstring PHP extensions and enables Apache's rewrite module.

## 2. Environment file

Create `.env` in the project root:

```env
MARIADB_ROOT_PASSWORD=REPLACE_WITH_A_STRONG_ROOT_PASSWORD
MARIADB_PASSWORD=REPLACE_WITH_A_STRONG_APPLICATION_PASSWORD
```

Generate strong passwords, for example:

```bash
openssl rand -hex 24
openssl rand -hex 24
```

Put the generated values into `.env`. The `MARIADB_PASSWORD` value must also be used in MantisBT's `config_inc.php`.

Restrict access to the file:

```bash
chmod 600 .env
```

Do not commit `.env` to source control.

## 3. Docker Compose

Create `docker-compose.yml`:

```yaml
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

  mailpit:
    image: axllent/mailpit:latest
    container_name: mantisbt-mailpit
    restart: unless-stopped

    ports:
      - "8025:8025"

volumes:
  mantisbt_db:
```

All three services are on the same Compose network. MantisBT connects to MariaDB using hostname `mariadb` and to Mailpit using hostname `mailpit`. Mailpit's SMTP port `1025` is available to other containers on the Compose network; it does not need to be published to the host for this setup.

MariaDB is not published on a host port.

> **Existing database warning:** If you already have a database with data, retain your existing volume or bind-mount configuration. Do not switch to a new volume unless you intend to start with an empty database. Do not run `docker compose down -v` unless you intentionally want to delete the database volume.

## 4. Start the containers

Run these commands from `~/BOOTCAMP/mantisbt`:

```bash
docker compose up -d --build
docker compose ps
```

Check the PHP extensions:

```bash
docker compose exec mantisbt php -m
```

Confirm that `mysqli` and `mbstring` appear in the output.

Open the services:

- MantisBT: <http://localhost:8080/>
- Mailpit inbox: <http://localhost:8025/>

View logs:

```bash
docker compose logs -f mantisbt
docker compose logs -f mariadb
docker compose logs -f mailpit
```

## 5. MantisBT configuration file

The configuration file is:

```text
mantis/config/config_inc.php
```

If the installer cannot write this file because the application directory is bind-mounted from the host, create it on the host:

```bash
nano ./mantis/config/config_inc.php
```

Use this as the configuration, replacing the password and salt placeholders with the values for your installation:

```php
<?php

$g_hostname = 'mariadb';
$g_db_type = 'mysqli';
$g_database_name = 'mantisbt';
$g_db_username = 'mantisuser';
$g_db_password = 'REPLACE_WITH_THE_MARIADB_PASSWORD_FROM_ENV';

$g_default_timezone = 'Asia/Kolkata';

$g_crypto_master_salt = 'REPLACE_WITH_YOUR_GENERATED_CRYPTO_SALT';

$g_path = 'http://localhost:8080/';

/* Local email testing through Mailpit */
$g_phpMailer_method = PHPMAILER_METHOD_SMTP;
$g_smtp_host = 'mailpit';
$g_smtp_port = 1025;
$g_smtp_connection_mode = '';
$g_smtp_username = '';
$g_smtp_password = '';
$g_enable_email_notification = ON;
$g_send_reset_password = ON;
```

The database password must exactly match `MARIADB_PASSWORD` in `.env`.

Use the crypto master salt generated by the installer, if one was provided. Otherwise generate a secure random value, for example:

```bash
openssl rand -base64 32
```

Keep the same crypto master salt after installation. Do not casually change it later, because MantisBT may use it to protect stored data.

Make the file readable by the web server:

```bash
chmod 644 ./mantis/config/config_inc.php
```

Keep this file private: it contains database credentials and the crypto master salt. Back it up securely.

## 6. Run the web installer

Open these pages in your browser:

- Installation check: <http://localhost:8080/admin/check/index.php>
- Database installer: <http://localhost:8080/admin/install.php>

Use these database settings:

| Installer field | Value |
|---|---|
| Database type | MySQL Improved |
| Hostname | `mariadb` |
| Database username | `mantisuser` |
| Database password | Value of `MARIADB_PASSWORD` in `.env` |
| Database name | `mantisbt` |
| Database admin username/password | Leave blank; Compose has already created the database and application user |
| Default time zone | Kolkata |
| Installation URL | `http://localhost:8080/` |

The installation URL is the address users enter in their browsers, not the internal Docker hostname. If other computers will access MantisBT, use the host machine's reachable IP address or DNS name instead.

The installer may warn that it cannot retrieve `http://localhost:8080/` from inside the container. That check can fail because `localhost` inside a container refers to that container itself. Keep the browser-facing installation URL correct.

After installation, verify that the login page opens at <http://localhost:8080/>.

If this is an existing installation with an existing database, do not run the installer again.

## 7. Log in

For a standard fresh MantisBT installation, the initial administrator login is commonly:

- **Username:** `administrator`
- **Password:** `root`

If you selected or created different credentials during setup, use those instead. Change the default password immediately after the first login.

## 8. Test email with Mailpit

1. Make sure the `mailpit` service is running:

   ```bash
   docker compose ps
   ```

2. Confirm that the Mailpit settings shown in section 5 are present in `mantis/config/config_inc.php`.
3. Restart MantisBT so the configuration is reloaded:

   ```bash
   docker compose restart mantisbt
   ```

4. Open the Mailpit inbox at <http://localhost:8025/>.
5. Trigger an email from MantisBT, such as a password-reset email for a test account or an issue notification.
6. Refresh the Mailpit inbox. The captured message should appear there.

MantisBT must have email notifications enabled and a valid recipient address on the test account. Mailpit captures messages sent by the application but does not deliver them to external inboxes. Do not use this setup when you need real outbound email.

## 9. Secure the installation

After installation is complete and verified, remove the installer/admin directory from the host:

```bash
rm -rf ./mantis/admin
```

Do not remove it before installation is complete.

Also:
- Keep `.env` and `config_inc.php` private.
- Do not expose the MariaDB port publicly.
- Back up the database volume and MantisBT configuration.
- Use a strong administrator password.
- Mailpit's web interface is published on host port `8025`; keep it limited to a trusted local environment.

## 10. Useful commands

| Task | Command |
|---|---|
| Start services | `docker compose up -d` |
| Build and start | `docker compose up -d --build` |
| Stop services | `docker compose down` |
| Restart MantisBT | `docker compose restart mantisbt` |
| Restart Mailpit | `docker compose restart mailpit` |
| Show service status | `docker compose ps` |
| MantisBT logs | `docker compose logs -f mantisbt` |
| MariaDB logs | `docker compose logs -f mariadb` |
| Mailpit logs | `docker compose logs -f mailpit` |
| Check PHP extensions | `docker compose exec mantisbt php -m` |

**Database caution:** `docker compose down` keeps the named database volume. `docker compose down -v` removes it and can permanently delete the database data.
