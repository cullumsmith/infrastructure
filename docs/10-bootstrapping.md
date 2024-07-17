Bootstrapping the Environment
=============================

Most hosts that are built with `boxconf` depend on at least two other hosts
being available:

  1. The IDM (identity management) server. This server provides DNS, Kerberos
     authentication, and LDAP user and group lookups.

  2. The package repository. Almost all hosts are FreeBSD, and they depend on
     the local `poudriere` server which hosts in-house packages built with
     custom options.

The IDM servers and the package repo are themselves built with boxconf, but you
must build them in a specific order to solve the chicken-and-egg problem.


## Step 1: The Hypervisor

It is assumed that most hosts will be FreeBSD jails. Therefore, you will need
a FreeBSD "hypervisor" along with our custom `jailctl` script.

Boxconf can be used to configure a FreeBSD hypervisor for running jails and
bhyve VMs. The only requirement for this server is a NIC that supports VLAN
tagging. By default, this interface is assumed to be `lagg0`.

Boxconf assumes any host named `alcatraz[0-9]` has the `freebsd_hypervisor`
hostclass. Therefore, you can run the following to setup the jail host:

    ./boxconf -s $FREEBSD_HYPERVISOR_IP alcatraz1

Then, on `alcatraz1`, download a FreeBSD rootfs image used for templating jails:

    alcatraz1# jailctl download-release 14.1-RELEASE


## Step 2: The Pkg Repository

First, we'll need a jail to serve as our Poudriere server. This jail will build
all the necessary packages and serve them over HTTP.

On the FreeBSD hypervisor, use `jailctl` to create a jail for the `pkg` repo.
The following command will create a jail named `pkg1` with VLAN tag `199`,
IP address `10.11.199.4`, 32G memory limit, 256G disk quota, and 32 CPU cores.
Note that running `poudriere` in a jail requires many custom jail options, which
are also set with this command.

    alcatraz1# jailctl create \
      -v 199 \
      -a 10.11.199.4 \
      -k ~/id_ed25519.pub \
      -c 64-95 \
      -m 32g \
      -q 256g \
      -e mount.procfs=true \
      -e allow.mount.tmpfs=true \
      -e allow.mount.devfs=true \
      -e allow.mount.procfs=true \
      -e allow.mount.nullfs=true \
      -e allow.mount.linprocfs=true \
      -e allow.raw_sockets=true \
      -e allow.socket_af=true \
      -e allow.mlock=true \
      -e sysvmsg=new \
      -e sysvsem=new \
      -e sysvshm=new \
      -e children.max=1000 \
      pkg1 freebsd14.1

Now you are ready to build all the packages and create the repository. `boxconf`
assumes that any host named `pkg[0-1]` has the `pkg_repository` hostclass.

    ./boxconf -e idm_bootstrap=true 10.11.199.4

Substitute whatever IP you chose for the `pkg1` jail as necessary. Note that it
will take a while to build all the packages for the first time.
