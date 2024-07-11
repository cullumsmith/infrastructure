# infrastructure

A shell-based configuration management framework for unix-like systems.

## boxconf

[boxconf](./boxconf) is an extremely simple config management system written in shell. As
long as you can SSH to the remote system (or "box") as root, the only requirement
is a POSIX-compliant sh(1) and coreutils.  It was inspired by many years of
frustration with Ansible.

### Running boxconf

To execute boxconf on a target host, just run the following:

    ./boxconf $TARGET_HOSTNAME

A deployment tarball will be generated and SCP'd to the remote box, where `boxconf`
will re-exec itself. After gathering some information about the target system (such
as the operating system, IP address, etc), `boxconf` will source your scripts in the following order:

         vars/common
    site/vars/common
         vars/os/${os}
    site/vars/os/${os}
         vars/distro/${distro}
    site/vars/distro/${distro}
         vars/hostclass/${hostclass}
    site/vars/hostclass/${hostclass}
         vars/hostname/${hostname}
    site/vars/hostname/${hostname}

         scripts/common
    site/scripts/common
         scripts/os/${os}
    site/scripts/os/${os}
         scripts/distro/${distro}
    site/scripts/distro/${distro}
         scripts/hostclass/${hostclass}
    site/scripts/hostclass/${hostclass}
         scripts/hostname/${hostname}
    site/scripts/hostname/${hostname}

If any of those paths point to a directory, boxconf will source all files in
that directory in glob order.

The `site/` directory does not exist in this repo. Its purpose is to hold personal
site-specific variables and scripts that you would rather not share in a public git repo.
Ideally, you would use git submodules for this.

The `hostname` value is taken from the short hostname of the remote system.
If the remote hostname is incorrect (or unset), you can override the hostname
detection by passing the `-o $HOSTNAME` flag to boxconf.

The `hostclass` value is matched based on the regular expressions listed in
the [hostclasses](./hostclasses) file.

### Encrypting source files

`boxconf` supports encrypting any script or file using OpenSSL's [pbkdf2](https://www.openssl.org/docs/man3.0/man1/openssl-enc.html).
The encrypted file will be automatically decrypted when generating the deployment tarball.
The encryption password is read from the `BOXCONF_VAULT_PASSWORD` environment
variable or the `.vault_password` file. If nether is set, you will be prompted for
the password interactively.

The [vault](./vault) script in the root of this directory can be used to manage
encrypted files.

### Copying files to the remote host

From your `boxconf` scripts, you can copy files in the `files/` (or `site/files/`)
directory to the target system using the `install_file` function. The source file
should have the same path as the remote path, and it can be tailored to the remote
system by adding a custom suffix. For example, if you ran the following code:

    install_file -m 0644 /etc/passwd

Then the following paths would be searched to find a suitable file to copy into
the target system (the first match wins):

    site/files/etc/passwd.${hostname}
         files/etc/passwd.${hostname}
    site/files/etc/passwd.${hostclass}.${distro}
         files/etc/passwd.${hostclass}.${distro}
    site/files/etc/passwd.${distro}.${hostclass}
         files/etc/passwd.${distro}.${hostclass}
    site/files/etc/passwd.${hostclass}.${os}
         files/etc/passwd.${hostclass}.${os}
    site/files/etc/passwd.${os}.${hostclass}
         files/etc/passwd.${os}.${hostclass}
    site/files/etc/passwd.${hostclass}
         files/etc/passwd.${hostclass}
    site/files/etc/passwd.${distro}
         files/etc/passwd.${distro}
    site/files/etc/passwd.${os}
         files/etc/passwd.${os}
    site/files/etc/passwd.common
         files/etc/passwd.common

If you use the `install_template` function, then the same file matching logic
applies. However, the content of the matched file will be treated like a
heredoc, allowing you to do things like interpolate `${shell_variables}` and perform
`$(process_substitution)` within the file content. Note that if you do this, you
must esacape any shell characters (like `$`) as needed.

### Copying TLS certificates

The `install_certificate` and `install_certificate_key` functions can be used
to copy certificates from the `site/ca` directory to the remote host. The certificates
should be created and managed using the included [pki](./pki) script.

Note that certificate keys are also encrypted with `$BOXCONF_VAULT_PASSWORD`. They
are automatically decrypted when generating the configuration tarball.


## vault

The [vault](./vault) script is used to manage encrypted files using OpenSSL's [pbkdf2](https://www.openssl.org/docs/man3.0/man1/openssl-enc.html).
The encryption password is read from the `BOXCONF_VAULT_PASSWORD` environment variable
or the `.vault_password` file.

### Create a new encrypted file

The following command will invoke `$EDITOR` to create a new encrypted file at the
specified path.

    ./vault create passwords.txt

### Decrypt file(s)

The plaintext content of the file(s) will be written to stdout.

    ./vault decrypt secrets.txt

### Edit an encrypted file

The file will be decrypted to a temporary file before being opened with `$EDITOR`.
When the editor is closed, the file is encrypted again.

    ./vault edit passwords.txt

### Encrypt an existing file

Encrypt an existing file in place:

    ./vault encrypt plain.txt

### Re-encrypt file(s) with a different password

The new password is read from the `VAULT_NEW_PASSWORD` environment variable.
If this variable is unset, you will be prompted interactively.

    ./vault reencrypt secrets.txt


## pki

The [pki](./pki) script is used to manage an internal certificate authority using OpenSSL.

Certificates and private keys are stored in the 'site/ca' directory with human-readable
names. The certificatess are mapped to their OpenSSL serial number via symlinks.

The private keys are encrypted with the `BOXCONF_VAULT_PASSWORD` variable, as
described previously. The private key of the CA itself is acquired from the `CA_PASSWORD`
environment variable, or the `.ca_password` file.

Every certificate is associated with a single `boxconf` hostname, along with a unique certificate
name. This allows you to store multiple certificates per host.

### Initialize the CA

`pki init` will create the CA certificate and private key, along with an OpenSSL
configuration file. [Name constraints](https://www.openssl.org/docs/man3.0/man5/x509v3_config.html)
for the CA can be added with the `-c` option.

For example, this command creates a CA for the `example.com` domain. This CA can sign
certificates for all subdomains of `example.com` and `example.net`, as well as plain IP
addresses in the 192.168.0.0/24 subnet:

    ./pki init -c 192.168.0.0/255.255.255.0 -c example.net example.com

### Create a server certificate

`pki cert` creates a **server** certificate keypair signed by the CA.

For example, this command creates a certificate pair for `nginx` for the host
`webserver1` with a 365 day expiration (`-d`). After the hostname and certificate
name, each additional argument is added to the Subject Alternative Names field.

The Common Name is taken from the first specified SAN. If you don't specify a type
for the SAN, `DNS` is assumed.

    ./pki cert -d 365 webserver1 nginx www.example.com DNS:example.com IP:192.168.0.5

### Create a client certificate

`pki client-cert` creates a **client** certificate keypair signed by the CA.

After the hostname and certificate name, the first argument must be an LDAP-style DN
for the certificate's Common Name value. SANs can be specified using additional arguments
in same way as described previously.

    ./pki client-cert -d 3650 ldap1 replicator cn=replicator,dc=idm,dc=example,dc=com

### Renew a certificate

`pki renew` is used to renew an existing certificate.

    ./pki renew -d 365 webserver1 nginx
