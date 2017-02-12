Prerequisites
=============

.. toctree::
   :maxdepth: 2

   :doc:`prerequisites`
   :doc:`lessonone`

Please check the systems requirements for your preferred operating system.  Docker is supported on the following platforms:

* Windows (Windows 10, 64-bit, with hardware / software limitations)
* Mac OS X (Mountain Lion or later, 64-bit, with hardware limitations)
* Linux (64-bit, kernel version 3.10 or higher, with hardware limitations)

###########
Linux Users
###########


* Install the Docker Engine:  https://docs.docker.com/engine/installation/

.. Note::
   * Docker only works on a 64-bit Linux installation.
   * Docker requires version 3.10 or higher of the Linux kernel.

See the Installation instructions for your flavor of Linux at the link above.

* Install Docker-Compose:  https://docs.docker.com/compose/install/
* Install Make (we will be using a Makefile as part of this class)

##############
Mac OS X Users
##############

* Install Docker for Mac:  https://docs.docker.com/engine/installation/mac/

.. Note::
   * Mac must be a 2010 or newer model, with Intel’s hardware support for memory management unit (MMU) virtualization; i.e., Extended Page Tables (EPT) macOS 10.10.3 Yosemite or newer
   * At least 4GB of RAM
   * VirtualBox prior to version 4.3.30 must NOT be installed (it is incompatible with Docker for Mac). Docker for Mac will error out on install in this case. Uninstall the older version of VirtualBox and re-try the install.

################
Windows 10 Users
################

* Install Docker for Windows:  https://docs.docker.com/engine/installation/windows/
* If you're running an up to date version of Windows 10 (you have the Anniversary Update installed), install the Linux Bash Shell:  http://www.howtogeek.com/249966/how-to-install-and-use-the-linux-bash-shell-on-windows-10/
* If "make" is not available in the Linux Bash Shell, install it with `sudo apt-get install make` or `sudo apt-get install build-essential`

.. Note::
   * 64bit Windows 10 Pro, Enterprise and Education (1511 November update, Build 10586 or later).
   * The Hyper-V package must be enabled. The Docker for Windows installer will enable it for you, if needed.

#########
All Users
#########

* Git, for your operating system
* Your favorite IDE / code editor
* A Github account, with a public key linked to the account for pushing code from your computer.
* A Travis-ci.org account (for the Travis integration / CI/CD portion of the training workshop), linked to your Github account
* A Docker Hub account (https://hub.docker.com/), linked to your GitHub account.

We will be using a Slack Channel to share gists during the workshop; if you didn't receive an invitation, see your instructor.

Slack has a web UI available.  If you want to join the Slack channel (we will provide a link during the class) using a desktop client, install the desktop Slack client for your operating system:  https://slack.com/downloads

.. Note::
   * This is an optional installation.
   * If you prefer, you can use the web interface for Slack.
