What is Docker?
===============
Docker is an open platform for developing, shipping, and running applications.  This is straight from the Docker website.

Docker was developed by a platform-as-a-service company in France, dotCloud, which was a subsidiary of cloudControl, a PAAS company in Berlin, Germany.  The Docker project was open-sourced in 2013.  cloudControl filed for bankruptcy in 2016.

Docker was originally built aroud LXC, or Linux Containers; however with version 0.9 of the Docker project, LXC was replaced with the `libcontainer` library written in GoLang, or Go as it's known now.  Docker has since spun `libcontainer` out as a separate project, now called `runc`, and has contributed it to the Open Container Initiative (https://www.opencontainers.org).  The Open Container Initiative is comprised of people from Docker, Redhat, Google, CoreOS and some independent developers.

The primary objective of the Open Container Initiative is to provide a structure for the creation of open industry standards with relation to container formats and runtime specifications.

Containers and Docker
#####################
Linux Containers, or LXC, have been around for quite some time.  LXC relies on linux kernel cgroups, which were introduced in Linux Kernel version 2.6.24.  LXC provides an operating system level of virtualization that faciitates the creation of bundles of software into a self-contained application that included all libraries and binaries needed to function.  Hoever, it wasn't until Linux Kernel version 3.8, and the subsequent release of LXC version 1.0, that LXCs could run as unprivileged applications.

Configuring LXCs has traditionally been an arduous task.  Docker, and many of Docker's supporting toolsets, has brought LXCs, and subsequently libcontainer applications, into the mainstream by providing an abstraction layer and API that lowered the barrier to entry, putting containerized applications within the reach of a technology population with a much broader and less specialized skillset; people like you and me.  This has led to a huge level of interest and adoption of Docker by the open source technology community.

Docker and Development
######################
As developers, Docker provides a virtualization toolset that allows us to easily construct project stacks comprised of containerized microservices in a modular fashion.  This greatly simplifies the configuration of development environment stacks for client projects.  Think of Docker containers as software application Legos, and each stack as a different Lego project.  You assemble your stack from the containers you need to provide the services you're seeking to develop your client application around.

This allows us to easily construct an application stack from reusable components, and assemble each stack from only the components we need for that particular project.

Purpose of this course
######################
The purpose of this course is to provide you, as a developer, with the basic fundamentals to begin utlizing Docker and Docker Compose in your development workflow.  More specifically, this course is designed to provide you with the ability to utilize Docker in your Drupal development workflow process.