# Running with Docker

This is a sample configuration for running Healthchecks with
[Docker](https://www.docker.com) and [Docker Compose](https://docs.docker.com/compose/).

Note: For the sake of simplicity, the sample configuration starts a single database
node and a single web server node, both on the same host. It does not handle TLS
termination.

## Getting Started

* Copy `/docker/.env.example` to `/docker/.env` and add your configuration in it.
  As a minimum, set the following fields:
    * `ALLOWED_HOSTS` – the domain name of your Healthchecks instance.
    Example: `ALLOWED_HOSTS=hc.example.org`.
    * `DEFAULT_FROM_EMAIL` – the "From:" address for outbound emails.
    * `EMAIL_HOST` – the SMTP server.
    * `EMAIL_HOST_PASSWORD` – the SMTP password.
    * `EMAIL_HOST_USER` – the SMTP username.
    * `SECRET_KEY` – secures HTTP sessions, set to a random value.
    * `SITE_ROOT` – The base public URL of your Healthchecks instance. Example:
    `SITE_ROOT=https://hc.example.org`.

* Create and start containers:

  ```sh
  docker compose up
  ```

* Create a superuser:

  ```sh
  docker compose run web /opt/healthchecks/manage.py createsuperuser
  ```

* Open [http://localhost:8000](http://localhost:8000) in your browser and log in with
  the credentials from the previous step.

## Running `manage.py` Commands

* `collectstatic`, `compress` – when running with Docker, you do
  not need to manually run these. These are run while building the container image,
  and their results are baked in the image (you can find them listed in the [Dockerfile](https://github.com/healthchecks/healthchecks/blob/master/docker/Dockerfile)).
* `migrate`, `sendalerts`, `sendreports`, `smtpd` – when running with Docker, you
  also do  not need to manually run these. They are run automatically on
  container startup (you can find them listed in [uwsgi.ini](https://github.com/healthchecks/healthchecks/blob/master/docker/uwsgi.ini)).
* `createsuperuser`, `pruneobjects`, `prunetokenbucket`, `pruneusers`,
  `settelegramwebhook` – you need to run them **inside the container**, not on
  the host system. Do it like so:

  ```sh
  docker compose run web /opt/healthchecks/manage.py <command>
  ```

## uWSGI Configuration

The reference Dockerfile uses [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)
as the WSGI server. You can configure uWSGI by setting `UWSGI_...` environment
variables in `docker/.env`. For example, to disable HTTP request logging, set:

    UWSGI_DISABLE_LOGGING=1

To adjust the number of uWSGI processes (for example, to save memory), set:

    UWSGI_PROCESSES=2

Read more about configuring uWSGI in [uWSGI documentation](https://uwsgi-docs.readthedocs.io/en/latest/Configuration.html#environment-variables).

## SMTP Listener Configuration via `SMTPD_PORT`

Healthchecks comes with a `smtpd` management command, which runs a SMTP listener
service. With the command running, you can ping your checks by sending email messages
to `your-uuid-here@your-hc-domain.com` email addresses.

The container is configured to start the SMTP listener conditionally, based
on the value of the `SMTPD_PORT` environment value:

* If `SMTPD_PORT` environment variable is not set, the SMTP listener will not run.
* If `SMTPD_PORT` is set, the listener will run and listen on the specified port.
  You may also need to edit `docker-compose.yml` to expose the listening port
  (see the "ports" section under the "web" service in `docker-compose.yml`).

The conditional logic lives in uWSGI configuration file,
[uwsgi.ini](https://github.com/healthchecks/healthchecks/blob/master/docker/uwsgi.ini).

## TLS Termination and CSRF Protection

If you plan to expose your Healthchecks instance to the public internet, make sure you
put a TLS-terminating reverse proxy or load balancer in front of it.

**Important:** This Dockerfile uses uWSGI, which relies on the [X-Forwarded-Proto](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Proto)
header to determine if a request is secure or not. Without this information you
may run into HTTP 403 "CSRF verification failed." errors when using your Healthchecks
instance. See [this issue comment](https://github.com/healthchecks/healthchecks/discussions/851#discussioncomment-6293396)
for more information.

Make sure your TLS-terminating reverse proxy:

* Discards the `X-Forwarded-Proto` header sent by the end user.
* Sets the `X-Forwarded-Proto` header value to match the protocol of the original request
  ("http" or "https").

For example, in NGINX you can use the `$scheme` variable like so:

```
proxy_set_header X-Forwarded-Proto $scheme;
```

If you are using haproxy, you can do the same like so:

```
http-request set-header X-Forwarded-Proto https if { ssl_fc }
http-request set-header X-Forwarded-Proto http unless { ssl_fc }
```

## Upgrading Database

When you upgrade the database version in `docker-compose.yml` (for example,
from `postgres:12` to `postgres:16`), you will also need to upgrade your postgres
data directory. One way to do this is using the
[pgautoupgrade](https://hub.docker.com/r/pgautoupgrade/pgautoupgrade) container.

Steps:

* As the very first step, **take a full backup of your database**.
* Stop the `db` and `web` containers: `docker compose stop`
* Look up the name of the postgres data volume name using `docker volume ls`
* Run `pgautoupgrade` like so:

```
docker run --rm --name pgauto -it \
   --mount type=volume,source=<pg-volume-name-here>,target=/var/lib/postgresql/data \
   -e POSTGRES_PASSWORD=password \
   -e PGAUTO_ONESHOT=yes \
   pgautoupgrade/pgautoupgrade:16-bookworm
```

* Update the `docker-compose.yml` file to use the `postgres:16` image
* Start containers: `docker compose up`

## Pre-built Images

Pre-built Docker images, built from the Dockerfile in this directory, are available
[on Docker Hub](https://hub.docker.com/r/healthchecks/healthchecks). The images are
built automatically for every new release.

The Docker images:

* Support amd64, arm/v7 and arm64 architectures.
* Use uWSGI as the web server. uWSGI is configured to perform database migrations
  on startup, and to run `sendalerts`, `sendreports`, and `smtpd` in the background.
  You do not need to run them separately. The SMTP listener (`manage.py smtpd`) is
  started conditionally, [based on the value of the `SMTPD_PORT` environment variable](https://github.com/healthchecks/healthchecks/tree/master/docker#smtp-listener-configuration-via-smtpd_port).
* Ship with both PostgreSQL and MySQL database drivers.
* Serve static files using the whitenoise library.
* Have the apprise library preinstalled.
* Do *not* handle TLS termination. In a production setup, you will want to put
  the Healthchecks container behind a reverse proxy or load balancer that handles TLS
  termination.

To use a pre-built image for Healthchecks version X.Y, in the `docker-compose.yml` file
replace the "build" section with:

```text
image: healthchecks/healthchecks:vX.Y
```

To upgrade to a newer Healthchecks version, edit `docker-compose.yml` and update the
version number in the container image name (the "X.Y" part) and run
`docker compose up`.