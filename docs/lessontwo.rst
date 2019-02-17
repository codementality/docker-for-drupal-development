Lesson 2:  Pinning a Version
============================

1: Open `docker-compose.yml` in your editor / IDE
#################################################

Open the `docker-compose.yml` file and take a look at the `image` designator:

.. code-block:: yaml
   :linenos:
   :emphasize-lines: 4

       version: '3'
       services:
         web:
           image: nginx:latest
           ports:
             - 8000:80

This line designates the image to be used in this build.  The format of this line is `<container>:<version>`.  For Docker containers, the "latest" version is usually the "bleeding edge" image, and is subject to change.

2:  Edit the `image` designator to pin it to version 1.14.2
###########################################################

Edit the line from Step 1 to read as follows:

.. code-block:: yaml
   :linenos:
   :emphasize-lines: 4

   version: '3'
   services:
     web:
       image: nginx: 1.14.2
       ports:
         - 8000:80

This "pins" your stack build to use this specific version of the official NginX container.  The available versions can be seen on Docker Hub, at `https://hub.docker.com/_/nginx?tab=tags`.

3:  Save the changes and build your stack
#########################################

Save the revised `docker-compose.yml` file, and issue the following command::

`docker-compose up -d`

You should have seen output similar to the following::

   Pulling web (nginx:1.14.2)...
   1.14.2: Pulling from library/nginx
   6ae821421a7d: Already exists
   7edc8c3fac48: Pull complete
   a19c4a4a77fe: Pull complete
   Recreating dockerdrop_web_1 ... done

Notice the 3th line of the output above::

   6ae821421a7d: Already exists

When Docker downloads an image, that image is comprised of several "layers".  Each layer is comprised of a set of instructions in the Dockerfile that generates the components that make up a particular image.

Each "layer" is stored on Docker Hub with a hash.  Whenever possible, Docker will "reuse" layers across multiple containers.

In our case, both the "latest" version and version "1.14.2" start with the official "debian/stretch-slim" Docker image, so naturally there will be some commonality between the two.  Since we had started with the "latest" image in our build, Docker had already downloaded the layers for the `nginx:latest` image and cached them locally.

There is one layer that is common between `nginx:latest` and `nginx:1.14.2`, which Docker identified via its hash, so Docker doesn't download that layer again; it simply downloads the ones where there are differences, and locally constructs the "nginx:1.14.2" image from those layers.

To list the images you have locally, execute the following command::

    docker images

You should see something similar to the following::

    REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
    nginx                              1.14.2              1293e2b0a1af        11 days ago         109MB
    nginx                              latest              7042885a156a        7 weeks ago         109MB

The IMAGE ID is the hash assigned to each image stored in your local cache.  At the time of this writing, `latest` image is `1.15.8` using the `debian:stretch-slim` base image, so their size very similar, but their hash (seen under IMAGE ID) is different.
