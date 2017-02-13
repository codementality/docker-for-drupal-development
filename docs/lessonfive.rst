Lesson 5:  Customize PHP for Drupal
===================================

1: Create a custom Docker image for PHP
#######################################

Let's take a look at the PHP Information that is displayed by our website's index.php file currently.

If you scan down the page you'll notice that the "official" php container is missing some key php extensions that are needed to be able to run a Drupal site:

- A Database extension for MySQL
- an Image Library extension (GD)
- Opcode Cache (opcache)

I would recommend that you also install:

- Mcrypt (an encryption library)
- Iconv extension (Human Language and Character Encoding Support), needed for some mime support, UTF8, latin character support etc.

Some Drupal security extensions will leverage Mcrypt if it's available.

Drush uses compression / decompression libraries for some functions, so we'll install:

- Zip
- Unzip
- Zip PHP extension

Finally, since our MySQL container is separate from the container where PHP is located, you will need a MySQL client application for some of the Drush sql commands, so we'll install:

- MariaDB Client

Well, since you can't shell into a container and add php extensions and have them persist after the container is destroyed, your next best option is to create a custom PHP container, similar to what we did for NginX.

First thing, create the directory `docker/php` in your project root.

Inside that directory create a `Dockerfile`, and add the following:

.. code-block:: yaml
   :linenos:

    FROM php:7.0-fpm

    MAINTAINER Lisa Ridley "lhridley@gmail.com"

We're going to base our Dockerfile on the same container we are using to build our application stack currently.

2:  Add code to add the PHP extensions
######################################

If you check out the documentation for the Official PHP Docker container, you will see instructions for adding PHP extensions to the base container build.  There are some convenience applications in the official PHP container images that make this process fairly straightforward.

Let's install the dependencies for our extensions, and install the extensions themselves using the convenience applications provided in the official container.  Add the following to your Dockerfile, below the first set of lines:

.. code-block:: yaml
   :linenos:

    RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng12-dev \
        unzip \
        zip \
        mariadb-client \
        && docker-php-ext-install -j$(nproc) iconv mcrypt \
        && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install -j$(nproc) gd \
        && docker-php-ext-install pdo_mysql \
        && docker-php-ext-install zip \
        && docker-php-ext-install opcache \
        && docker-php-ext-install zip

If you've ever had to install packages on a Linux server or a virtual machine, these commands may look familiar to you.  The RUN keyword is a Docker command that is used to execute commands available from binaries installed in the base container, using the "shell" application (`/bin/sh`).

3:  Add Composer, Drush and a Drush Alias file
##############################################

Well, we are building a container to work with Drupal, so naturally we need Drush installed.  For the container to work with Drupal 8, we also need Composer.  Let's add those two applications to our container.  Add the following lines below the one already added to our `Dockerfile`:

.. code-block:: yaml
   :linenos:

    # Register the COMPOSER_HOME environment variable
    # Add global binary directory to PATH and make sure to re-export it
    # Allow Composer to be run as root
    # Composer version
    ENV COMPOSER_HOME /composer
    ENV PATH /composer/vendor/bin:$PATH
    ENV COMPOSER_ALLOW_SUPERUSER 1
    ENV COMPOSER_VERSION 1.2.3

    # Setup the Composer installer
    RUN curl -o /tmp/composer-setup.php https://getcomposer.org/installer \
        && curl -o /tmp/composer-setup.sig https://composer.github.io/installer.sig \
        && php -r "if (hash('SHA384', file_get_contents('/tmp/composer-setup.php')) !== trim(file_get_contents('/tmp/composer-setup.sig'))) { unlink('/tmp/composer-setup.php'); echo 'Invalid installer' . PHP_EOL; exit(1); }" \

    # Install Composer
        && php /tmp/composer-setup.php --no-ansi --install-dir=/usr/local/bin --filename=composer --version=${COMPOSER_VERSION} && rm -rf /tmp/composer-setup.php \
    # Install Prestissimo plugin for Composer -- allows for parallel processing of packages during install / update
        && composer global require "hirak/prestissimo:^0.3" \
        && chown -Rf www-data:www-data /composer \
        && mkdir /var/www/docroot \
        && chown -Rf www-data:www-data /var/www \
    # Install Drush
        && mkdir /usr/local/drush \
        && cd /usr/local/drush \
        && composer init --require=drush/drush:8.* -n \
        && composer config bin-dir /usr/local/bin \
        && composer install \
        && drush init -y \
    # Set up default location for drush alias files
        && mkdir -p /etc/drush/site-aliases

    # Copy drush alias file into image
    COPY default.aliases.drushrc.php /etc/drush/site-aliases/

    # Install Mailhog Sendmail support:
    RUN apt-get update -qq && apt-get install -yq git golang-go \
        && mkdir -p /opt/go \
        && export GOPATH=/opt/go \
        && go get github.com/mailhog/mhsendmail

4: Create your Drush alias file, and modify `docker-compose.yml`
################################################################

Create a file in your `docker/php` directory called `default.aliases.drushrc.php`, and add the following to it:

.. code-block:: php
   :linenos:

    <?php
    $aliases[isset($_SERVER['PHP_SITE_NAME']) ? $_SERVER['PHP_SITE_NAME'] : 'dev'] = [
      'root' => '/var/www/html/' . (isset($_SERVER['PHP_DOCROOT']) ? $_SERVER['PHP_DOCROOT'] : ''),
      'uri' => isset($_SERVER['PHP_HOST_NAME']) ? $_SERVER['PHP_HOST_NAME'] : 'localhost:8000',
    ];

5:  Add a php.ini base file, and an entrypoint shell script
###########################################################

Create a base php.ini file with some commonly adjusted settings in it, plus some we will need in our next lesson.  Create a file named `php.ini` in `docker/php` and include the following:

