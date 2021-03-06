The `docker` directory contains all the resources needed to create a
production-ready Docker image for Portus.

The master branch of this repository is going to include the files needed
to build Portus from HEAD. Other branches are available to build Portus out of
more stable branches. These branches are going to be named using the following
scheme: `portus-<release>`.

## Security

SUSE's containers team cares about security, hence we made the following
decisions:

  * This image enforces the usage of HTTPS to access Portus.
  * Portus is installed from an RPM package, this ensures the final image will
    have only the runtime dependency; no build time dependency is ever installed.
  * The default user for this image is the `wwwrun` one, which is created by the
    Apache package.
  * The root account is locked, that means it's not possible to switch user via
    `su` or `sudo`.

SUSE's containers team is constantly working to automate the release process
of Portus and of this image to ensure it stays up-to-date and secure.

When deploying this image make sure to add all the required keys and
certificates at runtime instead of adding them to a Docker image.

## Anatomy of the image

The image is based on openSUSE 42.1 and installs Portus using the RPM package
built by SUSE's containers team inside of the [Open Build Service](https://build.opensuse.org/project/subprojects/Virtualization:containers:Portus)
top level project and subprojects (one subproject per portus branch).

### Exposed ports

This image will use [Phusion Passenger](https://www.phusionpassenger.com/) in
conjunction with Apache to run Portus.

The container will expose the following ports:

  * 443: this is used to serve the website over HTTPS.
  * 80: most of the requests sent to this unencrypted channel will be
    automatically redirected to the HTTPS channel.
    The redirection is made using Apache's [mod_rewrite](https://httpd.apache.org/docs/current/mod/mod_rewrite.html).

The redirection from port 80 to port 443 applies to all the common requests sent
to Portus. The only requests that are **not** redirected are the ones sent to
the `v2/webhooks/events` endpoint. This is the endpoint used by the Docker
Registry to send notifications to Portus.

Sending Registry's notification to the HTTPS endpoint would require Portus'
certificate to be added to the Docker container running the registry. This is
not hard to be done, but involves some changes to the official Registry image
maintained by Docker Inc (or some hacks like a custom entrypoint added via a volume).

### The init script

As mentioned before, this Docker image has a init script which takes care of the
following actions:

  1. Setup the database required by Portus
  2. Import all the `.crt` located under `/certificates`
  3. Start Apache

The next sections will provide more details about these steps.

### Volumes

Portus' state is stored inside of a MariaDB database. This makes this Docker
image stateless.

### External database

This image has a custom `init` script that takes care of configuring the external
database.

The script will keep trying to reach the database for 90 seconds. A 5 seconds
pause is done after each failed attempt. The container will exit with an error
message if the database is not reachable.

The script will take care of the creation of the Portus database and its initial
population via the usual Rails procedures.

The database is automatically migrated whenever a new migration is introduced
by the upstream project.

### Secrets and certificates

Portus requires both a SSL key and a certificate to serve its contents over
HTTPS.
These files must be located here:

  * SSL certificate file: `/certificates/portus-ca.crt`
  * SSL certificate key file: `/secrets/portus.key`

The same key is also used to sign the JWT tokens issued to authenticate all the
docker clients against the Registry.

It's also required to add the certificate of the Registry when the latter one
uses TLS to secure itself.
The Registry certificate must be placed inside of `/certificates`, the `init`
script of this image will automatically import it if it ends with the `.crt`
extension.

### Logging

Both Portus' and Apache's messages are redirected to `stdout` and `stderr`. This
makes it possible to handle the logs of this image in the usual ways.

## Deployment

It's possible to deploy this image using one of the existing orchestration
solutions for Docker images.

However we recommend the usage of [kubernetes](http://kubernetes.io/). We will
soon release a reference implementation.

### Environment variables

The following environment variables must be defined to run the image:

Security related settings:

  * `PORTUS_SECRET_KEY_BASE`: you can generate it using `openssl rand -hex 64`
  * `PORTUS_PORTUS_PASSWORD`: you can generate it using `openssl rand -hex 64`

Database releated settings:

  * `MARIADB_SERVICE_HOST`: the host running the MariaDB database.
  * `MARIADB_USER`: the database user to be used.
  * `MARIADB_PASSWORD`: the password of the database user.
  * `MARIADB_DATABASE`: the name of the Portus database.

Deployment related settings:

  * `PORTUS_MACHINE_FQDN_VALUE`: this is the fully qualified domain name of your
    Portus instance (eg: `portus.example.com`).
