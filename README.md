# Enterprise Drupal 8 Starter

The goals of this project are the following:

* Provide an end-to-end solution for a Drupal 8 application including both continuous integration and continuous deployment.
* Composer managed dependencies based on Drupal's [Composer template for Drupal projects](https://github.com/drupal-composer/drupal-project). The core should not be versioned (execept in composer.json).
* Use Docker for both development and production environments ensuring developer's local
   machine matches the remote environment.
* No assumptions should be made about the theme or high-level application layer.
* Strategies for automated continuous integration (CI).
* Strategies for automated continuous deployment (CD).

## Dependencies

* [Docker](https://www.docker.com/)
* [Docker Compose](https://docs.docker.com/compose/) _(Not required but recommended)_
* [Composer](https://getcomposer.org/) _(Development only)_

## Table of Contents

1. [Development](#development)
2. [Project Settings (development, production, staging, ect](#project-settings)
3. [Continuous Integration & Deployment](#continuous-integration-and-deployment)
4. [Enabling SSL](#enabling-ssl)

## Development

Clone or download a zip of this repository and `cd` in to `/src` and run:

```shell
composer install
```

This will install the Drupal's core and all dependencies. Then Build and run the
development environment with:

```shell
docker-compose -f docker-compose.dev.yml up -d


# You may want to add a namespace other than the directory the project is in. To
# do so provide a `-p` flag specifying the name of the project:

# docker-compose -p my_project -f docker-compose.dev.yml up -d
```

See the development environment at: [http://[your-docker-ip]:3000](http://[your-docker-ip]:3000)

### Handy commands during development

```shell
# Tail php logs.
docker logs -f <project-name>_drupal_1

# (Re-)build the
docker-compose -f docker-compose.dev.yml build

# View running containers.
docker images

# Destroy containers, but leave database and files volume.
docker-compose -f docker-compose.dev.yml down

# Destroy the database and drupal file persistent volume.
docker-compose -f docker-compose.dev.yml down --volume

# See all project volumes.
docker volume ls
```

## Project Settings

You'll notice by looking at the [docker-compose.dev.yml](docker-compose.dev.yml)
file that what the environmental variables are set to are not very secure. When
it comes time to stage the application you're going to want to do two things:

1. Upon file/container deployment, create a docker-compose.yml file (notice it's
   ignored in the [.gitignore](.gitignore) file).
2. Create either dev.* or prod.* versions of the [conf/](/conf/) files which
   will mirror the credentials in the [docker-compose.dev.yml](docker-compose.dev.yml) file.
   These files are also ignored and not included in the repository or container.

These files can be managed, stored, and deployed securely using [Stackahoy's](https://stackahoy.io/)
static files feature. Aternatively, you create them on the fly with a
post-receive shell script or just by SSH'ing on the server and creating them.

## Continuous Integration

Ideally, our CI will accomplish the following:

1. Test the code's integrity with PHPUnit.
2. Build the latest docker image and push it to a registery.
3. Deploy code to server based on the branch that was pushed to.

## Enabling HTTPS (SSL)

We highly recommend you use [Certbot](https://certbot.eff.org/) to create your
SSL certificate.

Before you run the `docker-compose up` command, you'll need to issue the
certificate into a volume which will be used by nginx.

_Make sure you change "YOUR_DOMAIN.com" to whatever your domain is._

```bash
# Create the volumes where certificates will persist. They need to persist
# forever so we can kill the containers without worring about losing them.
docker volume create --name certs
docker volume create --name certs-data

# Run a web server for authentication.
docker run -d --rm \
  -v certs:/etc/letsencrypt \
  -v certs-data:/data/letsencrypt \
  -v $(pwd)/conf/certbot.nginx.vh.conf:/etc/nginx/conf.d/default.conf \
  --name ssl_nginx \
  -p "80:80" \
  nginx:1.13-alpine

# Initialize the certificates from https://certbot.eff.org/ (EFF!). It's going
to ask you to provide some information.
docker run -it --rm \
  -v certs:/etc/letsencrypt \
  -v certs-data:/data/letsencrypt \
  deliverous/certbot \
  certonly --webroot --webroot-path=/data/letsencrypt -d YOUR_DOMAIN.com

# Stop and remove the certbot nginx instance.
docker stop ssl_nginx && docker rm ssl_nginx
```

This certificate will last for 90 days. You can create a cronjob which will
renew it by calling the following script every 80 days or so.

_Make sure you replace "YOUR_NAMESPACE" with whatever the prefix is for your
containers._

```bash
# Renew tokens... when ssl expires.
docker run -t --rm \
	--volumes-from YOUR_NAMESPACE_proxy_1 \
	deliverous/certbot \
	renew \
	--webroot --webroot-path=/data/letsencrypt
```

Now simply uncomment the 2 cert volumes in your docker-compose file, and start
the application.
