centos-ssh-mysql
================

Docker Image of CentOS-6 6.7 x86_64, MySQL 5.1.

Includes Automated password generation and an option for custom initialisation SQL. Supports custom configuration via environment variables and/or a configuration data volume.

## Overview & links

The latest CentOS-6 based release can be pulled from the centos-6 Docker tag. For a specific release tag the convention is `centos-6-1.5.0` for the [1.5.0](https://github.com/jdeathe/centos-ssh-mysql/tree/1.5.0) release tag.

- centos-6 [(centos-6/Dockerfile)](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/Dockerfile)

The Dockerfile can be used to build a base image that is the bases for several other docker images.

Included in the build are the [EPEL](http://fedoraproject.org/wiki/EPEL) and [IUS](https://ius.io/) repositories. Installed packages include [OpenSSH](http://www.openssh.com/portable.html) secure shell, [vim-minimal](http://www.vim.org/), [MySQL Server and client programs](http://www.mysql.com) are installed along with python-setuptools, [supervisor](http://supervisord.org/) and [supervisor-stdout](https://github.com/coderanger/supervisor-stdout).

Supervisor is used to start the mysqld server daemon when a docker container based on this image is run. To enable simple viewing of stdout for the service's subprocess, supervisor-stdout is included. This allows you to see output from the supervisord controlled subprocesses with `docker logs {container-name}`.

If enabling and configuring SSH access, it is by public key authentication and, by default, the [Vagrant](http://www.vagrantup.com/) [insecure private key](https://github.com/mitchellh/vagrant/blob/master/keys/vagrant) is required.

### SSH Alternatives

SSH is not required in order to access a terminal for the running container. The simplest method is to use the docker exec command to run bash (or sh) as follows:

```
$ docker exec -it {container-name-or-id} bash
```

For cases where access to docker exec is not possible the preferred method is to use Command Keys and the nsenter command. See [command-keys.md](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/command-keys.md) for details on how to set this up.

## Quick Example

Run up a container named `mysql.pool-1.1.1` from the docker image `jdeathe/centos-ssh-mysql` on port 3306 of your docker host.

```
$ docker run -d \
  --name mysql.pool-1.1.1 \
  -p 3306:3306 \
  -v /var/lib/mysql \
  jdeathe/centos-ssh-mysql:centos-6
```

Now you can verify it is initialised and running successfully by inspecting the container's logs.

```
$ docker logs mysql.pool-1.1.1
```

If it is the first run there should be additional output showing the initialisation SQL that was run and the root user's password.

![Docker Logs MySQL Bootstrap](https://raw.github.com/jdeathe/centos-ssh-mysql/centos-6/images/docker-logs-mysql-bootstrap.png)

The MySQL table data is persistent across container restarts by setting the MySQL data directory `/var/lib/mysql` as a data volume. We didn't specify a name or docker_host path so Docker will give it a unique name and store it in `/var/lib/docker/volumes/`; to find out where the data is stored on the Docker host you can use `docker inspect`.

```
$ docker inspect \
  --format '{{ json (index .Mounts 0).Source }}' \
  mysql.pool-1.1.1
```

To access the MySQL SQL shell run the following:

```
$ docker exec -it mysql.pool-1.1.1 mysql -p -u root
```

To import the Sakila example database from the [MySQL Documentation](https://dev.mysql.com/doc/index-other.html) and view the first 2 records from the film table.

```
$ export MYSQL_ROOT_PASSWORD={your-password}

$ docker exec -i mysql.pool-1.1.1 \
  mysql -p${MYSQL_ROOT_PASSWORD} -u root \
  <<< $(tar -xzOf /dev/stdin <<< $(curl -sS http://downloads.mysql.com/docs/sakila-db.tar.gz) sakila-db/sakila-schema.sql)

$ docker exec -i mysql.pool-1.1.1 \
  mysql -p${MYSQL_ROOT_PASSWORD} -u root \
  <<< $(tar -xzOf /dev/stdin <<< $(curl -sS http://downloads.mysql.com/docs/sakila-db.tar.gz) sakila-db/sakila-data.sql)

$ docker exec -it  mysql.pool-1.1.1 \
  mysql -p${MYSQL_ROOT_PASSWORD} -u root \
  -e "SELECT * FROM sakila.film LIMIT 2 \G;"
```

## Instructions

### (Optional) Configuration Data Volume

A configuration "data volume" allows you to share the same configuration files between multiple docker containers. Docker mounts a host directory into the data volume allowing you to edit the default configuration files and have those changes persist.

Each service that requires a common set of configuration files could use a single Configuration Volume as illustrated in the following diagram:

```
+---------------------------------------------------+
|                (Docker Host system)               |
|                                                   |
| /var/lib/docker/volumes/{volume-name}/_data       |
|                         +                         |
|                         |                         |
|            +============*===========+             |
|            |  Configuration Volume  |             |
|            |    Service Container   |             |
|            +============*===========+             |
|                         |                         |
|         +---------------*---------------+         |
|         |               |               |         |
|   +=====*=====+   +=====*=====+   +=====*=====+   |
|   |  Service  |   |  Service  |   |  Service  |   |
|   | Container |   | Container |   | Container |   |
|   |    (1)    |   |    (2)    |   |    (n)    |   |
|   +===========+   +===========+   +===========+   |
+---------------------------------------------------+
```

#### Standard data volume container

Naming of the container's volume is optional, it is possible to leave the naming up to Docker by simply specifying the container path only.

```
$ docker run \
  --name volume-config.mysql.pool-1.1.1 \
  -v /etc/services-config \
  jdeathe/centos-ssh-mysql:centos-6 \
  /bin/true
```

To identify the docker host directory path to the volume within the container `volume-config.mysql.pool-1.1.1` you can use `docker inspect` to view the Mounts.

```
$ docker inspect \
  --format '{{ json (index .Mounts 0).Source }}' \
  volume-config.mysql.pool-1.1.1
```

#### Named data volume container

To create a named data volume, mounting our docker host's configuration directory /var/lib/docker/volumes/volume-config.mysql.pool-1.1.1 to /etc/services-config in the docker container use the following run command. Note that we use the same image as for the application container to reduce the number of images/layers required.

```
$ docker run \
  --name volume-config.mysql.pool-1.1.1 \
  -v volume-config.mysql.pool-1.1.1:/etc/services-config \
  jdeathe/centos-ssh-mysql:centos-6 \
  /bin/true
```

##### Populating Named configuration data volumes  
When using named volumes the directory path from the docker host mounts the path on the container so we need to upload the configuration files. The simplest method of achieving this is to upload the contents of the [etc/services-config](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/) directory using `docker cp`.

```
$ docker cp \
  ./etc/services-config/. \
  volume-config.mysql.pool-1.1.1:/etc/services-config
```

If you don't have a copy of the required configuration files locally you can run a temporary container as the source of the configuration files and use `docker cp` to stream the files into the named data volume container.

```
$ docker run -d \
  --name mysql.tmp \
  jdeathe/centos-ssh-mysql:centos-6 \
  /bin/sh -c 'while true; do echo -ne .; sleep 1; done';
  && docker cp \
  mysql.tmp:/etc/services-config/. - | \
  docker cp - \
  volume-config.mysql.pool-1.1.1:/etc/services-config
  && docker rm -f mysql.tmp
```

#### Editing configuration

To make changes to the configuration files you need a running container that uses the volumes from the configuration volume. To edit a single file you could use the following, where {path_to_file} can be one of the [required configuration files](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/README.md#required-configuration-files), or you could run a `bash` shell and then make the changes required using `vi`. On exiting the container it will be removed since we specify the `--rm` parameter.

```
$ docker run --rm -it \
  --volumes-from volume-config.mysql.pool-1.1.1 \
  jdeathe/centos-ssh-mysql:centos-6 \
  vi /etc/services-config/{path_to_file}
```

##### Required configuration files

The following configuration files are required to run the application container and should be located in the directory /etc/services-config/.

- [mysql/my.cnf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/mysql/my.cnf)
- [mysql/mysql-bootstrap.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/mysql/mysql-bootstrap.conf)
- [supervisor/supervisord.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.conf)
- [supervisor/supervisord.d/sshd-bootstrap.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.d/sshd-bootstrap.conf)
- [supervisor/supervisord.d/sshd.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.d/sshd.conf)
- [supervisor/supervisord.d/mysqld-bootstrap.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/supervisor/supervisord.d/mysqld-bootstrap.conf)
- [supervisor/supervisord.d/mysqld-wrapper.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/supervisor/supervisord.d/mysqld-wrapper.conf)


### Running

To run the a docker container from this image you can use the included [run.sh](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/run.sh) and [run.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/run.conf) scripts. The helper script will stop any running container of the same name, remove it and run a new daemonised container on an unspecified host port. Alternatively you can use the following methods to make the service available on port 3306 of the docker host.

#### Using environment variables

The following example sets up a custom MySQL database, user and user password on first run. This will only work when MySQL runs the initialisation process and values must be specified for MYSQL_USER and MYSQL_USER_DATABASE. If MYSQL_USER_PASSWORD is not specified or left empty a random password will be generated.

*Note:* Settings applied by environment variables will override those set within configuration volumes from release 1.3.1. Existing installations that use the mysql-bootstrap.conf saved on a configuration "data" volume will not allow override by the environment variables. Also users can update mysql-bootstrap.conf to prevent the value being replaced by that set using the environment variable.

```
$ docker stop mysql.pool-1.1.1 && \
  docker rm mysql.pool-1.1.1

$ docker run -d \
  --name mysql.pool-1.1.1 \
  -p 3306:3306 \
  --env "MYSQL_SUBNET=127.0.0.1" \
  --env "MYSQL_USER=app-user" \
  --env "MYSQL_USER_PASSWORD=" \
  --env "MYSQL_USER_DATABASE=app-db" \
  -v volume-data.mysql.pool-1.1.1:/var/lib/mysql \
  jdeathe/centos-ssh-mysql:centos-6
```

#### Using configuration volume

The following example uses the settings from the optional configuration volume volume-config.mysql.pool-1.1.1 and maps a data volume for persistent storage of the MySQL data on the docker host.

*Note:* In the above example we are using `docker volume rm` to destroy the named data volume and allow the mysql-bootstrap initialisation process to run. This might be necessary if the named data volume already exists from the previous example but should be **used with caution** since it will **destroy all existing mysql data** for any containers that use it.

```
$ docker stop mysql.pool-1.1.1 && \
  docker rm mysql.pool-1.1.1

$ docker volume rm volume-data.mysql.pool-1.1.1

$ docker run -d \
  --name mysql.pool-1.1.1 \
  -p 3306:3306 \
  --env "MYSQL_SUBNET=0.0.0.0/0.0.0.0" \
  --volumes-from volume-config.mysql.pool-1.1.1 \
  -v volume-data.mysql.pool-1.1.1:/var/lib/mysql \
  jdeathe/centos-ssh-mysql:centos-6
```

The environmental variable `MYSQL_SUBNET` is optional but can be used to generate users with access to databases outside the `localhost`, (the default for the root user). In the example, the subnet definition `0.0.0.0/0.0.0.0` allows connections from any network which is equivalent to the wildcard symbol, `%`, in MySQL GRANT definitions.

Now you can verify it is initialised and running successfully by inspecting the container's logs:

```
$ docker logs mysql.pool-1.1.1
```

#### Environment Variables

There are several environmental variables defined at runtime these allow the operator to customise the running container.

*Note:* Most of these settings are only evaluated during the first run of a named container; if the data volume already exists and contains database table data then changing these values will have no effect.

##### MYSQL_ROOT_PASSWORD

On first run the root user is created with an auto-generated password. If you require a specific password,  `MYSQL_ROOT_PASSWORD` can be used when running the container.

```
...
  --env "MYSQL_ROOT_PASSWORD=Passw0rd!" \
...
```

##### MYSQL_ROOT_PASSWORD_HASHED

To indicate `MYSQL_ROOT_PASSWORD` is a pre-hashed value instead of the default plain-text type set `MYSQL_ROOT_PASSWORD_HASHED` to `true`.

```
...
  --env "MYSQL_ROOT_PASSWORD=*03F7361A0E18DA99361B7A82EA575944F53E206B" \
  --env "MYSQL_ROOT_PASSWORD_HASHED=true" \
...
```

*Note:* To generate a pre-hashed password you could use the following MySQL command.

```
$ mysql -u root -p{mysql_root_password} \
  -e "SELECT PASSWORD('{mysql_user_password}');"
```

##### MYSQL_USER

On first run, a database user and database can be created. Set `MYSQL_USER` to a non-empty string. A corresponding `MYSQL_USER_DATABASE` value must also be set for the user to be given access too.

```
...
  --env "MYSQL_USER=app-user" \
...
```

##### MYSQL_USER_PASSWORD

On first run, if the database user `MYSQL_USER` is specified then it is created with an auto-generated password. If you require a specific password,  `MYSQL_USER_PASSWORD` can be used when running the container.

```
...
  --env "MYSQL_USER_PASSWORD=appPassw0rd!" \
...
```

##### MYSQL_USER_PASSWORD_HASHED

To indicate `MYSQL_USER_PASSWORD` is a pre-hashed value instead of the default plain-text type set `MYSQL_USER_PASSWORD_HASHED` to `true`.

```
...
  --env "MYSQL_USER_PASSWORD=*4215553ECE7A18BC09C16DB9EBF03FACFF49166B" \
  --env "MYSQL_USER_PASSWORD_HASHED=true" \
...
```

##### MYSQL_USER_DATABASE

On first run, if the database user `MYSQL_USER` is specified then you must also define a corresponding database name.  `MYSQL_USER_DATABASE` can be used when running the container.

```
...
  --env "MYSQL_USER_DATABASE=app-db" \
...
```

### Custom Configuration

If using the optional data volume for container configuration you are able to customise the configuration. In the following examples your custom docker configuration files should be located on the Docker host under the directory `/var/lib/docker/volumes/{volume-name}/` where `{volume-name}` should identify the applicable container name such as "volume-config.mysql.pool-1.1.1" if using named volumes or will be an ID generated automatically by Docker. To identify the correct path on the Docker host use the `docker inspect` command.

#### [mysql/mysql-bootstrap.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/mysql/mysql-bootstrap.conf)

The bootstrap script initialises the MySQL install with `/usr/bin/mysql_install_db` then secures it and applies any custom SQL with `/usr/bin/mysqld_safe` - any test or root users and test databases are dropped, then a password for the localhost only MySQL root user is generated.

#### [mysql/my.cnf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/mysql/my.cnf)

MySQL can be configured via the my.cnf - refer to the MySQL documentation with respect to what settings are available.

#### [supervisor/supervisord.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.conf)

The supervisor service's primary configuration can also be overridden by editing the custom supervisord.conf file. Program specific configuration files will be loaded from /etc/supervisor.d/ from the container.

#### [supervisor/supervisord.d/sshd-bootstrap.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.d/sshd-bootstrap.conf)

The supervisor program configuration for the sshd_boostrap script.

#### [supervisor/supervisord.d/sshd.conf](https://github.com/jdeathe/centos-ssh/blob/centos-6/etc/services-config/supervisor/supervisord.d/sshd.conf)

The supervisor program configuration for the sshd service.

#### [supervisor/supervisord.d/mysqld-bootstrap.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/supervisor/supervisord.d/mysqld-bootstrap.conf)

The supervisor program configuration for the mysqld_boostrap script.

#### [supervisor/supervisord.d/mysqld-wrapper.conf](https://github.com/jdeathe/centos-ssh-mysql/blob/centos-6/etc/services-config/supervisor/supervisord.d/mysqld-wrapper.conf)

The supervisor program configuration for the mysqld service.
