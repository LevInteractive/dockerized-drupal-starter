# Enterprise Drupal 8 Starter

*Note:* Documentation is still being completed. Stand by!

The goals of this project are the following:

* Provide an end-to-end solution for a Drupal 8 application including both continuous integration and continuous deployment.
* Use Docker for both development and production environments ensuring developer's local
   machine matches the remote environment.
* Use composer manage all dependencies based on Drupal's [Composer template for Drupal projects](https://github.com/drupal-composer/drupal-project). The core should not be versioned (execept in composer.json).
* Strategies for automated continuous integration (CI).
* Strategies for automated continuous deployment (CD).

# Dependencies

* [Docker](https://www.docker.com/)
* [Docker Compose](https://docs.docker.com/compose/) _(Not required but recommended)_

# Table of Contents

1. [Installation](#installation)
2. [Development](#development)
3. [Continuous Integration](#continuous-integration)
4. [Continuous Deployment](#continuous-deployment)

# Installation

##### Step 1

Clone or download a zip of this repository and `cd` in to `/src` and run:

```shell
composer install
```

This will install the Drupal's core and all dependencies.

##### Step 2

Move back in to the project's root and run:

```shell
docker-compose -f docker-compose.dev.yml build
```

This will build the docker images. Once complete, view them with:

```shell
docker images
```

# Development

Run the development environment with:

```shell
docker-compose -f docker-compose.dev.yml up -d

# You may want to add a namespace other than the directory the project is in. To
# do so provide a `-p` flag specifying the name of the project:
# docker-compose -p my_project -f docker-compose.dev.yml up -d
```

See the development environment at: http://[your-docker-ip]:3000


### Handy commands during development

```shell
# Tail php logs.
docker logs -f <project-name>_drupal_1

# View running containers.
docker images

# Destroy the database and drupal file persistent volume.
docker-compose -f docker-compose.dev.yml down --volume

# Destroy containers, but leave database and files volume.
docker-compose -f docker-compose.dev.yml down
```

# Continuous Integration

Ideally, CI will do two important tasks for us:

1. Test the code's integrity with PHPUnit.
2. Build the latest docker image and push it to a registery.

Luckily, Drupal 8 comes with a suite of phpunit tests which will test the
integrity of its core. To run them, you must `cd` into the core directory and
run:

```shell
../../vendor/phpunit/phpunit/phpunit --testsuite=unit
```

For executing this, we recommend any of the following CI providers:

* [GitLab CI](https://about.gitlab.com/features/gitlab-ci-cd/)
* [Travis CI](https://travis-ci.org/)
* [CircleCi](https://circleci.com/)

# Continuous Deployment

[Stackahoy CLI](https://stackahoy.io/docs/cli) can be used to make automated and
unobtrusive deployments once the CI has completed (if using CI). This usually
happens as the last phase of the CI pipeline, and with the
[stackahoy-cli](https://hub.docker.com/r/stackahoy/stackahoy-cli/) docker
image, we don't even have to create a custom runner for it.

The following is example from a [.gitlab-ci.yml](https://docs.gitlab.com/ee/ci/yaml/) file:

```yaml
deploy_staging:
  image: stackahoy/stackahoy-cli
  stage: deploy_staging
  script:
    - stackahoy deploy -t $STACKAHOY_TOKEN -b staging -r $REPO_ID
  only:
    - staging
  tags:
    - dind
```


