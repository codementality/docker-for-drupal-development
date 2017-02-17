Lesson 6:  Add MailHog and Selenium
===================================

1: Add Mailhog to your stack
############################

Mailhog is an email testing tool for developers that allows you to fully test mail functions for a website without having a mail server installed.  Mailhog can act as a SMTP proxy, and will capture outgoing email from an application configured to use it.

Conveniently, the maintainers of Mailhog have provided us with....Docker containers!

To add Mailhog to your application stack, open your `docker-compose.yml` file and insert the following under the `services` key:

.. code-block:: yaml
   :linenos:

    mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "8002:8025"

This will add a MailHog container to your application stack, and will make the MailHog GUI available on port 8002.

2: Configure PHP to route outgoing mail to our MailHog container
################################################################

By default, PHP uses sendmail to send out email from a web service; however, we don't have sendmail installed in our PHP container.  The MailHog community has written a sendmail replacement that is specifically designed to route mail to a MailHog instance.

We have already inserted the code into the PHP container to install the MailHog Sendmail replacement, which will send mail to the MailHog container in our stack.

That code specifically is as follows:

.. code-block:: yaml
   :linenos:

    # Install Mailhog Sendmail support:
    RUN apt-get update -qq && apt-get install -yq git golang-go \
        && mkdir -p /opt/go \
        && export GOPATH=/opt/go \
        && go get github.com/mailhog/mhsendmail

We also added the configuration to our php.ini file to configure PHP to use the Sendmail replacement:

.. code-block:: yaml
   :linenos:

    [mailhog]
    ; Mailhog php.ini settings.
    sendmail_path = "/opt/go/bin/mhsendmail --smtp-addr=mailhog:1025"

By default, the MailHog container exposes port 1025, so we will route our mail to MailHog via it's alias (mailhog) on port 1025.  This will route over the stack's network.

3: Add a Selenium container to our stack for testing purposes
#############################################################

If you are in the practice of writing user acceptance and unit tests for your applications, and you like to test your applications as you write them, you can implement a testing process using Selenium to execute your user acceptance tests.

As you are probably aware, Drupal has a plugin for the Symfony2 Behat module that enhances the available user acceptance testing steps that Behat provides for testing web applications.  While we won't go through the hows and whys of writing user acceptance tests as part of this training, we will implement a basic user acceptance test suite to execute enough tests to have a workable model for building our application on Travis-CI, and we'll use Selenium to execute those tests.

If you've ever installed Selenium for testing purposes you're probably aware that it is a cumbersome and time consuming application to install and configure.  The Selenium community has been kind enough to provide us with pre-built Docker container images that we can leverage for use in our projects for executing automated tests driven by Selenium.

To add a Selenium container to our stack, edit the `docker-compose.yml` file and add the following beneath the `services` key:

.. code-block:: yaml
   :linenos:

    selenium:
      image: selenium/standalone-firefox:2.53.0

The selenium container exposes port 4444 by default.  We will need this information when we configure Behat to use Selenium for testing.

That's it!  No length builds, no configuration, other than to configure Behat to use the Selenium instance in our container to execute tests.

4: Recap
########
Now, our `docker-compose.yml` file looks like this:

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

      mailhog:
        image: mailhog/mailhog:latest
        ports:
          - "8002:8025"

      selenium:
        image: selenium/standalone-firefox:2.53.0

    volumes:
      mysql-data:
        driver: local