.. code-block:: ini
   :linenos:

    ; php.ini
    [php]
    memory_limit = 192M
    allow_url_include = On
    ;post_max_size =
    ;upload_max_filesize =
    ;max_execution_time =

    [opcache]
    opcache.enable = On
    opcache.validate_timestamps = 1
    opcache.revalidate_freq = 2
    opcache.max_accelerated_files = 20000
    opcache.memory_consumption = 64
    opcache.interned_strings_buffer = 16
    opcache.fast_shutdown = 1

    [mailhog]
    ; Mailhog php.ini settings.
    sendmail_path = "/opt/go/bin/mhsendmail --smtp-addr=mailhog:1025"

Now, we will add an Entrypoint script for our container.  Create a file called `docker-entrypoint.sh` in `docker/php` and include the following:

.. code-block:: bash
   :linenos:

    !/bin/bash

    set -eo pipefail

    if [ -n "$PHP_MEMORY_LIMIT" ]; then
         sed -i 's@^memory_limit.*@'"memory_limit = ${PHP_MEMORY_LIMIT}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_MAX_EXECUTION_TIME" ]; then
         sed -i 's@^;max_execution_time.*@'"max_execution_time = ${PHP_MAX_EXECUTION_TIME}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_POST_MAX_SIZE" ]; then
         sed -i 's@^;post_max_size.*@'"post_max_size = ${PHP_POST_MAX_SIZE}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    if [ -n "$PHP_UPLOAD_MAX_FILESIZE" ]; then
         sed -i 's@^;upload_max_filesize.*@'"upload_max_filesize = ${PHP_UPLOAD_MAX_FILESIZE}"'@' /usr/local/etc/php/conf.d/php.ini
    fi

    exec "$@"


Add these two files to our Dockerfile by inserting the following lines, and specify the executable to be run:

.. code-block:: YAML
   :linenos:


    # Add php.ini base file
    COPY php.ini /usr/local/etc/php/conf.d/php.ini

    # Add entrypoint script
    COPY docker-entrypoint.sh /usr/local/bin/
    # Make sure it's executable
    RUN chmod a+x /usr/local/bin/docker-entrypoint.sh

    ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

    CMD ["php-fpm"]


5:  Modify the `docker-compose.yml` file to use our custom image definition
###########################################################################

Open `docker-compose.yml`, and replace the following:

.. code-block:: ini
   :linenos:

    php:
      image: php:7.0-fpm
      expose:
        - 9000
      volumes:
        - ./web:/var/www/html/web

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

After your containers are up and running, navigate to `localhost:8000` and take a look at the information displayed.  You will now see that PHP has additional extensions installed for zip, iconv, mcrypt, pdo-mysql and gd, which were not installed previously.

Congratulations!  You have a complete Docker stack that is configured to support Drupal development.

Your `docker-compose.yml` file should look as follows:

.. code-block:: yaml
   :linenos:

    version: '2'
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
          NGINX_DOCROOT: www
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
        image: mariadb:10.1.21
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

    FROM php:7.0-fpm

    MAINTAINER Lisa Ridley "lhridley@gmail.com"

    RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng12-dev \
        unzip \
        zip \
        mariadb-client \
    && docker-php-ext-install -j$(nproc) iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install zip \
    && docker-php-ext-install opcache \
    && docker-php-ext-install zip

    # Register the COMPOSER_HOME environment variable
    # Add global binary directory to PATH and make sure to re-export it
    # Allow Composer to be run as root
    # Composer version
    ENV COMPOSER_HOME /composer
    ENV PATH /composer/vendor/bin:$PATH
    ENV COMPOSER_ALLOW_SUPERUSER 1
    ENV COMPOSER_VERSION 1.2.3

    # Setup the Composer installer
    RUN curl -o /tmp/composer-setup.php https://getcomposer.org/installer \
        && curl -o /tmp/composer-setup.sig https://composer.github.io/installer.sig \
        && php -r "if (hash('SHA384', file_get_contents('/tmp/composer-setup.php')) !== trim(file_get_contents('/tmp/composer-setup.sig'))) { unlink('/tmp/composer-setup.php'); echo 'Invalid installer' . PHP_EOL; exit(1); }" \

    # Install Composer
        && php /tmp/composer-setup.php --no-ansi --install-dir=/usr/local/bin --filename=composer --version=${COMPOSER_VERSION} && rm -rf /tmp/composer-setup.php \
    # Install Prestissimo plugin for Composer -- allows for parallel processing of packages during install / update
        && composer global require "hirak/prestissimo:^0.3" \
        && chown -Rf www-data:www-data /composer \
        && mkdir /var/www/docroot \
        && chown -Rf www-data:www-data /var/www \
    # Install Drush
        && mkdir /usr/local/drush \
        && cd /usr/local/drush \
        && composer init --require=drush/drush:8.* -n \
        && composer config bin-dir /usr/local/bin \
        && composer install \
        && drush init -y \
    # Set up default location for drush alias files
        && mkdir -p /etc/drush/site-aliases

    # Copy drush alias file into image
    COPY default.aliases.drushrc.php /etc/drush/site-aliases/

    # Install Mailhog Sendmail support:
    RUN apt-get update -qq && apt-get install -yq git golang-go \
        && mkdir -p /opt/go \
        && export GOPATH=/opt/go \
        && go get github.com/mailhog/mhsendmail

    # Add php.ini base file
    COPY php.ini /usr/local/etc/php/conf.d/php.ini

    # Add entrypoint script
    COPY docker-entrypoint.sh /usr/local/bin/
    # Make sure it's executable
    RUN chmod a+x /usr/local/bin/docker-entrypoint.sh

    ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

    CMD ["php-fpm"]
