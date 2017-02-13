Lesson 7:  Add a Makefile for Convenience
=========================================

Now, our stack is complete.  However, some of the commands that we will need to execute to do things like run Drush commmands, run Composer, and execute tests are rather lengthy.

For example, to clear the cache using Drush, the command would be:

    docker-compose exec php /path/to/drush @default.dev cc all

Let's break that down:

* `docker-compose`  is the executable you will use to run the command
* `exec`  tells Docker Compose that you are going to execute a command in a running container
* `php` is the alias of the container in which you're going to run the command
* `/path/to/drush` is the command you're executing
* `@default.dev` is the drush alias file you're utilizing to locate your Drupal instance
* `cc all` is the drush command you wish to execute.

That's quite a bit to type in every time you want to clear the cache.

You can make life easier on yourself by settin up a Makefile that can be used to execute commands to not only build your project (which will make Travis integration easier), but to execute commonly used commands, such as Drush commands.

1:  Create a file called `Makefile` in the root of your project
###############################################################

A Makefile simply executes bash commands via an executable called "make", which parses the targets in a Makefile and executes the commands in each target in order.

2:  Insert the following code:
##############################

Insert the followning code in the Makefile, and save it:

.. code-block:: bash
   :linenos:


    init:
        if [ ! -f "data/database.sql" ]; then make download-seed-db; fi
        if [ ! -f "config/settings/settings.php" ] || [ ! -f "config/settings.settings.local.php" ]; then make download-drupal-settings; fi
        if [ ! -d "tests" ]; then make download-behat-tests; fi
        if [ ! -d "config/sync" ]; then make download-drupal-config; fi
        if [ ! -f "www/composer.json" ]; then make download-drupal-site; fi
        docker-compose up -d --build
        echo "Waiting for database to initialize"; sleep 15
        docker-compose ps
        docker-compose exec -T php composer require drush/drush:8.* -n --working-dir=/var/www/html/www
        docker-compose exec -T php composer update --working-dir=/var/www/html/www
        -make update-tests
        docker-compose exec -T php chmod a+w /var/www/html/www/sites/default
        docker-compose exec -T php cp config/settings/settings.php www/sites/default/settings.php
        docker-compose exec -T php cp config/settings/settings.local.php www/sites/default/settings.local.php
        if [ ! -d "www/sites/default/files" ]; then docker-compose exec -T php mkdir /var/www/html/www/sites/default/files; fi
        docker-compose exec -T php chmod a+w /var/www/html/www/sites/default/files
    ifdef TRAVIS
        docker-compose exec php /bin/bash -c "chown -Rf 1000:1000 /var/www"
    endif
        docker-compose exec -T php /bin/bash -c "ls -l /var/www/html/www"
        docker-compose exec -T php www/vendor/bin/drush @default.dev status
        @make provision

    download-seed-db:
        curl -o data/database.sql https://s3.us-east-2.amazonaws.com/dockerdrop/database.sql

    down:
        docker-compose down
        docker-compose ps

    clean-data:
        docker volume rm dockerdrop_db

    provision:
        @echo "Running database updates..."
        @docker-compose exec -T php www/vendor/bin/drush @default.dev updb
        @echo "Running entity updates..."
        @docker-compose exec -T php www/vendor/bin/drush @default.dev entup
        @echo "Importing configuration..."
        @docker-compose exec -T php www/vendor/bin/drush @default.dev cim
        @echo "Running reverting features..."
        -docker-compose exec -T php www/vendor/bin/drush @default.dev fra -y
        @echo "Resetting cache..."
        @docker-compose exec -T php www/vendor/bin/drush @default.dev cr

    phpcs:
        docker-compose exec -T php tests/bin/phpcs --config-set installed_paths tests/vendor/drupal/coder/coder_sniffer
        # Drupal 8
        docker-compose exec -T php tests/bin/phpcs --standard=Drupal www/modules/* www/themes/* --ignore=*.css --ignore=*.css,*.min.js,*features.*.inc,*.svg,*.jpg,*.png,*.json,*.woff*,*.ttf,*.md,*.sh --exclude=Drupal.InfoFiles.AutoAddedKeys

    behat:
        docker-compose exec -T php tests/bin/behat -c tests/behat.yml --tags=~@failing --colors -f progress

    update-tests:
        docker-compose exec -T php composer update --working-dir=/var/www/html/tests
        docker-compose exec -T php tests/bin/behat -c tests/behat.yml --init

    download-drupal-settings:
        curl -o config/settings/settings.php https://s3.us-east-2.amazonaws.com/dockerdrop/settings.php
        curl -o config/settings/settings.local.php https://s3.us-east-2.amazonaws.com/dockerdrop/settings.local.php

    download-drupal-site:
        rm -Rf www
        git clone --depth=1 https://github.com/codementality/drupal8-standard.git www
        rm -Rf www/.git

    download-drupal-config:
        git clone --depth=1 https://github.com/codementality/dockerdrop-config.git config/sync
        rm -Rf config/sync/.git

    download-behat-tests:
        git clone --depth=1 https://github.com/codementality/dockerdrop-tests.git tests
        rm -Rf tests/.git
        curl -o .travis.yml https://s3.us-east-2.amazonaws.com/dockerdrop/traviscfg.yml

For the most part this file is pretty agnostic; however there are a few commands that will need to be customized for each project:

* If the DOCROOT you're using is something other than "www", you'll need to replace all occurrences of "www" with your project's docroot.
* If your project resides in a folder other than "dockerdrop", you'll need to replace all occurrences of "dockerdrop" with the name of your project folder.
* There are several statements in the `init:` target that are specific for this project.  You will want to modify and/or remove those lines, and their associated targets.

This file is here for your convenience, so customize it to your liking for your projects.

This file will also make configuring Travis much easier.
