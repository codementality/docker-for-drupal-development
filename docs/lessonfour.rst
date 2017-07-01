Lesson 4:  Add a database service
=================================

1: Add a MySQL Container to the stack
#####################################

Let's add a new service to our application stack.  Open `docker-compose.yml` in your favorite editor and insert the following under the `services` tag:

.. code-block:: yaml
   :linenos:

   db:
     image: mariadb:10.3.0


What we've done is add a MariaDB container to our application stack, and pinned it to version 10.3.0.

2. Add database environment variables
#####################################

Now, if we look at the documentation for the official MariaDB container on Docker Hub, we'll see that there are a number of "environment variables" that can be passed to our container when we start it up.  These environment variables can be added to our `docker-compose.yml` file as part of our `db` service.

Below the above image declaration, add the following:

.. code-block:: yaml
   :linenos:

    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: drupal
      MYSQL_USER: drupal
      MYSQL_PASSWORD: drupal

This sets the root user password for our MySQL installation to `root`, and sets the username, password and database name for our application database to `drupal`.

3. Add MySQL startup command line options
#########################################

We can also add "command line" options when we start our MySQL container and create our database that consists of any options you would normally configure for your MySQL database instance.  In our case, we're going to set the "character set" and "collation" parameters for our database.

Add the following below the `environment` tag:

.. code-block:: yaml
   :linenos:

    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci # The simple way to override the mariadb config.

The `command`  key will pass any of the parameters listed to whatever executable application is launched by either the container's entrypoint script, or designated in the CMD instruction of the Dockerfile that generated the image.  For MariaDB (and MySQL for that matter), that would be the `mysqld` binary that launches the database server.

4.  Add a volume declaration to the MySQL container
###################################################

Just as we did with our PHP container, we're going to add some data volumes to our MySQL container.

The first volume we're going to add will be a local data volume to store our MySQL database.  Normally when you destroy a container with `docker-compose down`, the container and all of its contents are destroyed.  This is probably not the desired behavior if you're working on a content management system; we want our database data to persist even if we spin our application stack down.

We can do this by declaring a data volume that we mount in the physical location where MySQL normally creates its MySQL databases, which in this case is `/var/lib/mysql`.  Add the following right below the `command` tag:

.. code-block:: yaml

    volumes:
      - mysql-data:/var/lib/mysql

Now this maps a local data volume named `mysql-data` (which we haven't created yet) to the physical file path `/var/lib/mysql` inside our container.

5.  Add a local data volume
###########################

Now we need to add a local data volume to our application stack for use by MySQL.  Add a new top level tag called `volumes` to your `docker-compose.yml`, below the `services` tag and all of the declared services, and define your data volume as follows:

.. code-block:: yaml

    volumes:
      mysql-data:
        driver: local

Notice that the volume name, `mysql-data`, is the same as the volume we mapped for our MySQL container.  Now what will happen when we start our stack is that Docker will create a data volume with the virtual name `mysql-data`, and will mount that volume at `/var/lib/mysql` inside our database container.

When we spin down our application stack with `docker-compose down`, our MySQL container will be destroyed, but our data volume will persist until we physically destroy it.  The next time we spin up our application stack with `docker-compose up -d`, Docker will check to see if the `mysql-data` volume exists; if it does, Docker will mount it to our MySQL container; if it doesn't Docker will create a new data volume to mount inside our MySQL container.

6.  Add a shared data volume for a seed database
################################################

When a container is started for the first time, a new database with the specified name will be created and initialized with the provided configuration variables. Furthermore, it will execute files with extensions `.sh`, `.sql` and `.sql.gz` that are found in the internal path `/docker-entrypoint-initdb.d`.

We can use this information to map a shared data volume where we can place a seed database (which we'll create later) that will be imported into our designated database, `drupal`, when our MySQL container starts.

Create a directory in your project called `data`, and add the following to your `volumes` tag under your `db` service:

.. code-block:: yaml

      - ./data:/docker-entrypoint-initdb.d # Place init .sql file(s) here.

When our MySQL container is started, Docker will mount our host directory, `data`, at the physical location `docker-entrypoint-initdb.d` inside our MySQL container.  The official MySQL (and MariaDB) containers have a startup script that executes every time a new container instance is created; the startup script will import any `.sql` or `.sql.gz` files it finds in `db` into the database we created, `drupal`.

7.  Start up your application stack
###################################

Issue the command:

    docker-compose up -d

Wait about 20 seconds, and issue the command:

    docker-compose ps

You should see something similar to the following::

          Name                    Command               State               Ports
    -----------------------------------------------------------------------------------------
    dockerdrop_db_1    docker-entrypoint.sh --cha ...   Up      3306/tcp
    dockerdrop_php_1   php-fpm                          Up      9000/tcp
    dockerdrop_web_1   nginx -g daemon off;             Up      443/tcp, 0.0.0.0:8000->80/tcp

You can now see we have a new container called `db` in our stack.

Your docker compose file should look as follows:

.. code-block:: yaml
   :linenos:

    version: '3'
    services:
      web:
        build: ./docker/nginx/
        ports:
          - "8000:80"
        volumes_from:
          - php
        depends_on:
          - php

      php:
        image: php:7.0-fpm
        expose:
          - 9000
        volumes:
          - .:/var/www/html

      db:
        image: mariadb:10.3.0
        environment:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: drupal
          MYSQL_USER: drupal
          MYSQL_PASSWORD: drupal
        command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci # The simple way to override the mariadb config.
        volumes:
          - mysql-data:/var/lib/mysql
          - ./data:/docker-entrypoint-initdb.d # Place init .sql file(s) here.

    volumes:
      mysql-data:
        driver: local
