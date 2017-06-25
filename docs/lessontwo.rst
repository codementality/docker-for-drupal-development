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

2:  Edit the `image` designator to pin it to version 1.12.0
###########################################################

Edit the line from Step 1 to read as follows:

.. code-block:: yaml
   :linenos:
   :emphasize-lines: 4

   version: '3'
   services:
     web:
       image: nginx: 1.12.0
       ports:
         - 8000:80

This "pins" your stack build to use this specific version of the official NginX container.  The available versions can be seen on Docker Hub, at `https://hub.docker.com/r/library/nginx/tags/`.

3:  Save the changes and build your stack
#########################################

Save the revised `docker-compose.yml` file, and issue the following command::

`docker-compose up -d`

You should have seen output similar to the following::


    Creating network "dockerdrop_default" with the default driver
    Pulling web (nginx:1.12.0)...
    1.12.0: Pulling from library/nginx
    e6e142a99202: Already exists
    60807a6589a7: Pull complete
    55c917c49935: Pull complete
    Digest: sha256:aea0e686832d38c32a33ea6c6fcd0070598d7f09dce33d3bf7b2ce27b347f600
    Status: Downloaded newer image for nginx:1.12.0
    Creating dockerdrop_web_1 ...
    Creating dockerdrop_web_1 ... done    Creating dockerdrop_web_1

Notice the 4th line of the output above::

    e6e142a99202: Already exists

When Docker downloads an image, that image is comprised of several "layers".  Each layer is comprised of a set of instructions in the Dockerfile that generates the components that make up a particular image.

Each "layer" is stored on Docker Hub with a hash.  Whenever possible, Docker will "reuse" layers across multiple containers.

In our case, both the "latest" version and version "1.12.0" start with the official "debian/jessie" Docker image, so naturally there will be some commonality between the two.  Since we had started with the "latest" image in our build, Docker had already downloaded the layers for the `nginx:latest` image and cached them locally.

There is one layer that is common between `nginx:latest` and `nginx:1.12.0`, which Docker identified via its hash, so Docker doesn't download that layer again; it simply downloads the ones where there are differences, and locally constructs the "nginx:1.12.0" image from those layers.

4:  Delete the `nginx:latest` image from your cache
###################################################

This is an optional step, but helps keep your local Docker cache from becoming cluttered with unused images.  Issue the following command::

    docker images

You should see something similar to the following::

    REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
    nginx                          1.12.0              313ec0a602bc        2 days ago          107MB
    nginx                          latest              c246cd3dd41d        2 days ago          107MB


The IMAGE ID is the hash assigned to each image stored in your local cache.  To delete the `nginx:latest` image, issue the following command, replacing the hash for `nginx:latest` with the one shown on your local machine::

   docker rmi c246cd3dd41d <== replace hash

Now when you issue the command `docker images` you should only see the `nginx:1.12.0` image.
