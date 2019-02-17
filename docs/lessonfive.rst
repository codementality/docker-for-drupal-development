Lesson 5:  Customize PHP for Drupal
===================================

1: Create a custom Docker image for PHP
#######################################

Let's take a look at the PHP information that is displayed by our website's index.php file.

If you scan down the page you'll notice that the "official" php container is missing some key php extensions that are needed to be able to run a Drupal site:

- A Database extension for MySQL
- an Image Library extension (GD)
- Opcode Cache (opcache)
- Mbstring (for multilingual and UTF-8 / UCS-2 encoding)

I would recommend that you also install:

- Mcrypt (an encryption library, which must be installed using PECL)
- Iconv extension (Human Language and Character Encoding Support), needed for some mime support, UTF8, UCS-2, latin character support etc.

Some Drupal security extensions will leverage Mcrypt if it's available.

Drush uses compression / decompression libraries for some functions, so we'll install:

- Zip
- Unzip
- Zip PHP extension

Drupal 8 and the Composer Project for Drupal both use yaml files, so we'll also install:

-PHP Yaml

Finally, since our MySQL container is separate from the container where PHP is located, you will need a MySQL client application for some of the Drush sql commands, so we'll install:

- MariaDB Client

Well, since you can't shell into a container and add php extensions and have them persist after the container is destroyed, your next best option is to create a custom PHP container, similar to what we did for NginX.

First thing, create the directory `docker/php` in your project root.

Inside that directory create a `Dockerfile`, and add the following:

.. code-block:: yaml
   :linenos:

    FROM php:7.2-fpm

    MAINTAINER Lisa Ridley "lisa@codementality.com"

We're going to base our Dockerfile on the same container we are using to build our application stack currently.

While we are at it, go ahead and create the directory `docker/php/conf.d` as well, we'll be using that later.

2:  Add code to add the PHP extensions
######################################

If you check out the documentation for the Official PHP Docker container, you will see instructions for adding PHP extensions to the base container build.  There are some convenience applications in the official PHP container images that make this process fairly straightforward.

Let's install the dependencies for our extensions, and install the extensions themselves using the convenience applications provided in the official container.  Add the following to your Dockerfile, below the first set of lines:

