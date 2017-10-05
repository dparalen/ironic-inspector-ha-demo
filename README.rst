**ironic inspector** HA demo
============================

This repository contains configuration files and instructions about how to set
up and perform a single-node **devstack** deployment, running 2 instances of
**ironic-inspector** with concurrent client requests being distributed
round-robin to the instances thru the **HAProxy** service.

The goal is to show the reader the ability of inspector working in an
*active-active* deployment. You may wish to check the video_ of this demo; the
fun starts at 10min.

.. _video: https://www.youtube.com/watch?v=I5wLmBfbhds

VM setup
--------

This demo was tested in **VirtualBox** on an **OS X** laptop. The base system
used here is a **Fedora 26 Server** with 20G disk room, a single CPU core and 4G
of RAM (please, consider using 8G or have a big enough swap though). The network
setup is the default that **VirtualBox** gives, which is a NAT. As far as the OS
setup goes, make sure you install **git** and enable remote management over SSH;
the rest should be pulled as dependencies during the `devstack installation`_. I
also set up password-less ``sudo`` for my user.

**devstack** installation
-------------------------

Clone the **devstack** project repository and use the `localrc`_ configuration
file::

  $ git clone git://git.openstack.org/openstack-dev/devstack
  $ pushd devstack
  $ curl -O https://raw.githubusercontent.com/dparalen/ironic-inspector-ha-demo/master/localrc
  $ ./stack.sh
  $ popd

.. note:: You may need to workaround some issues with **devstack** installation;
  it's almost always always the case ;)

.. _localrc: https://raw.githubusercontent.com/dparalen/ironic-inspector-ha-demo/master/localrc

Patching **inspector**
----------------------

The base for this demo is the **dnsmasq** PXE filter driver chain as it allows
for multiple **inspector** instances to run on a single node (mind replacing
``you@review...`` with your **Gerrit** user name)::

   $ pushd /opt/stack/ironic-inspector
   $ git checkout -b ha_inspector
   $ # the dnsmasq PXE filter patch
   $ git fetch ssh://you@review.openstack.org:29418/openstack/ironic-inspector refs/changes/48/466448/63 && git cherry-pick FETCH_HEAD
   $ # the patch to allow for concurrent updates of dnsmasq config
   # git fetch ssh://you@review.openstack.org:29418/openstack/ironic-inspector refs/changes/38/504438/8 && git cherry-pick FETCH_HEAD


There is a crucial patch that ensures state inconsistencies aren't observed in
the introspection status that didn't land yet::

  $ git fetch ssh://you@review.openstack.org:29418/openstack/ironic-inspector refs/changes/28/510928/6 && git cherry-pick FETCH_HEAD

.. note::
  Please, make sure the ``git log`` shows these patches

Now we suppose to install the patched sources of **inspector**; I use these
commands::

  $ sudo pip install -e .
  $ sudo python setup.py install
  $ popd

Environment adjustment
----------------------

So far these instructions could have been followed online. Now we're going to
adjust the system configuration so it's better to clone this repo::

  $ git clone https://github.com/dparalen/ironic-inspector-ha-demo.git
  $ pushd ironic-inspector-ha-demo
  $ install etc/ironic-inspector/dnsmasq.conf /etc/ironic-inspector/
  $ install etc/ironic-inspector/rootwrap.d/ironic-inspector-dnsmasq-systemctl.filters /etc/ironic-inspector/rootwrap.d/
  $ mkdir /etc/ironic-inspector/dhcp-hostsdir
  $ sudo install etc/haproxy/haproxy.cfg /etc/haproxy/

The **devstack inspector** service needs stopping now and the **HAProxy**
service needs a configuration update::

  $ sudo systemctl stop devstack@ironic-inspector
  $ sudo systemctl restart haproxy
  $ popd

At this point one can continue with the `HA Demo`_ itself.

HA Demo
-------

The screenplay is as follows:

* 2 instances of **inspector** are being run simultaneously (listening on
  different ports)

* the **HAProxy** is used as a (harsh) load balancer that proxies the user
  requests to the **inspector** ports

* there are 3 user sessions running:

  * periodic introspection status poll

  * periodic introspection start request

  * periodic introspection abort request

* the two **inspector** control the (shared) **dnsmasq** service

The goal is to make sure state discrepancies or odd stack traces don't show up
in **inspector** logs and the node introspection *status is consistent* all the
time. Let's run the **inspector** instances, each in a *separate* terminal
instance::

  $ /usr/bin/ironic-inspector --config-file ~/ironic-inspector-ha-demo/etc/inspector_5051.conf
  $ /usr/bin/ironic-inspector --config-file ~/ironic-inspector-ha-demo/etc/inspector_5052.conf

Now we'd better make sure an introspection is actually possible for a node in
this setup, again, in a separate terminal::

  $ source ~/devstack/openrc admin demo
  $ ironic node-set-provision-state node-0 manage
  $ openstack baremetal introspection start node-0
  $ while true; do date; openstack baremetal introspection status node-0 ; sleep 1 ; done

The introspection having finished, let's move to the racing sessions, in yet
another separate terminal instances I'm firing up the ``abort`` request::

  $ source devstack/openrc admin demo
  $ while true; do date ; openstack baremetal introspection abort node-0; sleep $(( 10 + (RANDOM % 3) )); done
 
The ``start`` request::

  $ source ~/devstack/openrc admin demo
  $ while true; do date ; openstack baremetal introspection abort node-0; start $(( 10 + (RANDOM % 3) )); done

The reader may be interested in playing with the setup in different ways. I for
instance like to ``CTRL+Z`` one of the **inspector** instances to simulate
partitioning. Another test case may be to adjust the **HAProxy** settings or to
involve more clients or to involve the other node.
