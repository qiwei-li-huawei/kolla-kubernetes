.. host-setup:

=================================
Kolla Kubernetes Host Setup Guide
=================================

Dependencies
============

=====================   ===========  ===========  =========================
Component               Min Version  Max Version  Comment
=====================   ===========  ===========  =========================
Ansible                 2.00         none         On deployment host
Docker                  1.10         none         On target nodes
Docker Python           1.6.0        none         On target nodes
Python Jinja2           2.8.0        none         On deployment host
Kubernetes              1.3.0        none         On all hosts
=====================   ===========  ===========  =========================

.. NOTE:: Kolla (which provides the templating) is sensitive about the
  Ansible version.  Mainline currently requires 2.0.x or above.

Installing Docker
=================

Since Docker is required to build images as well as be present on all deployed
targets, the Kolla community recommends installing the official Docker, Inc.
packaged version of Docker for maximum stability and compatibility with the
following command:

.. NOTE:: Docker 1.11.0 is not compatible with Kubernetes due to some issues in
  Docker. The below command will install the latest Docker and revert back to
  1.10.3.  For different Debian or Ubuntu distributions, you may need to use
  ``apt-cache madison docker-engine`` to get the correct version.

::

    # Install Docker
    curl -sSL https://get.docker.io | bash

Setup Docker
============

Docker needs to run with the root filesystem as shared in order for
Neutron to function in 'thin' containers. The reason for that is mount
propogation.  Mounts need to be shared so the network namespaces are
shared among the host and the Neutron containers.

For CentOS and other systemd distros, change MountFlags from "slave"
to "shared" and restart Docker.

::

   # CentOS (and other systemd distros)
   cat > /etc/systemd/system/docker.service <<EOF
   .include /usr/lib/systemd/system/docker.service
   [Service]
   MountFlags=shared
   EOF

   # Restart the Docker daemon
   systemctl daemon-reload
   systemctl start docker

For Ubuntu 14.04 LTS, add a command to /etc/rc.local to mark the root
filesystem as shared upon startup.

::

   # Ubuntu (and other non-systemd distros)
   # Edit /etc/rc.local to add:
   mount --make-shared /

   # Ensure the mount is shared
   sudo sh /etc/rc.local

Kubernetes Setup
================

A user can show up with an already running Kubernetes or follow
the :doc:`minikube` for help on getting Kubernetes setup.  The minikube
doc is the most well tested method.

In addition there's the :doc:`kubernetes-all-in-one` doc.

Installing Kolla and Kolla-Kubernetes
=====================================

Operators should follow the instructions for a **full install**.  Developers
should follow the instructions for a **development install**. A development
install allows the ability to ```git pull``` within the repository in order to
use the latest code without having to re-install.  It also removes the need to
copy files to system directories such as /etc/kolla, and allows the use of
```git diff``` to see all code or resource file changes.

Full Installation
-----------------

::

    # Clone Kolla-Ansible
    git clone https://git.openstack.org/openstack/kolla-ansible
    mv kolla-ansible kolla

    # Install Kolla-Ansible
    pushd kolla
    sudo pip install .
    sudo cp -r ./etc/kolla /etc
    popd

    # Clone Kolla-Kubernetes
    git clone https://git.openstack.org/openstack/kolla-kubernetes

    # Install Kolla-Kubernetes
    pushd kolla-kubernetes
    sudo pip install .
    sudo cp -r ./etc/kolla-kubernetes /etc
    popd

Development Installation
------------------------

::

    # Clone Kolla-Ansible
    git clone https://git.openstack.org/openstack/kolla-ansible
    mv kolla-ansible kolla

    # Install Kolla-Ansible
    pushd kolla
    sudo pip install --editable .
    sudo ln -sf `readlink -f ./etc/kolla` /etc/  # link from hard-coded kolla-ansible path
    popd

    # Clone Kolla-Kubernetes
    git clone https://git.openstack.org/openstack/kolla-kubernetes

    # Install Kolla-Kubernetes
    pushd kolla-kubernetes
    sudo pip install --editable .
    sudo ln -sf `readlink -f ./etc/kolla-kubernetes` /etc/
    popd

.. NOTE::
  - Ansible commands (e.g. kolla-ansible) targeting the local machine require
    sudo because ansible creates ``/etc/.ansible_*`` and
    ``/etc/kolla/<service>`` files which require root permissions.
  - Executing local versions of kolla tools ``./tools/kolla-ansible`` instead
    of from the system path, will locate resource files from relative locations
    instead of system locations.
  - The development install will also work with Python virtual environments.

Generate Config Files
---------------------

Kolla-kubernetes relies on configuration files generated by kolla-ansible,
and it relies on images generated by the kolla deliverable. When fully
installed, kolla-ansible default configuration files globals.yml and
passwords.yml are expected in ``/etc/kolla``. Newly generated configuration
files are placed in the same directory. Kolla-ansible's ``generate_passwords.py``
generates passwords and populates the already existing passwords.yml file
which contains passwords and encryption keys.

Kolla-ansible's ``kolla-ansible genconfig`` will generate the
config files for each kolla service container based on the contents of
globals.yml and passwords.yml.

First, edit ``/etc/kolla/globals.yml`` and add the following::

  # Kolla-kubernetes custom configuration
  orchestration_engine: "KUBERNETES"
  api_interface_address: "0.0.0.0"
  memcached_servers: "memcached"
  keystone_database_address: "mariadb"
  keystone_admin_url: "http://keystone-admin:35357/v3"
  keystone_internal_url: "http://keystone-public:5000/v3"
  keystone_public_url: "http://keystone-public:5000/v3"
  glance_registry_host: "glance"

Then, generate the config files for all the services::

  cd kolla-ansible
  ./tools/kolla-ansible genconfig

Building Kolla Containers
=========================

Kolla-kubernetes uses Kolla's containers.

The Kolla documentation engine has a detailed `overview of building the
containers <http://docs.openstack.org/developer/kolla/image-building.html>`_.

Build Kolla's containers locally::

    kolla-build mariadb glance neutron nova openvswitch memcached \
                kolla-toolbox keystone horizon
