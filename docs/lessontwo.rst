Lesson 2:  Pinning a Version
============================

1: Open `docker-compose.yml` in your editor / IDE
#################################################

Open the `docker-compose.yml` file and take a look at the `image` designator:

.. code-block:: yaml
   :linenos:
   :emphasize-lines: 4

       version: '2'
       services:
         web:
           image: nginx:latest
           ports:
             - 8000:80

This line designates the image to be used in this build.  The format of this line is `<container>:<version>`.  For Docker containers, the "latest" version is usually the "bleeding edge" image, and is subject to change.

2:  Edit the `image` designator to pin it to version 1.10.3
###########################################################

Edit the line from Step 1 to read as follows:

.. code-block:: yaml
   :linenos:
       :emphasize-lines: 4

           version: '2'
           services:
             web:
               image: nginx: 1.10.3
               ports:
                 - 8000:80

This "pins" your stack build to use this specific version of the official NginX container.  The available versions can be seen on Docker Hub, at `https://hub.docker.com/r/library/nginx/tags/`.

3:  Save the changes and build your stack
#########################################

Save the revised `docker-compose.yml` file, and issue the following command:

`docker-compose up -d`

You should have seen output similar to the following:


    Creating network "dockerdrop_default" with the default driver
    Pulling web (nginx:1.10.3)...
    1.10.3: Pulling from library/nginx
    386a066cd84a: Already exists
    21413bff969a: Pull complete
    eee080e089c4: Pull complete
    Digest: sha256:eb7e3bbd8e3040efa71d9c2cacfa12a8e39c6b2ccd15eac12bdc49e0b66cee63
    Status: Downloaded newer image for nginx:1.10.3
    Creating dockerdrop_web_1


Notice the 4th line of the output above:

    386a066cd84a: Already exists

When Docker downloads an image, that image is comprised of several "layers".  Each layer is comprised of a set of instructions in the Dockerfile that generate the components that make up a particular image.

Each "layer" is stored on Docker Hub with a hash.  Whenever possible Docker will "reuse" layers across multiple containers.

In our case, both the "latest" version and version "1.10.3" start with the official "debian/jessie" Docker image, so naturally there will be some commonality between the two.  Since we had started with the "latest" image in our build, Docker had already downloaded the layers for the `nginx:latest` image and cached them locally.

There is one layer that is common between `nginx:latest` and `nginx:1.10.3`, which Docker identified via its hash, so Docker doesn't download that layer again; it simply downloads the ones where there are differences.

4:  Delete the `nginx:latest` image from your cache
###################################################

This is an optional step, but helps keep your local Docker cache from becoming cluttered with unused images.  Issue the following command:

    docker images

You should see something similar to the following:


    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    nginx               1.10.3              5acd1b9bc321        2 weeks ago         180.7 MB
    nginx               latest              05a60462f8ba        2 weeks ago         181.5 MB


The IMAGE ID is the hash assigned to each image stored in your local cache.  To delete the `nginx:latest` image, issue the following command, replacing the hash for `nginx:latest` with the one shown on your local machine:

   docker rmi 05a60462f8ba <== replace hash

Now when you issue the command `docker images` you should only see the `nginx:1.10.3` image.
