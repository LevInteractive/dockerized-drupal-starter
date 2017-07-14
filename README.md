# Dockerized Drupal 8 Starter

[![Build Status](https://travis-ci.org/LevInteractive/enterprise-drupal-starter.svg?branch=master)](https://travis-ci.org/LevInteractive/enterprise-drupal-starter)

The goals of this project are the following:

* Strategies for continuous integration (CI).
* Strategies for continuous deployments (CD).
* Composer managed dependencies based on Drupal's [Composer template for Drupal projects](https://github.com/drupal-composer/drupal-project). The core should not be versioned (execept in composer.json).
* Use Docker for both development and production environments ensuring developer's local
   machine matches the remote environment.
* Provide a easy way to enable SSL/HTTPS using [Certbot](https://certbot.eff.org/).
* No assumptions should be made about the theme or high-level application layer. **This project is not a theme.**

## Dependencies

* [Docker](https://www.docker.com/)
* [Docker Compose](https://docs.docker.com/compose/) _(Not required but recommended)_
* [Composer](https://getcomposer.org/) _(Development only)_

## Table of Contents

1. [Development](#development)
2. [Project Settings (development, production, staging, ect)](#project-settings)
3. [Continuous Integration & Deployment](#continuous-integration-and-deployment)
4. [Enabling SSL](#user-content-enabling-https-ssl)

## Development

Clone or download a zip of this repository and `cd` into it.

```shell
# Install deps. (will install on your actual machine because of volumes).
docker-compose run drupal composer --working-dir="/var/www" install

# Run the dev environment.
docker-compose up -d
```

See the development environment at: [http://[your-docker-ip]:3000](http://[your-docker-ip]:3000)

#### Handy commands during development

```shell
# Tail php logs.
docker logs -f <project-name>_drupal_1

# (Re-)build the
docker-compose build

# View running containers.
docker images

# Destroy containers, but leave database and files volume.
docker-compose down

# Destroy the database and drupal file persistent volume.
docker-compose down --volume

# See all project volumes.
docker volume ls
```

## Project Settings

You'll notice by looking at the [docker-compose.yml](docker-compose.yml)
file that what the environmental variables are set to are not very secure. When
it comes time to stage the application you're going to want to do two things:

1. Upon file deployment, create production-ready [conf/](/conf/) files (nginx and settings). For example,
   `prod.nginx.vh.default.conf` which will point to your real world domain.
2. Upon file deployment, create a docker-compose.(prod|staging|anything).yml file (notice it's
   ignored in the [.gitignore](.gitignore) file) with updated volumes pointed to
   the conf files you created in step one (the drupal and nginx service).

These files can be managed, stored, and deployed securely using [Stackahoy's](https://stackahoy.io/)
static files feature. Aternatively, you create them on the fly with a
post-receive shell script or just by SSH'ing on the server and creating them.

## Continuous Integration

Ideally, the CI will accomplish the following:

1. Build the latest docker image and push it to a registery. There are a few
   different registries you can go with. Some popular options being:
     * [Docker Hub](https://hub.docker.com/)
     * [GitLab](https://about.gitlab.com/)
     * [Google Cloud](https://cloud.google.com/container-registry/)
     * [AWS](https://aws.amazon.com/ecr/)
2. Test the container/code with PHPUnit.
3. Deploy code to server based on the branch that was pushed to. [stackahoy.io](https://stackahoy.io) allows
   you to handle your deployment specific dependencies (servers, post deployment
   commands, ect) while using the command-line to trigger it.

#### Testing

Whether you're using GitLab, Travis, CicleCI, or another CI provider, you'll be
able to simply run the following to test it thanks to docker:

```yaml
scripts:
  - docker-compose run drupal /usr/local/bin/composer --working-dir="/var/www" install
  - docker-compose run drupal ../vendor/phpunit/phpunit/phpunit -c core --testsuite unit --exclude-group Composer,DependencyInjection,PageCache
  - docker-compose run drupal ../vendor/bin/drush
```

You won't have to rely on the CI provider to spin up a instance of php or mysql
because Docker will handle that for you. Pretty easy, eh?

#### Deployment

See the [.travis.yml](.travis.yml) file. For deployment, simply configure the
deployment procedure in [stackahoy.io](https://stackahoy.io) and populate the
three environmental variables in the project settings within Travis. This works
very similarly in most other CI providers.

```yaml
after_success:
  # Supply the $STACKAHOY_TOKEN, $STACKAHOY_REPO_ID, and $STACKAHOY_BRANCH from
  # stackahoy.io. Stackahoy can take care of any other post-deployment commands
  # necessary or notifications.
  - if [[ $RELEASE = stable ]]; then docker run -it stackahoy/stackahoy-cli stackahoy deploy --token="$STACKAHOY_TOKEN" --repo="$STACKAHOY_REPO_ID" --branch="$STACKAHOY_BRANCH"; fi;
```


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

Now in order for it to work, you'll need to configure the prod.nginx.vh.conf
file to use the certs and uncomment the 2 cert volumes in the docker-compose
file. The nginx file will need:

```nginx
ssl_certificate           /etc/letsencrypt/live/YOUR_DOMAIN.com/fullchain.pem;
ssl_certificate_key       /etc/letsencrypt/live/YOUR_DOMAIN.com/privkey.pem;
ssl_trusted_certificate   /etc/letsencrypt/live/YOUR_DOMAIN.com/chain.pem;
```
