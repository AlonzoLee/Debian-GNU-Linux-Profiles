#### Keyword: Ciper suites, Hardened, GnuTLS, Public Authentication

#### Scenario

Using Cisco Anyconnect to tunnel all traffic. Ocserv is an Anyconnect compatible server. Anyconnect is widely used in company and university.
This *protocol* is too special to forbidden :). So it's necessary and very useful in remote access. It build with GnuTLS, so we can custom our cipher suite.
For security purpose, we use `Public key` for authentication(at least).

#### Download sourcecode
```
 wget ftp://ftp.infradead.org/pub/ocserv/ocserv-1.0.1.tar.xz
 wget ftp://ftp.infradead.org/pub/ocserv/ocserv-1.0.1.tar.xz.sig
```
#### Verify sourcecode

```
$ sudo apt install gnupg  
$ gpg  --keyserver hkps://keyserver.ubuntu.com --search 0x29EE58B996865171   
    
gpg: data source: https://162.213.33.9:443
(1)	Nikos Mavrogiannopoulos <nmav@gnutls.org>
	Nikos Mavrogiannopoulos <nmav@hushmail.com>
	Nikos Mavrogiannopoulos <n.mavrogiannopoulos@gmail.com>
	  3104 bit RSA key 29EE58B996865171, created: 2008-05-04
Keys 1-1 of 1 for "0x29EE58B996865171".  Enter number(s), N)ext, or Q)uit > 1
gpg: key 29EE58B996865171: public key "Nikos Mavrogiannopoulos <n.mavrogiannopoulos@gmail.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1

$ gpg --verify ocserv-1.0.1.tar.xz.sig   

gpg: assuming signed data in 'ocserv-1.0.1.tar.xz'
gpg: Signature made Friday, April 10, 2020 AM05:10:16 HKT
gpg:                using RSA key 1F42418905D8206AA754CCDC29EE58B996865171
gpg: Good signature from "Nikos Mavrogiannopoulos <n.mavrogiannopoulos@gmail.com>" [unknown]
gpg:                 aka "Nikos Mavrogiannopoulos <nmav@gnutls.org>" [unknown]
gpg:                 aka "Nikos Mavrogiannopoulos <nmav@hushmail.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 1F42 4189 05D8 206A A754  CCDC 29EE 58B9 9686 5171
```

#### Build dependencies[1]

```
sudo apt-get update
sudo apt install libgnutls28-dev libev-dev build-essential autogen pkg-config libprotobuf-c-dev
```

check libgnutls version, if you using `AES-GCM`, GnuTLS should be 3.2.7 or later
```
sudo apt search gnutls | grep -i installed
```
Optional dependencies that enable specific functionality

```
sudo apt install libseccomp-dev  #for seccomp
sudo apt install libreadline-dev  #for occtl
sudo apt install libsystemd-dev  #for systemd
sudo apt install libwrap0-dev libpam0g-dev liblz4-dev libnl-route-3-dev libradcli-dev libkrb5-dev
```

#### Extract files

```
tar xvf ocserv-1.0.1.tar.xz
```

#### Configure
```
cd ocserv-1.0.1
./configure
make
make check
sudo make install
sudo mkdir /usr/local/share/ocserv/
sudo mkdir /etc/ocserv
sudo cp -r doc/ /usr/local/share/ocserv/
sudo cp /usr/local/share/ocserv/doc/sample.config /etc/ocserv/ocserv.conf
ln -s /usr/local/bin/ocserv-fw /usr/bin/ocserv-fw
```
Add daemon user

```
sudo useradd -r -s /bin/false ocservdaemon
```

#### Generating needed key and certificate

Generate Self-sign Root Certificate Authority


gen-root-ca.sh

```
#!/bin/bash   

gen_template()
{
cat <<EOF> root-ca.tmpl
cn = "TYA Company Limited Root Certificate Authority"
organization = "TYA Company Limited."
unit = "TYA Infrastructure Assurance"
country = "CN"
expiration_days = "720"
ca
signing_key
cert_signing_key
crl_signing_key
EOF
}

gen_ecc_key()
{   
certtool --generate-privkey --ecc --sec-param high -d 2 > root-ca-ecdsa-key.pem   
}

gen_ecc_cert()   
{
certtool --generate-self-signed \
--load-privkey root-ca-ecdsa-key.pem \
--template root-ca.tmpl \
--outfile root-ca-ecdsa-cert.pem \
--hash=SHA384
}
   
gen_template
gen_ecc_key
gen_ecc_cert
EOF
```
gen-client-ecdsa-key-and-cert.sh

```
    #!/bin/bash
    gen_ecc_key()
    {
    certtool --generate-privkey --ecc --sec-param high > vpn-client-ecdsa-key.pem
    }

    gen_template()
    {
    cat <<EOF> client-cert.tmpl
    country = CN
    organization = "TYA Company Limited."
    unit = "CISRT"                            
    cn = "username"
    tls_www_client
    signing_key
    EOF
    }
    #unit option: You can take it as your division name
    #cn option: You can take it as your username

    gen_client_cert()
    {
    certtool --generate-certificate \
             --load-ca-privkey root-ca-ecdsa-key.pem \
	     --load-ca-certificate root-ca-ecdsa-cert.pem \
	     --load-privkey vpn-client-ecdsa-key.pem \
	     --template client-cert.tmpl \
	     --outfile vpn-client-ecdsa-cert.pem \
	     --hash=SHA384
    }

    gen_ecc_key
    gen_template
    gen_client_cert
```

gen-client-rsa-key-and-cert.sh

```
    #!/bin/bash
    gen_rsa_key()
    {
    certtool --generate-privkey --rsa --sec-param high > vpn-client-rsa-key.pem
    }

    gen_template()
    {
    cat <<EOF> client-cert.tmpl
    country = CN
    organization = "TYA Company Limited."
    unit = "CISRT"                            
    cn = "username"
    tls_www_client
    signing_key
    EOF
    }
    #unit option: You can take it as your division name
    #cn option: You can take it as your username

    gen_client_cert()
    {
    certtool --generate-certificate \
             --load-ca-privkey root-ca-ecdsa-key.pem \
	     --load-ca-certificate root-ca-ecdsa-cert.pem \
	     --load-privkey vpn-client-rsa-key.pem \
	     --template client-cert.tmpl \
	     --outfile vpn-client-rsa-cert.pem \
	     --hash=SHA384
    }

    gen_rsa_key
    gen_template
    gen_client_cert
```

