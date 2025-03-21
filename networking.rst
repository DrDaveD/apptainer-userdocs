.. _networking:

######################
Network virtualization
######################

.. _sec:networking:

{Project} provides full integration with `CNI
<https://github.com/containernetworking/cni>`_ , and the ability to join an
existing network namespace, to make network virtualization easy. 
The following options can be used with the the
action commands (``exec``, ``run``, and ``shell``) to create and
configure a virtualized network for a container.

.. note::

   Many of these options are available only to the ``root`` user by
   default. Unrestricted ability to configure networking for containers
   would allow users to affect networking on the host, or in a cluster.

   The administrator of a setuid-root installation of {Project}
   is able to permit a list of
   unprivileged users and/or groups to use specified network
   configurations. This is accomplished through settings in
   ``{command}.conf`` and only work in setuid mode.
   See the `administrator guide
   <{admindocs}/configfiles.html#networking-options>`_ for details.

*********
``--dns``
*********

The ``--dns`` option allows you to specify a comma separated list of DNS
servers to add to the ``/etc/resolv.conf`` file.

.. code::

   $ nslookup google.com | grep Server
   Server:             127.0.0.53

   $ sudo {command} exec --dns 8.8.8.8 ubuntu.sif nslookup google.com | grep Server
   Server:             8.8.8.8

   $ sudo {command} exec --dns 8.8.8.8 ubuntu.sif cat /etc/resolv.conf
   nameserver 8.8.8.8

**************
``--hostname``
**************

The ``--hostname`` option accepts a string argument to change the
hostname within the container.

.. code::

   $ hostname
   ubuntu

   $ sudo {command} exec --hostname hal-9000 my_container.sif hostname
   hal-9000

****************
``--netns-path``
****************

The ``--netns-path`` flag takes a path to a network namespace to join when
starting a container. The root user may join any network namespace. An
unprivileged user can only join a network namespace specified in the new
allowed ``netns paths directive`` in ``{command}.conf``, if they are also
listed in ``allow net users`` / ``allow net groups``. Not currently supported
with ``--fakeroot`` mode.

For example, a network namespace can be created with the ``ip`` command on the
host, and then a container started that will run within this namespace:

.. code::

   # Create an example named network namespace
   $ sudo ip netns add my-net

   # Add a dummy network interface to the network namespace
   $ sudo ip netns exec my-net ip link add dummy0 type dummy

   # Run a container in the network namespace
   $ sudo {command} run --netns-path /run/netns/my-net library://alpine
   INFO:    Using cached image
   Apptainer> ip a
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN qlen 1000
      link/ether 92:5c:ab:ab:d3:d0 brd ff:ff:ff:ff:ff:ff
   Apptainer>

Note that the ``ip a`` command run inside the container shows the ``dummy0``
interface we added to the ``my-net`` network namespace.

The root user can join any network namespace with ``--netns-path``.

A non-root user can only join a network namespace if the following are true:

* {Project} is installed with setuid privileges.
* The path of the network namespace is listed in the ``allow netns paths``
  directive in ``{command}.conf``, which is typically managed by the system
  administrator.
* The user is listed in the ``allow net users`` directive in ``{command}.conf``,
  or the user is a member of a group listed in the ``allow net groups`` directive
  in ``{command}.conf``.

*********
``--net``
*********

Passing the ``--net`` flag will cause the container to join a new
network namespace when it initiates. A bridge interface is also set up by
default.

.. code::

   $ hostname -I
   10.0.2.15

   $ sudo {command} exec --net my_container.sif hostname -I
   10.22.0.4

*************
``--network``
*************

The ``--network`` option can only be invoked in combination with the
``--net`` flag. It accepts a comma delimited string of network types.
Each entry will bring up a dedicated interface inside container.

.. code::

   $ hostname -I
   172.16.107.251 10.22.0.1

   $ sudo {command} exec --net --network ptp ubuntu.sif hostname -I
   10.23.0.6

   $ sudo {command} exec --net --network bridge,ptp ubuntu.sif hostname -I
   10.22.0.14 10.23.0.7

When invoked, the ``--network`` option searches the {command}
configuration directory (commonly
``/usr/local/etc/{command}/network/``) for the cni configuration file
corresponding to the requested network type(s). Several configuration
files are installed with {Project} by default corresponding to the
following network types:

   -  bridge
   -  ptp
   -  ipvlan
   -  macvlan
   -  none (must be used alone)

By default, ``none`` is the only network configuration that can be used
by non-privileged users. It isolates the container network from the host
network with a loopback interface.

Administrators can permit certain users or groups to request other
network configurations through options in ``{command}.conf``.
Additional cni configuration files can be added to the ``network``
configuration directory as required, and {Project}'s provided
configurations may also be modified.

******************
``--network-args``
******************

The ``--network-args`` option provides a convenient way to specify
arguments to pass directly to the cni plugins. It must be used in
conjunction with the ``--net`` flag.

For instance, let's say you want to start an `NGINX
<https://www.nginx.com/>`_ server on port 80 inside of the container,
but you want to map it to port 8080 outside of the container:

.. code::

   $ sudo {command} instance start --writable-tmpfs \
       --net --network-args "portmap=8080:80/tcp" docker://nginx web2

The above command will start the Docker Hub official NGINX image running
in a background instance called ``web2``. The NGINX instance will need
to be able to write to disk, so we've used the ``--writable-tmpfs``
argument to allocate some space in memory. The ``--net`` flag is
necessary when using the ``--network-args`` option, and specifying the
``portmap=8080:80/tcp`` argument which will map port 80 inside of the
container to 8080 on the host.

Now we can start NGINX inside of the container:

.. code::

   $ sudo {command} exec instance://web2 nginx

And the ``curl`` command can be used to verify that NGINX is running on
the host port 8080 as expected.

.. code::

   $ curl localhost:8080
   10.22.0.1 - - [16/Oct/2018:09:34:25 -0400] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0" "-"
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
   </head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>

   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>

   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>

For more information about cni, check the `cni specification
<https://github.com/containernetworking/cni/blob/master/SPEC.md>`_.
