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
IP address `10.99.99.4`, 32G memory limit, 256G disk quota, and 32 CPU cores.
Note that running `poudriere` in a jail requires many custom jail options, which
are also set with this command.

    alcatraz1# jailctl create \
      -v 199 \
      -a 10.99.99.4 \
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

    ./boxconf -e idm_bootstrap=true -s 10.99.99.4 pkg1

Substitute whatever IP you chose for the `pkg1` jail as necessary. Note that it
will take a while to build all the packages for the first time.


## Step 3: The IDM Servers

Next, we'll build the IDM jails. While you technically only need one, you should
build at least two so that you can reboot one of them without causing a DNS
outage for your entire environment.


### Create the Jails

Let's two jails named `idm1` and `idm2`. Note that `boxconf` assumes any host
named `idm[0-9]` has the `idm_server` hostclass.

    alcatraz1# jailctl create \
      -v 199 \
      -a 10.99.99.2 \
      -k ~/id_ed25519.pub \
      -c 2-3 \
      -m 4g \
      -q 32G \
      idm1 freebsd14

    alcatraz1# jailctl create \
      -v 199 \
      -a 10.99.99.3 \
      -k ~/id_ed25519.pub \
      -c 4-5 \
      -m 4g \
      -q 32G \
      idm2 freebsd14


## Set Boxconf Variables

Before continuing, you'll need to tailor `site/vars/common` to your
environment:

  - The `domain` variable must contain your internal domain name. Eg:

        domain=idm.example.com

  - The `pkg_host_ip` variable must contain the IP address of the `pkg1` jail:

        pkg_host_ip=10.99.99.4

  - The `idm_server_list` variable must contain a newline-separated list of
    IDM server hostnames, along with their associated LDAP server ID and IP address.
    These should be the IPs of the jails you just created.

    The server ID can be any number 1-9, as long as it is unique for each host
    and you never, ever change it. Eg:

        idm_server_list="\
        idm1  1  10.99.99.2
        idm2  2  10.99.99.3"

  - The `reverse_dns_zones` variable must contain a space-separated list of
    all the reverse DNS zones in your environment. Eg:

        reverse_dns_zones="\
        99.99.10.in-addr.arpa
        88.99.10.in-addr.arpa"

    Note: only 3-octet IPv4 zones are supported (`10.in-addr.arpa` won't work).


## Create TLS Certificates

We will also need some TLS certificates for the LDAP servers. These certificates
allow for secure replication between the LDAP daemons on the IDM servers.

First, initialize the PKI. This will create a root certificate authority with
a name contraint for hostnames underneath your internal domain.

*However,* the LDAP servers replicate using their IP addresses, rather than DNS
names. Therefore, you will need to specify additional constraints for the IP
address of each IDM server. Eg:

    ./pki init \
      -c IP:10.99.0.0/255.255.0.0 \
      idm.example.com

Next, create server certificates for each IDM server. Each certificate will
need three SANs:

  - The FQDN of the IDM server.
  - The IP of the IDM server.
  - The bare domain name (we'll make this a multi-valued A record later).

Eg:

    ./pki cert -d 3650 idm1 slapd idm1.idm.example.com IP:10.99.99.2 idm.example.com
    ./pki cert -d 3650 idm2 slapd idm2.idm.example.com IP:10.99.99.3 idm.example.com

Finally, create a client certificate for the OpenLDAP replicator DN. Eg:

    ./pki client-cert -d 3650 idm1 replicator cn=replicator,dc=idm,dc=example,dc=com
    ./pki client-cert -d 3650 idm2 replicator cn=replicator,dc=idm,dc=example,dc=com


## Configure the IDM servers.

Now, you're ready to build the IDM servers!

The first server in the `$idm_server_list` is somewhat special, as the
`boxconf` scripts will use that one to create all the initial LDAP objects.
So make sure you configure that one first.

    ./boxconf -s 10.99.99.2 -e idm_bootstrap=true idm1
    ./boxconf -s 10.99.99.3 -e idm_bootstrap=true idm2


## Verify LDAP replication, DNS, etc.

If everything is working, you should get the same result from each of the
following `dig` queries:

    $ dig +short @10.99.99.2 idm.example.com
    10.99.99.3
    10.99.99.2

    $ dig +short @10.99.99.3 idm.example.com
    10.99.99.3
    10.99.99.2