.. code-block:: yaml
   :linenos:

    RUN apt-get update && apt-get install -yq \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev \
        libyaml-dev \
        unzip \
        zip \
        mariadb-client \
        && rm -rf /var/lib/apt/lists/* \
        && apt autoremove -y \
        && docker-php-ext-install -j$(nproc) iconv \
        && docker-php-ext-configure gd --with-png-dir=/usr --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install -j$(nproc) gd \
        && docker-php-ext-install pdo_mysql \
        && docker-php-ext-install zip \
        && docker-php-ext-install opcache \
        && docker-php-ext-install zip \
        && docker-php-ext-install mbstring \
        && pecl install mcrypt-1.0.1 yaml \
        && docker-php-ext-enable mcrypt yaml \

If you've ever had to install packages on a Linux server or a virtual machine, these commands may look familiar to you.  The RUN keyword is a Docker command that is used to execute commands available from binaries installed in the base container, using the "shell" application (`/bin/sh`).

Since we are installing the `opcache` extension, we'll need to configure it, so after the above lines, as part of the `RUN` directive, include the following::

      # set recommended PHP.ini settings
      # see https://secure.php.net/manual/en/opcache.installation.php
          && { \
              echo 'opcache.memory_consumption=128'; \
              echo 'opcache.interned_strings_buffer=8'; \
              echo 'opcache.max_accelerated_files=4000'; \
              echo 'opcache.revalidate_freq=0'; \
              echo 'opcache.fast_shutdown=1'; \
              echo 'opcache.enable_cli=1'; \
          } > /usr/local/etc/php/conf.d/opcache-recommended.ini
      #


3:  Add instructions to copy any php config files, and add Composer
###################################################################

Well, we are building a container to work with Drupal 8.  For the container to work with Drupal 8, we need Composer.  Add the following lines below the one already added to our `Dockerfile`:

.. code-block:: yaml
   :linenos:

    # Copy PHP configs.
    COPY conf.d/* /usr/local/etc/php/conf.d/
    RUN chmod 644 /usr/local/etc/php/conf.d/* \
        && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
        && composer --version \
    # Install Prestissimo plugin for Composer -- allows for parallel processing of packages during install / update
        && composer global require "hirak/prestissimo:^0.3"

4.  Install MHSendmail support
##############################

We will be using an application calle "MHSendmail" to capture outgoing mail from our application, which will redirect mail to an application called "MailHog", which we'll include in our stack in a bit.  To install MHSendmail and configure PHP to use MHSendMail for outgoing mail routing, include the following in your Dockerfile below the lines abovw:

    RUN apt-get update -qq && apt-get install -yq git golang-go \
        && mkdir -p /opt/go \
        && export GOPATH=/opt/go \
        && go get github.com/mailhog/mhsendmail \
    # Add configuration to PHP for MHSendmail
        && { \
          echo 'sendmail_path = "/opt/go/bin/mhsendmail --smtp-addr=mailhog:1025"'; \
        } > /usr/local/etc/php/conf.d/mailhog.ini

5:  Add a `php.ini` base file, and an entrypoint shell script
#############################################################

Create a base `php.ini` file with some commonly adjusted settings in it, plus some we will need in our next lesson.  Create a file named `php.ini` in `docker/php/conf.d` and include the following:

.. code-block:: ini
   :linenos:

    ; php.ini
    [php]
    memory_limit = 192M
    allow_url_include = On
    ;post_max_size =
    ;upload_max_filesize =
    ;max_execution_time =

Now, we will add an Entrypoint script for our container.  Create a file called `docker-entrypoint.sh` in `docker/php` and include the following:

.. code-block:: bash
   :linenos:

    #!/bin/bash

    set -eo pipefail

    if [ -n "$PHP_MEMORY_LIMIT" ]; then
         sed -i 's@^memory_limit.*@'"memory_limit = ${PHP_MEMORY_LIMIT}"'@' \
     /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_MAX_EXECUTION_TIME" ]; then
         sed -i 's@^;max_execution_time.*@'"max_execution_time = \
         ${PHP_MAX_EXECUTION_TIME}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_POST_MAX_SIZE" ]; then
         sed -i 's@^;post_max_size.*@'"post_max_size = ${PHP_POST_MAX_SIZE}"'@' \
         /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_UPLOAD_MAX_FILESIZE" ]; then
         sed -i 's@^;upload_max_filesize.*@'"upload_max_filesize = \
         ${PHP_UPLOAD_MAX_FILESIZE}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    exec "$@"


Add these two files to our Dockerfile by inserting the following lines, and specify the executable to be run:

.. code-block:: YAML
   :linenos:

    # add PHP override config file
    COPY conf.d/php.ini /usr/local/etc/php/conf.d/php.ini
    # Add entrypoint script
    COPY docker-entrypoint.sh /usr/local/bin/
    # Make sure it's executable
    RUN chmod a+x /usr/local/bin/docker-entrypoint.sh

    ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

    CMD ["php-fpm"]

6:  Modify the `docker-compose.yml` file to use our custom image definition
###########################################################################

Open `docker-compose.yml`, and replace the following:

.. code-block:: ini
   :linenos:

    php:
      image: php:7.2-fpm
      expose:
        - 9000
      volumes:
        - ./web:/var/www/html;

with:

.. code-block:: yaml
   :linenos:

    php:
      build: ./docker/php/
      expose:
        - 9000
      volumes:
        - .:/var/www/html
      depends_on:
        - db
      environment:
        PHP_MEMORY_LIMIT: 256M
        PHP_MAX_EXECUTION_TIME: 120
        # If you set this,make sure you also set it for Nginx
        PHP_POST_MAX_SIZE: 20M
        PHP_UPLOAD_MAX_FILESIZE: 20M
        # used by Drush Alias; if not specified Drush defaults to dev
        PHP_SITE_NAME: dev
        # used by Drush alias; if not specified Drush defaults to localhost:8000
        PHP_HOST_NAME: localhost:8000
        # Make this the same for Nginx
        PHP_DOCROOT: www

and save it.

Now, execute `docker-compose up -d --build`, and let's see what happens.

After your containers are up and running, navigate to `localhost:8000` and take a look at the information displayed.  You will now see that PHP has additional extensions installed for zip, iconv, mcrypt, pdo-mysql, gd and mbstring, which were not installed previously.

Congratulations!  You have a complete Docker stack that is configured to support Drupal development.

Your `docker-compose.yml` file should look as follows:

.. code-block:: yaml
   :linenos:

    version: '3'

    services:
      web:
        build: ./docker/nginx/
        ports:
          - "8000:80"
        volumes:
          - .:/var/www/html
        depends_on:
          - php
        environment:
         #Make this the same for PHP
         NGINX_DOCROOT: web
         NGINX_SERVER_NAME: localhost
         # Set to the same as the PHP_POST_MAX_SIZE, but use lowercase "m"
         NGINX_MAX_BODY_SIZE: 20m

      php:
        build: ./docker/php/
        expose:
          - 9000
        volumes:
          - .:/var/www/html
        depends_on:
          - db
        environment:
          PHP_MEMORY_LIMIT: 256M
          PHP_MAX_EXECUTION_TIME: 120
          # If you set this,make sure you also set it for Nginx
          PHP_POST_MAX_SIZE: 20M
          PHP_UPLOAD_MAX_FILESIZE: 20M
          # used by Drush Alias; if not specified Drush defaults to dev
          PHP_SITE_NAME: dev
          # used by Drush alias; if not specified Drush defaults to localhost:8000
          PHP_HOST_NAME: localhost:8000
          # Make this the same for Nginx
          PHP_DOCROOT: www

      db:
        image: mariadb:10.4.2
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

And your php `Dockerfile should look like:

.. code-block:: yaml
   :linenos:

    FROM php:7.2-fpm

    MAINTAINER Lisa Ridley "lisa@codementality.com"

    RUN apt-get update && apt-get install -yq \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev \
        libyaml-dev \
        unzip \
        zip \
        mariadb-client \
        && rm -rf /var/lib/apt/lists/* \
        && apt autoremove -y \
        && docker-php-ext-install -j$(nproc) iconv \
        && docker-php-ext-configure gd --with-png-dir=/usr --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install -j$(nproc) gd \
        && docker-php-ext-install pdo_mysql \
        && docker-php-ext-install zip \
        && docker-php-ext-install opcache \
        && docker-php-ext-install zip \
        && docker-php-ext-install mbstring \
        && pecl install mcrypt-1.0.1 yaml \
        && docker-php-ext-enable mcrypt yaml
    RUN chmod 644 /usr/local/etc/php/conf.d/* \
        && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
        && composer --version
    # Install Mailhog Sendmail support:
    RUN apt-get update -qq && apt-get install -yq git golang-go \
        && mkdir -p /opt/go \
        && export GOPATH=/opt/go \
        && go get github.com/mailhog/mhsendmail \
    # Add configuration to PHP for MHSendmail
        && { \
          echo 'sendmail_path = "/opt/go/bin/mhsendmail --smtp-addr=mailhog:1025"'; \
        } > /usr/local/etc/php/conf.d/mailhog.ini
    # add PHP override config file
    COPY conf.d/php.ini /usr/local/etc/php/conf.d/php.ini
    # Add entrypoint script
    COPY docker-entrypoint.sh /usr/local/bin/
    # Make sure it's executable
    RUN chmod a+x /usr/local/bin/docker-entrypoint.sh

    ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

    CMD ["php-fpm"]