#### Generate p12 format certificate

for ecdsa key
```
    certtool --to-p12 --load-privkey vpn-client-ecdsa-key.pem --pkcs-cipher 3des-pkcs12 --load-certificate vpn-client-ecdsa-cert.pem --outfile vpn-client-ecdsa.p12 --outder
```
for rsa key
```
    certtool --to-p12 --load-privkey vpn-client-rsa-key.pem --pkcs-cipher 3des-pkcs12 --load-certificate vpn-client-rsa-cert.pem --outfile vpn-client-rsa.p12 --outder
```
#### Server certificate

[Let's encrypt](https://letsencrypt.org) is recommended
You can get a certificate by using acme.sh

Install acme.sh

```
sudo apt install curl
curl https://get.acme.sh | sh
```

Request certificate

```
acme.sh --issue --alpn -d example.com
```


It will using your server's 443 port to verify it.

#### Cipher Suite select(for RSA keys)
```
 gnutls-cli --priority SECURE256:%PROFILE_MEDIUM:%SERVER_PRECEDENCE:-ECDHE-ECDSA:-RSA:-DHE-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2 -l
    Cipher suites for SECURE256:%PROFILE_MEDIUM:%SERVER_PRECEDENCE:-ECDHE-ECDSA:-RSA:-DHE-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2
    TLS_ECDHE_RSA_AES_256_GCM_SHA384                  	0xc0, 0x30	TLS1.2
    TLS_ECDHE_RSA_CAMELLIA_256_GCM_SHA384             	0xc0, 0x8b	TLS1.2

    Certificate types: CTYPE-X.509
    Protocols: VERS-TLS1.2
    Compression: COMP-NULL
    Elliptic curves: CURVE-SECP384R1, CURVE-SECP521R1
    PK-signatures: SIGN-RSA-SHA384, SIGN-ECDSA-SHA384, SIGN-RSA-SHA512, SIGN-ECDSA-SHA512
```
We choose priority strings `SECURE256`. We can use `gnutls-cli --priority SECURE256 -l` to see all cipher suites for SECURE256
```
 gnutls-cli --priority SECURE256 -l
    Cipher suites for SECURE256
    TLS_ECDHE_ECDSA_AES_256_GCM_SHA384                	0xc0, 0x2c	TLS1.2
    TLS_ECDHE_ECDSA_CAMELLIA_256_GCM_SHA384           	0xc0, 0x87	TLS1.2
    TLS_ECDHE_ECDSA_AES_256_CBC_SHA384                	0xc0, 0x24	TLS1.2
    TLS_ECDHE_ECDSA_CAMELLIA_256_CBC_SHA384           	0xc0, 0x73	TLS1.2
    TLS_ECDHE_ECDSA_AES_256_CCM                       	0xc0, 0xad	TLS1.2
    TLS_ECDHE_RSA_AES_256_GCM_SHA384                  	0xc0, 0x30	TLS1.2
    TLS_ECDHE_RSA_CAMELLIA_256_GCM_SHA384             	0xc0, 0x8b	TLS1.2
    TLS_ECDHE_RSA_AES_256_CBC_SHA384                  	0xc0, 0x28	TLS1.2
    TLS_ECDHE_RSA_CAMELLIA_256_CBC_SHA384             	0xc0, 0x77	TLS1.2
    TLS_RSA_AES_256_GCM_SHA384                        	0x00, 0x9d	TLS1.2
    TLS_RSA_CAMELLIA_256_GCM_SHA384                   	0xc0, 0x7b	TLS1.2
    TLS_RSA_AES_256_CBC_SHA256                        	0x00, 0x3d	TLS1.2
    TLS_RSA_CAMELLIA_256_CBC_SHA256                   	0x00, 0xc0	TLS1.2
    TLS_RSA_AES_256_CCM                               	0xc0, 0x9d	TLS1.2
    TLS_DHE_RSA_AES_256_GCM_SHA384                    	0x00, 0x9f	TLS1.2
    TLS_DHE_RSA_CAMELLIA_256_GCM_SHA384               	0xc0, 0x7d	TLS1.2
    TLS_DHE_RSA_AES_256_CBC_SHA256                    	0x00, 0x6b	TLS1.2
    TLS_DHE_RSA_CAMELLIA_256_CBC_SHA256               	0x00, 0xc4	TLS1.2
    TLS_DHE_RSA_AES_256_CCM                           	0xc0, 0x9f	TLS1.2

    Certificate types: CTYPE-X.509
    Protocols: VERS-TLS1.2, VERS-TLS1.1, VERS-TLS1.0, VERS-DTLS1.2, VERS-DTLS1.0
    Compression: COMP-NULL
    Elliptic curves: CURVE-SECP384R1, CURVE-SECP521R1
    PK-signatures: SIGN-RSA-SHA384, SIGN-ECDSA-SHA384, SIGN-RSA-SHA512, SIGN-ECDSA-SHA512
```
In `SECURE256` cipher suite, `compression methods` is `null` by default, so it's no problem with `CRIME SSL/TLS attack`.
Because server certificate I used is powered by [let's encrypt](https://letsencrypt.org/) it use 2048 bits RSA key, but `SECURE256`
Using `%PROFILE_HIGH`, for `%PROFILE_HIGH` it required 	RSA key with 3072 bits. So we should use `%PROFILE_MEDIUM` for 2048 bits RSA key support.
We using `%SERVER_PRECEDENCE` to using cipher suites that in server priorities not the client's. We using rsa key rather 
than ecc key so we using `-ECDHE-ECDSA` to disable ECDSA support. RSA without DHE or ECDHE can't provide perfect forward secrecy, so we disable it. DHE required more computation and ECDHE will be more efficient. So we using `TLS_ECDHE_RSA`. `CBC` does not provide authenticated encryption, but `AES GCM` do, so we using AES_GCM.
We using cipher suites provide by TLS1.2, so disable TLS1.1, and TLS1.0 sucks.we use `-VERS-TLS1.1:-VERS-TLS1.0` to disable it. We don't use any DTLS so we disable it.

So, here is our cipher suites:
```
    tls-priorities = "SECURE256:%PROFILE_MEDIUM:%SERVER_PRECEDENCE:-ECDHE-ECDSA:-DHE-RSA:-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2"

#### Cipher Suite select(for ECDSA keys)

 gnutls-cli --priority SECURE256:%PROFILE_MEDIUM:%SERVER_PRECEDENCE:-ECDHE-RSA:-RSA:-DHE-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2 -l
    Cipher suites for SECURE256:%PROFILE_MEDIUM:%SERVER_PRECEDENCE:-ECDHE-RSA:-RSA:-DHE-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2
    TLS_ECDHE_ECDSA_AES_256_GCM_SHA384                	0xc0, 0x2c	TLS1.2
    TLS_ECDHE_ECDSA_CAMELLIA_256_GCM_SHA384           	0xc0, 0x87	TLS1.2
    TLS_ECDHE_ECDSA_AES_256_CCM                       	0xc0, 0xad	TLS1.2

    Certificate types: CTYPE-X.509
    Protocols: VERS-TLS1.2
    Compression: COMP-NULL
    Elliptic curves: CURVE-SECP384R1, CURVE-SECP521R1
    PK-signatures: SIGN-RSA-SHA384, SIGN-ECDSA-SHA384, SIGN-RSA-SHA512, SIGN-ECDSA-SHA512
```
We choose priority strings `SECURE256`. We can use `gnutls-cli --priority SECURE256 -l` to see all cipher suites for SECURE256
```
 gnutls-cli --priority SECURE256 -l
    Cipher suites for SECURE256
    TLS_ECDHE_ECDSA_AES_256_GCM_SHA384                  0xc0, 0x2c      TLS1.2
    TLS_ECDHE_ECDSA_CAMELLIA_256_GCM_SHA384             0xc0, 0x87      TLS1.2
    TLS_ECDHE_ECDSA_AES_256_CBC_SHA384                  0xc0, 0x24      TLS1.2
    TLS_ECDHE_ECDSA_CAMELLIA_256_CBC_SHA384             0xc0, 0x73      TLS1.2
    TLS_ECDHE_ECDSA_AES_256_CCM                         0xc0, 0xad      TLS1.2
    TLS_ECDHE_RSA_AES_256_GCM_SHA384                    0xc0, 0x30      TLS1.2
    TLS_ECDHE_RSA_CAMELLIA_256_GCM_SHA384               0xc0, 0x8b      TLS1.2
    TLS_ECDHE_RSA_AES_256_CBC_SHA384                    0xc0, 0x28      TLS1.2
    TLS_ECDHE_RSA_CAMELLIA_256_CBC_SHA384               0xc0, 0x77      TLS1.2
    TLS_RSA_AES_256_GCM_SHA384                          0x00, 0x9d      TLS1.2
    TLS_RSA_CAMELLIA_256_GCM_SHA384                     0xc0, 0x7b      TLS1.2
    TLS_RSA_AES_256_CBC_SHA256                          0x00, 0x3d      TLS1.2
    TLS_RSA_CAMELLIA_256_CBC_SHA256                     0x00, 0xc0      TLS1.2
    TLS_RSA_AES_256_CCM                                 0xc0, 0x9d      TLS1.2
    TLS_DHE_RSA_AES_256_GCM_SHA384                      0x00, 0x9f      TLS1.2
    TLS_DHE_RSA_CAMELLIA_256_GCM_SHA384                 0xc0, 0x7d      TLS1.2
    TLS_DHE_RSA_AES_256_CBC_SHA256                      0x00, 0x6b      TLS1.2
    TLS_DHE_RSA_CAMELLIA_256_CBC_SHA256                 0x00, 0xc4      TLS1.2
    TLS_DHE_RSA_AES_256_CCM                             0xc0, 0x9f      TLS1.2

    Certificate types: CTYPE-X.509
    Protocols: VERS-TLS1.2, VERS-TLS1.1, VERS-TLS1.0, VERS-DTLS1.2, VERS-DTLS1.0
    Compression: COMP-NULL
    Elliptic curves: CURVE-SECP384R1, CURVE-SECP521R1
    PK-signatures: SIGN-RSA-SHA384, SIGN-ECDSA-SHA384, SIGN-RSA-SHA512, SIGN-ECDSA-SHA512
```
In `SECURE256` cipher suite, `compression methods` is `null` by default, so it's no problem with `CRIME SSL/TLS attack`.
Because server certificate I used is powered by [let's encrypt](https://letsencrypt.org/) it use 2048 bits RSA key, but `SECURE256`
We using `%SERVER_PRECEDENCE` to using cipher suites that in server priorities not the client's. We using ecc key rather
than rsa key so we using `-ECDHE-ECDSA` to disable RSA support. We don't use RSA related cipher suites, so we disable it. `CBC` does not provide authenticated encryption, but `AES GCM` do, so we using AES_GCM. We using cipher suites provide by TLS1.2, so disable TLS1.1, and TLS1.0 sucks.we use `-VERS-TLS1.1:-VERS-TLS1.0` to disable it. We don't use any DTLS so we disable it.

So, here is our cipher suites:
```
    tls-priorities = "SECURE256:%SERVER_PRECEDENCE:-ECDHE-RSA:-DHE-RSA:-RSA:-AES-256-CBC:-CAMELLIA-256-CBC:-VERS-TLS1.1:-VERS-TLS1.0:-VERS-DTLS1.0:-VERS-DTLS1.2"
```
#### ocserv.conf

```
### The following directives do not change with server reload.

# User authentication method. To require multiple methods to be
# used for the user to login, add multiple auth directives. The values
# in the 'auth' directive are AND composed (if multiple all must
# succeed).
# Available options: certificate, plain, pam, radius, gssapi.
# Note that authentication methods utilizing passwords cannot be
# combined (e.g., the plain, pam or radius methods).

# certificate:
#  This indicates that all connecting users must present a certificate.
#  The username and user group will be then extracted from it (see
#  cert-user-oid and cert-group-oid). The certificate to be accepted
#  it must be signed by the CA certificate as specified in 'ca-cert' and
#  it must not be listed in the CRL, as specified by the 'crl' option.
#
# pam[gid-min=1000]:
#  This enabled PAM authentication of the user. The gid-min option is used
# by auto-select-group option, in order to select the minimum valid group ID.
#
# plain[passwd=/etc/ocserv/ocpasswd,otp=/etc/ocserv/users.otp]
#  The plain option requires specifying a password file which contains
# entries of the following format.
# "username:groupname1,groupname2:encoded-password"
# One entry must be listed per line, and 'ocpasswd' should be used
# to generate password entries. The 'otp' suboption allows one to specify
# an oath password file to be used for one time passwords; the format of
# the file is described in https://github.com/archiecobbs/mod-authn-otp/wiki/UsersFile
#
# radius[config=/etc/radiusclient/radiusclient.conf,groupconfig=true,nas-identifier=name]:
#  The radius option requires specifying freeradius-client configuration
# file. If the groupconfig option is set, then config-per-user/group will be overridden,
# and all configuration will be read from radius. That also includes the
# Acct-Interim-Interval, and Session-Timeout values.
#
# See doc/README-radius.md for the supported radius configuration atributes.
#
# gssapi[keytab=/etc/key.tab,require-local-user-map=true,tgt-freshness-time=900]
#  The gssapi option allows one to use authentication methods supported by GSSAPI,
# such as Kerberos tickets with ocserv. It should be best used as an alternative
# to PAM (i.e., have pam in auth and gssapi in enable-auth), to allow users with
# tickets and without tickets to login. The default value for require-local-user-map
# is true. The 'tgt-freshness-time' if set, it would require the TGT tickets presented
# to have been issued within the provided number of seconds. That option is used to
# restrict logins even if the KDC provides long time TGT tickets.

#auth = "pam"
#auth = "pam[gid-min=1000]"
#auth = "plain[passwd=./sample.passwd,otp=./sample.otp]"
#auth = "plain[passwd=./sample.passwd]"
auth = "certificate"
#auth = "radius[config=/etc/radiusclient/radiusclient.conf,groupconfig=true]"

# Specify alternative authentication methods that are sufficient
# for authentication. That is, if set, any of the methods enabled
# will be sufficient to login, irrespective of the main 'auth' entries.
# When multiple options are present, they are OR composed (any of them
# succeeding allows login).
#enable-auth = "certificate"
#enable-auth = "gssapi"
#enable-auth = "gssapi[keytab=/etc/key.tab,require-local-user-map=true,tgt-freshness-time=900]"

# Accounting methods available:
# radius: can be combined with any authentication method, it provides
#      radius accounting to available users (see also stats-report-time).
#
# pam: can be combined with any authentication method, it provides
#      a validation of the connecting user's name using PAM. It is
#      superfluous to use this method when authentication is already
#      PAM.
#
# Only one accounting method can be specified.
#acct = "radius[config=/etc/radiusclient/radiusclient.conf]"

# Use listen-host to limit to specific IPs or to the IPs of a provided
# hostname.
#listen-host = [IP|HOSTNAME]

# Use udp-listen-host to limit udp to specific IPs or to the IPs of a provided
# hostname. if not set, listen-host will be used
#udp-listen-host = [IP|HOSTNAME]

# When the server has a dynamic DNS address (that may change),
# should set that to true to ask the client to resolve again on
# reconnects.
#listen-host-is-dyndns = true

# TCP and UDP port number
tcp-port = 443
udp-port = 443

# Accept connections using a socket file. It accepts HTTP
# connections (i.e., without SSL/TLS unlike its TCP counterpart),
# and uses it as the primary channel. That option is experimental
# and it has many known issues.
#  * It can only be combined with certificate authentication, when receiving
#    channel information through proxy protocol (see listen-proxy-proto)
#  * It cannot derive any keys needed for the DTLS session (hence no support for dtls-psk)
#  * It cannot enforce the framing of the SSL/TLS packets, and that
#    breaks assumptions held by several openconnect clients.
# This option is not recommended for use, and may be removed
# in the future.
#
#listen-clear-file = /var/run/ocserv-conn.socket

# The user the worker processes will be run as. It should be
# unique (no other services run as this user).
#run-as-user = nobody
#run-as-group = daemon

run-as-user = ocservdaemon
run-as-group = ocservdaemon

# socket file used for IPC with occtl. You only need to set that,
# if you use more than a single servers.
#occtl-socket-file = /var/run/occtl.socket

# socket file used for server IPC (worker-main), will be appended with .PID
# It must be accessible within the chroot environment (if any), so it is best
# specified relatively to the chroot directory.
socket-file = /var/run/ocserv-socket

# The default server directory. Does not require any devices present.
#chroot-dir = /var/lib/ocserv

# The key and the certificates of the server
# The key may be a file, or any URL supported by GnuTLS (e.g., 
# tpmkey:uuid=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx;storage=user
# or pkcs11:object=my-vpn-key;object-type=private)
#
# The server-cert file may contain a single certificate, or
# a sorted certificate chain.
# There may be multiple server-cert and server-key directives,
# but each key should correspond to the preceding certificate.
# The certificate files will be reloaded when changed allowing for in-place
# certificate renewal (they are checked and reloaded periodically;
# a SIGHUP signal to main server will force reload).

#server-cert = /etc/ocserv/server-cert.pem
#server-key = /etc/ocserv/server-key.pem
server-cert = ../tests/certs/server-cert.pem
server-key = ../tests/certs/server-key.pem

# Diffie-Hellman parameters. Only needed if for old (pre 3.6.0
# versions of GnuTLS for supporting DHE ciphersuites.
# Can be generated using:
# certtool --generate-dh-params --outfile /etc/ocserv/dh.pem
#dh-params = /etc/ocserv/dh.pem

# In case PKCS #11, TPM or encrypted keys are used the PINs should be available
# in files. The srk-pin-file is applicable to TPM keys only, and is the 
# storage root key.
#pin-file = /etc/ocserv/pin.txt
#srk-pin-file = /etc/ocserv/srkpin.txt

# The password or PIN needed to unlock the key in server-key file.
# Only needed if the file is encrypted or a PKCS #11 object. This
# is an alternative method to pin-file.
#key-pin = 1234

# The SRK PIN for TPM.
# This is an alternative method to srk-pin-file.
#srk-pin = 1234

# The Certificate Authority that will be used to verify
# client certificates (public keys) if certificate authentication
# is set.
ca-cert = /etc/ocserv/ca.pem


### All configuration options below this line are reloaded on a SIGHUP.
### The options above, will remain unchanged. Note however, that the 
### server-cert, server-key, dh-params and ca-cert options will be reloaded
### if the provided file changes, on server reload. That allows certificate
### rotation, but requires the server key to remain the same for seamless
### operation. If the server key changes on reload, there may be connection
### failures during the reloading time.


# Whether to enable seccomp/Linux namespaces worker isolation. That restricts the number of 
# system calls allowed to a worker process, in order to reduce damage from a
# bug in the worker process. It is available on Linux systems at a performance cost.
# The performance cost is roughly 2% overhead at transfer time (tested on a Linux 3.17.8).
# Note however, that process isolation is restricted to the specific libc versions
# the isolation was tested at. If you get random failures on worker processes, try
# disabling that option and report the failures you, along with system and debugging
# information at: https://gitlab.com/ocserv/ocserv/issues
isolate-workers = true

# A banner to be displayed on clients
#banner = "Welcome"

# Limit the number of clients. Unset or set to zero for unlimited.
#max-clients = 1024
max-clients = 16

# Limit the number of identical clients (i.e., users connecting 
# multiple times). Unset or set to zero for unlimited.
max-same-clients = 9

# When the server receives connections from a proxy, like haproxy
# which supports the proxy protocol, set this to obtain the correct
# client addresses. The proxy protocol would then be expected in
# the TCP or UNIX socket (not the UDP one). Although both v1
# and v2 versions of proxy protocol are supported, the v2 version
# is recommended as it is more efficient in parsing.
#listen-proxy-proto = true

# Limit the number of client connections to one every X milliseconds 
# (X is the provided value). Set to zero for no limit.
#rate-limit-ms = 100

# Stats report time. The number of seconds after which each
# worker process will report its usage statistics (number of
# bytes transferred etc). This is useful when accounting like
# radius is in use.
#stats-report-time = 360

# Stats reset time. The period of time statistics kept by main/sec-mod
# processes will be reset. These are the statistics shown by cmd
# 'occtl show stats'. For daily: 86400, weekly: 604800
# This is unrelated to stats-report-time.
server-stats-reset-time = 604800

# Keepalive in seconds
keepalive = 32400

# Dead peer detection in seconds.
# Note that when the client is behind a NAT this value
# needs to be short enough to prevent the NAT disassociating
# his UDP session from the port number. Otherwise the client
# could have his UDP connection stalled, for several minutes.
dpd = 90

# Dead peer detection for mobile clients. That needs to
# be higher to prevent such clients being awaken too 
# often by the DPD messages, and save battery.
# The mobile clients are distinguished from the header
# 'X-AnyConnect-Identifier-Platform'.
mobile-dpd = 1800

# If using DTLS, and no UDP traffic is received for this
# many seconds, attempt to send future traffic over the TCP
# connection instead, in an attempt to wake up the client
# in the case that there is a NAT and the UDP translation
# was deleted. If this is unset, do not attempt to use this
# recovery mechanism.
switch-to-tcp-timeout = 25

# MTU discovery (DPD must be enabled)
try-mtu-discovery = true

# If you have a certificate from a CA that provides an OCSP
# service you may provide a fresh OCSP status response within
# the TLS handshake. That will prevent the client from connecting
# independently on the OCSP server.
# You can update this response periodically using:
# ocsptool --ask --load-cert=your_cert --load-issuer=your_ca --outfile response
# Make sure that you replace the following file in an atomic way.
#ocsp-response = /etc/ocserv/ocsp.der

# The object identifier that will be used to read the user ID in the client 
# certificate. The object identifier should be part of the certificate's DN
# Useful OIDs are: 
#  CN = 2.5.4.3, UID = 0.9.2342.19200300.100.1.1, SAN(rfc822name)
cert-user-oid = 2.5.4.3

# The object identifier that will be used to read the user group in the 
# client certificate. The object identifier should be part of the certificate's
# DN. If the user may belong to multiple groups, then use multiple such fields
# in the certificate's DN. Useful OIDs are: 
#  OU (organizational unit) = 2.5.4.11 
#cert-group-oid = 2.5.4.11

# The revocation list of the certificates issued by the 'ca-cert' above.
# See the manual to generate an empty CRL initially. The CRL will be reloaded
# periodically when ocserv detects a change in the file. To force a reload use
# SIGHUP.
#crl = /etc/ocserv/crl.pem

# Uncomment this to enable compression negotiation (LZS, LZ4).
compression = true

# Set the minimum size under which a packet will not be compressed.
# That is to allow low-latency for VoIP packets. The default size
# is 256 bytes. Modify it if the clients typically use compression
# as well of VoIP with codecs that exceed the default value.
no-compress-limit = 256

# GnuTLS priority string; note that SSL 3.0 is disabled by default
# as there are no openconnect (and possibly anyconnect clients) using
# that protocol. The string below does not enforce perfect forward
# secrecy, in order to be compatible with legacy clients.
#
# Note that the most performant ciphersuites are the moment are the ones
# involving AES-GCM. These are very fast in x86 and x86-64 hardware, and
# in addition require no padding, thus taking full advantage of the MTU.
# For that to be taken advantage of, the openconnect client must be
# used, and the server must be compiled against GnuTLS 3.2.7 or later.
# Use "gnutls-cli --benchmark-tls-ciphers", to see the performance
# difference with AES_128_CBC_SHA1 (the default for anyconnect clients)
# in your system.

tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"

# More combinations in priority strings are available, check
# http://gnutls.org/manual/html_node/Priority-Strings.html
# E.g., the string below enforces perfect forward secrecy (PFS) 
# on the main channel.
#tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-RSA:-VERS-SSL3.0:-ARCFOUR-128"

# That option requires the established DTLS channel to use the same
# cipher as the primary TLS channel. This cannot be combined with
# listen-clear-file since the ciphersuite information is not available
# in that configuration. Note also, that this option implies that
# dtls-legacy option is false; this option cannot be enforced
# in the legacy/compat protocol.
#match-tls-dtls-ciphers = true

# The time (in seconds) that a client is allowed to stay connected prior
# to authentication
auth-timeout = 240

# The time (in seconds) that a client is allowed to stay idle (no traffic)
# before being disconnected. Unset to disable.
#idle-timeout = 1200

# The time (in seconds) that a client is allowed to stay connected
# Unset to disable. When set a client will be disconnected after being
# continuously connected for this amount of time, and its cookies will
# be invalidated (i.e., re-authentication will be required).
#session-timeout = 86400

# The time (in seconds) that a mobile client is allowed to stay idle (no
# traffic) before being disconnected. Unset to disable.
#mobile-idle-timeout = 2400

# The time (in seconds) that a client is not allowed to reconnect after 
# a failed authentication attempt.
min-reauth-time = 300

# Banning clients in ocserv works with a point system. IP addresses
# that get a score over that configured number are banned for
# min-reauth-time seconds. By default a wrong password attempt is 10 points,
# a KKDCP POST is 1 point, and a connection is 1 point. Note that
# due to difference processes being involved the count of points
# will not be real-time precise.
#
# Score banning cannot be reliably used when receiving proxied connections
# locally from an HTTP server (i.e., when listen-clear-file is used).
#
# Set to zero to disable.
max-ban-score = 80

# The time (in seconds) that all score kept for a client is reset.
ban-reset-time = 300

# In case you'd like to change the default points.
#ban-points-wrong-password = 10
#ban-points-connection = 1
#ban-points-kkdcp = 1

# Cookie timeout (in seconds)
# Once a client is authenticated he's provided a cookie with
# which he can reconnect. That cookie will be invalidated if not
# used within this timeout value. This cookie remains valid, during
# the user's connected time, and after user disconnection it
# remains active for this amount of time. That setting should allow a
# reasonable amount of time for roaming between different networks.
cookie-timeout = 300

# If this is enabled (not recommended) the cookies will stay
# valid even after a user manually disconnects, and until they
# expire. This may improve roaming with some broken clients.
#persistent-cookies = true

# Whether roaming is allowed, i.e., if true a cookie is
# restricted to a single IP address and cannot be re-used
# from a different IP.
deny-roaming = false

# ReKey time (in seconds)
# ocserv will ask the client to refresh keys periodically once
# this amount of seconds is elapsed. Set to zero to disable (note
# that, some clients fail if rekey is disabled).
rekey-time = 172800

# ReKey method
# Valid options: ssl, new-tunnel
#  ssl: Will perform an efficient rehandshake on the channel allowing
#       a seamless connection during rekey.
#  new-tunnel: Will instruct the client to discard and re-establish the channel.
#       Use this option only if the connecting clients have issues with the ssl
#       option.
rekey-method = ssl

# Script to call when a client connects and obtains an IP.
# The following parameters are passed on the environment.
# REASON, VHOST, USERNAME, GROUPNAME, DEVICE, IP_REAL (the real IP of the client),
# IP_REAL_LOCAL (the local interface IP the client connected), IP_LOCAL
# (the local IP in the P-t-P connection), IP_REMOTE (the VPN IP of the client),
# IPV6_LOCAL (the IPv6 local address if there are both IPv4 and IPv6
# assigned), IPV6_REMOTE (the IPv6 remote address), IPV6_PREFIX, and
# ID (a unique numeric ID); REASON may be "connect" or "disconnect".
# In addition the following variables OCSERV_ROUTES (the applied routes for this
# client), OCSERV_NO_ROUTES, OCSERV_DNS (the DNS servers for this client),
# will contain a space separated list of routes or DNS servers. A version
# of these variables with the 4 or 6 suffix will contain only the IPv4 or
# IPv6 values. The connect script must return zero as exit code, or the
# client connection will be refused.

# The disconnect script will receive the additional values: STATS_BYTES_IN,
# STATS_BYTES_OUT, STATS_DURATION that contain a 64-bit counter of the bytes 
# output from the tun device, and the duration of the session in seconds.

#connect-script = /usr/bin/myscript
#disconnect-script = /usr/bin/myscript

# UTMP
# Register the connected clients to utmp. This will allow viewing
# the connected clients using the command 'who'.
#use-utmp = true

# Whether to enable support for the occtl tool (i.e., either through D-BUS,
# or via a unix socket).
use-occtl = true

# PID file. It can be overridden in the command line.
pid-file = /var/run/ocserv.pid

# Set the protocol-defined priority (SO_PRIORITY) for packets to
# be sent. That is a number from 0 to 6 with 0 being the lowest
# priority. Alternatively this can be used to set the IP Type-
# Of-Service, by setting it to a hexadecimal number (e.g., 0x20).
# This can be set per user/group or globally.
#net-priority = 3

# Set the VPN worker process into a specific cgroup. This is Linux
# specific and can be set per user/group or globally.
#cgroup = "cpuset,cpu:test"

#
# Network settings
#

# The name to use for the tun device
device = vpns

# Whether the generated IPs will be predictable, i.e., IP stays the
# same for the same user when possible.
predictable-ips = true

# The default domain to be advertised
default-domain = example.com

# The pool of addresses that leases will be given from. If the leases
# are given via Radius, or via the explicit-ip? per-user config option then 
# these network values should contain a network with at least a single
# address that will remain under the full control of ocserv (that is
# to be able to assign the local part of the tun device address).
# Note that, you could use addresses from a subnet of your LAN network if you
# enable [proxy arp in the LAN interface](http://ocserv.gitlab.io/www/recipes-ocserv-pseudo-bridge.html);
# in that case it is recommended to set ping-leases to true.
ipv4-network = 172.16.0.0
ipv4-netmask = 255.255.255.0

# An alternative way of specifying the network:
#ipv4-network = 192.168.1.0/24

# The IPv6 subnet that leases will be given from.
#ipv6-network = fda9:4efe:7e3b:03ea::/48 

# Specify the size of the network to provide to clients. It is
# generally recommended to provide clients with a /64 network in
# IPv6, but any subnet may be specified. To provide clients only
# with a single IP use the prefix 128.
#ipv6-subnet-prefix = 128
#ipv6-subnet-prefix = 64

# Whether to tunnel all DNS queries via the VPN. This is the default
# when a default route is set.
#tunnel-all-dns = true

# The advertized DNS server. Use multiple lines for
# multiple servers.
# dns = fc00::4be0
dns = 192.168.1.2

# The NBNS server (if any)
#nbns = 192.168.1.3

# The domains over which the provided DNS should be used. Use
# multiple lines for multiple domains.
#split-dns = example.com

# Prior to leasing any IP from the pool ping it to verify that
# it is not in use by another (unrelated to this server) host.
# Only set to true, if there can be occupied addresses in the
# IP range for leases.
ping-leases = false

# Use this option to set a link MTU value to the incoming
# connections. Unset to use the default MTU of the TUN device.
# Note that the MTU is negotiated using the value set and the
# value sent by the peer.
mtu = 1420

# Unset to enable bandwidth restrictions (in bytes/sec). The
# setting here is global, but can also be set per user or per group.
#rx-data-per-sec = 40000
#tx-data-per-sec = 40000

# The number of packets (of MTU size) that are available in
# the output buffer. The default is low to improve latency.
# Setting it higher will improve throughput.
#output-buffer = 10

# Routes to be forwarded to the client. If you need the
# client to forward routes to the server, you may use the 
# config-per-user/group or even connect and disconnect scripts.
#
# To set the server as the default gateway for the client just
# comment out all routes from the server, or use the special keyword
# 'default'.

#route = 10.10.10.0/255.255.255.0
#route = 192.168.0.0/255.255.0.0
route = 0.0.0.0/128.0.0.0
route = 128.0.0.0/128.0.0.0
#route = fef4:db8:1000:1001::/64
#route = default

# Subsets of the routes above that will not be routed by
# the server.

#no-route = 192.168.5.0/255.255.255.0

# Note the that following two firewalling options currently are available
# in Linux systems with iptables software. 

# If set, the script /usr/bin/ocserv-fw will be called to restrict
# the user to its allowed routes and prevent him from accessing
# any other routes. In case of defaultroute, the no-routes are restricted.
# All the routes applied by ocserv can be reverted using /usr/bin/ocserv-fw
# --removeall. This option can be set globally or in the per-user configuration.
#restrict-user-to-routes = true

# This option implies restrict-user-to-routes set to true. If set, the
# script /usr/bin/ocserv-fw will be called to restrict the user to
# access specific ports in the network. This option can be set globally
# or in the per-user configuration.
#restrict-user-to-ports = "tcp(443), tcp(80), udp(443), sctp(99), tcp(583), icmp(), icmpv6()"

# You could also use negation, i.e., block the user from accessing these ports only.
#restrict-user-to-ports = "!(tcp(443), tcp(80))"

# When set to true, all client's iroutes are made visible to all
# connecting clients except for the ones offering them. This option
# only makes sense if config-per-user is set.
expose-iroutes = true

# Groups that a client is allowed to select from.
# A client may belong in multiple groups, and in certain use-cases
# it is needed to switch between them. For these cases the client can
# select prior to authentication. Add multiple entries for multiple groups.
# The group may be followed by a user-friendly name in brackets.
#select-group = group1
#select-group = group2[My special group]

# The name of the (virtual) group that if selected it would assign the user
# to its default group.
#default-select-group = DEFAULT

# Instead of specifying manually all the allowed groups, you may instruct
# ocserv to scan all available groups and include the full list.
#auto-select-group = true

# Configuration files that will be applied per user connection or
# per group. Each file name on these directories must match the username
# or the groupname.
# The options allowed in the configuration files are dns, nbns,
#  ipv?-network, ipv4-netmask, rx/tx-per-sec, iroute, route, no-route,
#  explicit-ipv4, explicit-ipv6, net-priority, deny-roaming, no-udp, 
#  keepalive, dpd, mobile-dpd, max-same-clients, tunnel-all-dns,
#  restrict-user-to-routes, cgroup, stats-report-time,
#  mtu, idle-timeout, mobile-idle-timeout, restrict-user-to-ports,
#  split-dns and session-timeout.
#
# Note that the 'iroute' option allows one to add routes on the server
# based on a user or group. The syntax depends on the input accepted
# by the commands route-add-cmd and route-del-cmd (see below). The no-udp
# is a boolean option (e.g., no-udp = true), and will prevent a UDP session
# for that specific user or group. The hostname option will set a
# hostname to override any proposed by the user. Note also, that, any 
# routes, no-routes, DNS or NBNS servers present will overwrite the global ones.

config-per-user = /etc/ocserv/config-per-user/
#config-per-group = /etc/ocserv/config-per-group/

# When config-per-xxx is specified and there is no group or user that
# matches, then utilize the following configuration.
#default-user-config = /etc/ocserv/defaults/user.conf
#default-group-config = /etc/ocserv/defaults/group.conf

# The system command to use to setup a route. %{R} will be replaced with the
# route/mask, %{RI} with the route in CIDR format, and %{D} with the (tun) device.
#
# The following example is from linux systems. %{R} should be something
# like 192.168.2.0/255.255.255.0 and %{RI} 192.168.2.0/24 (the argument of iroute).

#route-add-cmd = "ip route add %{R} dev %{D}"
#route-del-cmd = "ip route delete %{R} dev %{D}"

# This option allows one to forward a proxy. The special keywords '%{U}'
# and '%{G}', if present will be replaced by the username and group name.
#proxy-url = http://example.com/
#proxy-url = http://example.com/%{U}/

# This option allows you to specify a URL location where a client can
# post using MS-KKDCP, and the message will be forwarded to the provided
# KDC server. That is a translation URL between HTTP and Kerberos.
# In MIT kerberos you'll need to add in realms:
#   EXAMPLE.COM = {
#     kdc = https://ocserv.example.com/KdcProxy
#     http_anchors = FILE:/etc/ocserv-ca.pem
#   }
# In some distributions the krb5-k5tls plugin of kinit is required.
#
# The following option is available in ocserv, when compiled with GSSAPI support. 

#kkdcp = "SERVER-PATH KERBEROS-REALM PROTOCOL@SERVER:PORT"
#kkdcp = "/KdcProxy KERBEROS.REALM udp@127.0.0.1:88"
#kkdcp = "/KdcProxy KERBEROS.REALM tcp@127.0.0.1:88"
#kkdcp = "/KdcProxy KERBEROS.REALM tcp@[::1]:88"

# Client profile xml. This can be used to advertise alternative servers
# to the client. A minimal file can be:
# <?xml version="1.0" encoding="UTF-8"?>
# <AnyConnectProfile xmlns="http://schemas.xmlsoap.org/encoding/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://schemas.xmlsoap.org/encoding/ AnyConnectProfile.xsd">
#	<ServerList>
#		<HostEntry>
#	            <HostName>VPN Server name</HostName>
#	            <HostAddress>localhost</HostAddress>
#		</HostEntry>
#	</ServerList>
# </AnyConnectProfile>
#
# Other fields may be used by some of the CISCO clients.
# This file must be accessible from inside the worker's chroot. 
# Note that:
#  (1) enabling this option is not recommended as it will allow the
#      worker processes to open arbitrary files (when isolate-workers is
#      set to true).
#  (2) This option cannot be set per-user or per-group; only the global
#      version is being sent to client.
#user-profile = profile.xml

#
# The following options are for (experimental) AnyConnect client 
# compatibility. 

# This option will enable the pre-draft-DTLS version of DTLS, and
# will not require clients to present their certificate on every TLS
# connection. It must be set to true to support legacy CISCO clients
# and openconnect clients < 7.08. When set to true, it implies dtls-legacy = true.
cisco-client-compat = true

# This option allows one to disable the DTLS-PSK negotiation (enabled by default).
# The DTLS-PSK negotiation was introduced in ocserv 0.11.5 to deprecate
# the pre-draft-DTLS negotiation inherited from AnyConnect. It allows the
# DTLS channel to negotiate its ciphers and the DTLS protocol version.
#dtls-psk = false

# This option allows one to disable the legacy DTLS negotiation (enabled by default,
# but that may change in the future).
# The legacy DTLS uses a pre-draft version of the DTLS protocol and was
# from AnyConnect protocol. It has several limitations, that are addressed
# by the dtls-psk protocol supported by openconnect 7.08+.
dtls-legacy = true

#Advanced options

# Option to allow sending arbitrary custom headers to the client after
# authentication and prior to VPN tunnel establishment. You shouldn't
# need to use this option normally; if you do and you think that
# this may help others, please send your settings and reason to
# the openconnect mailing list. The special keywords '%{U}'
# and '%{G}', if present will be replaced by the username and group name.
#custom-header = "X-My-Header: hi there"



# An example virtual host with different authentication methods serviced
# by this server.

#[vhost:www.example.com]
#auth = "certificate"

#ca-cert = ../tests/certs/ca.pem

# The certificate set here must include a 'dns_name' corresponding to
# the virtual host name.

#server-cert = ../tests/certs/server-cert-secp521r1.pem
#server-key = ../tests/certs/server-key-secp521r1.pem

#ipv4-network = 192.168.2.0
#ipv4-netmask = 255.255.255.0

#cert-user-oid = 0.9.2342.19200300.100.1.1

```
Please read this configuration file carefully. You could make it work out of box by specific the `server-cert =`, `server-key =` and `ca-cert =`.


#### Enable Systemd

```
$ sudo bash -c 'cat> /lib/systemd/system/ocserv.service' << EOF
[Unit]
Description=OpenConnect SSL VPN server
Documentation=man:ocserv(8)
After=network-online.target
After=dbus.service

[Service]
PrivateTmp=true
PIDFile=/run/ocserv.pid
ExecStart=/usr/local/sbin/ocserv --foreground --pid-file /run/ocserv.pid --config /etc/ocserv/ocserv.conf
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF
```
Enable the service and start the ocserv service
```
sudo systemctl enable ocserv
sudo systemctl start ocserv
```
#### Firewall


edit `/etc/sysctl.d/99-sysctl.conf`

add following option
```    
net.ipv4.ip_forward=1
```

```
$ sudo sysctl -p

iptables -t nat -A POSTROUTING  -s 172.16.0.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -s 172.16.0.0/24 -d 172.16.0.0/24 -j DROP
```
(for test purpose)

####iptables persistence

```
iptables-save > /etc/iptables.up.rules

sudo bash -c 'cat > /etc/network/if-pre-up.d/iptables' << EOF
#!/bin/bash
/sbin/iptables-restore < /etc/iptables.up.rules
EOF

sudo chmod +x /etc/network/if-pre-up.d/iptables
```
#### Client side support

iOS  

	Anyconnect https://itunes.apple.com/us/app/cisco-anyconnect/id392790924?mt=8  

Android  

	Anyconnect https://play.google.com/store/apps/details?id=com.cisco.anyconnect.vpn.android.avf&hl=en  
	OpenConnect https://play.google.com/store/apps/details?id=app.openconnect&hl=en  

Windows  

	Anyconnect https://software.cisco.com/download/release.html?mdfid=286281283&softwareid=282364313&os=&release=4.3.03086&relind=AVAILABLE&rellifecycle=&reltype=latest&i=!pp  
	OpenConnect-gui http://openconnect.github.io/openconnect-gui/  

Mac OS X  

	Anyconnect https://software.cisco.com/download/release.html?mdfid=286281283&softwareid=282364313&os=&release=4.3.03086&relind=AVAILABLE&rellifecycle=&reltype=latest&i=!pp  
	OpenConnect-gui http://openconnect.github.io/openconnect-gui/  
	OpenConnect CLI brew install openconnect  

Linux  

	Anyconnect https://software.cisco.com/download/release.html?mdfid=286281283&softwareid=282364313&os=&release=4.3.03086&relind=AVAILABLE&rellifecycle=&reltype=latest&i=!pp  
	Openconnect http://www.infradead.org/openconnect/  

In our cipher suites, it would work with Anyconnect and OpenConnect :)
#### Reference:

[1] https://gitlab.com/ocserv/ocserv   
[2] https://www.gnutls.org/manual/html_node/Supported-ciphersuites.html   
[3] http://www.gnutls.org/manual/gnutls.html#Priority-Strings   
