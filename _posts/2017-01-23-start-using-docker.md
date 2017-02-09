---
title: Start using Docker

---

Docker can be an overwhelming technology at the beginning.
However we don't need to use it in every piece of our developments,
even less when we are learning it.
We can start using slowly.

![Docker](/assets/images/docker.svg){: .align-center}

## Use docker for databases in a development environment

I think a good first step to start using it can be like a database for development purpose.
If we have installed a Postgres, MySQL or MongoDB service on our development machine,
usually, the information we are storing there is for testing and not important at all.
In those cases, we can virtualize those services inside a docker instance without fear.

Moreover, we will have several advantages:

* We can have multiple versions running at the same time without conflicts
* We can use the production version without depending on your distribution packages
* We can stop using them with just a command, reducing memory when need it
* We can learn more things about docker in the process

## Dockerizing a Postgres Database

### Previous steps

Before going on remember to uninstall your current database if you have already installed:

```
# pacman -Rns postgres
```

And of course, install [docker](https://wiki.archlinux.org/index.php/Docker).
Pay special attention to the `docker` group and remember to log out correctly of your current session.

### Running Postgres in a Docker container

A simple way to start could be:

```bash
$ docker run --restart=unless-stopped --name postgres -p 5432:5432 -d postgres:latest
```

Let's explain the command bit by bit:

* `docker` the main program
* `run` is a subcommand defined in docker. It allows us to run a command in a new container
* `--restart=unless-stopped` in case the container died for any reason, this will reboot the docker container.
It also works to start the container automatically when computer boots
* `--name postgres` a label to identify the container in an easy way, in this case, we called _postgres_
* `-p 5432:5432` exposes the port of `5432` of the container like `5432` in our real computer. This port is the default _postgres_ port
* `-d` sends the process to the background
* `postgres:latest` is the image we are going to execute. In this case is the [official postgres image](https://hub.docker.com/_/postgres/).
It is ready to execute the database automatically.
_latest_ is to use the last version available, but we could use other options like _9.6_, _9.5_ or even more specific _9.5.2_.

And now we have an postgres running as docker container. To see the docker currently running we can use:

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
462b1bae2bf8        postgres            "/docker-entrypoint.s"   4 weeks ago         Up 2 days           0.0.0.0:5432->5432/tcp   postgres
```

If for any reason we want to stop it:

```
$ docker stop postgres
```

And to remove definitely we will use:

```
$ docker rm postgres
```

## Connecting to the database

You can now connect to the database. The default credentials are:

```
host: localhost
user: postgres
password: (empty)
```

You can modify those values. More information in the [official postgres image](https://hub.docker.com/_/postgres/).
Remember to change those values in your development projects. In `Ruby on Rails` would be something like this:

```yaml
# File config/database.yml
development:
  adapter:  postgresql
  host:     localhost
  encoding: unicode
  database: project_development
  pool:     5
  username: postgres
  password:
  template: template0

```

## Conclusion

This is it, this is the very minimum to start with docker.
It exists a long way before mastering this awesome technology:
docker-compose, production environment, interconnection, etc.
However, we can benefit from minute one, just learning the basics.

And do you use docker to virtualize your databases in development?
