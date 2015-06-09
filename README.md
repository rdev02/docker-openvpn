### **Original credit: https://github.com/kylemanna/docker-openvpn**
# OpenVPN for Docker

OpenVPN server in a Docker container complete with an EasyRSA PKI CA.
This fork contains changes necessary to generate config and run another instance of vpn on tcp

## Quick Start

* Create the the keys and clients: run this on a machine you trust

        1. docker run --rm -it -v ~/openvpn_udp:/etc/openvpn rdev02/docker-openvpn ovpn_genconfig -u udp://yourserver.com
        2. docker run --rm -it -v ~/openvpn_udp:/etc/openvpn rdev02/docker-openvpn ovpn_initpki
        3. docker run --rm -it -v ~/openvpn_udp:/etc/openvpn rdev02/docker-openvpn ovpn_copy_server_files
        4. docker run --rm -it -v ~/openvpn_udp:/etc/openvpn rdev02/docker-openvpn easyrsa build-client-full my_client_name nopass
        5. docker run --rm -it -v ~/openvpn_udp:/etc/openvpn rdev02/docker-openvpn ovpn_getclient my_client_name > my_client_name_udp.ovpn        
  Repeat steps 4-5 for any number of clients you have. in the folder that you executed this you will have client ovpn files which you can use to connect to the server. In the ~/openvpn_udp folder
  you will have all the configuration, which, at this point, you can back up.

* Copy client generated certs to server config 

        cp ~/openvpn_udp/pki/issued/* ~/openvpn_udp/server/pki/issued
        cp ~/openvpn_udp/pki/private/* ~/openvpn_udp/server/pki/private

* Copy ~/openvpn_udp/server to the server, where open vpn will be running
* Run open vpn(suppose you copied server files to /etc/openvpn_udp)

        docker run -v /etc/openvpn_udp:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN rdev02/docker-openvpn

* Now you can connect with your client
* Repeat all the above changing 'udp' to 'tcp'. Don't forget to specify port in your tcp vpn server url.

## How Does It Work?

Automatically generate:

- Diffie-Hellman parameters
- a private key
- a self-certificate matching the private key for the OpenVPN server
- an EasyRSA CA key and certificate
- a TLS auth key from HMAC security

The OpenVPN server is started with the default run cmd of `ovpn_run`

The configuration is located in `/etc/openvpn`, and the Dockerfile
declares that directory as a volume. It means that you can start another
container with the `-v` flag, and access different set of configuration(tcp).
The volume also holds the PKI keys and certs so that it could be backed up.

To generate a client certificate, `rdev02/docker-openvpn` uses EasyRSA via the
`easyrsa` command in the container's path.  The `EASYRSA_*` environmental
variables place the PKI CA under `/etc/opevpn/pki`.

Conveniently, `rdev02/docker-openvpn` comes with a script called `ovpn_getclient`,
which dumps an inline OpenVPN client configuration file.  This single file can
then be given to a client for access to the VPN.


## OpenVPN Details

We use `tun` mode, because it works on the widest range of devices.
`tap` mode, for instance, does not work on Android, except if the device
is rooted.

The topology used is `net30`, because it works on the widest range of OS.
`p2p`, for instance, does not work on Windows.

The UDP server uses`192.168.255.0/24` for dynamic clients by default.

The client profile specifies `redirect-gateway def1`, meaning that after
establishing the VPN connection, all traffic will go through the VPN.
This might cause problems if you use local DNS recursors which are not
directly reachable, since you will try to reach them through the VPN
and they might not answer to you. If that happens, use public DNS
resolvers like those of Google (8.8.4.4 and 8.8.8.8) or OpenDNS
(208.67.222.222 and 208.67.220.220).


## Security Discussion

The Docker container runs its own EasyRSA PKI Certificate Authority.  This was
chosen as a good way to compromise on security and convenience.  The container
runs under the assumption that the OpenVPN container is running on a secure
host, that is to say that an adversary does not have access to the PKI files
under `/etc/openvpn/pki`.  This is a fairly reasonable compromise because if an
adversary had access to these files, the adversary could manipulate the
function of the OpenVPN server itself (sniff packets, create a new PKI CA, MITM
packets, etc).

* The certificate authority key is kept in the container by default for
  simplicity.  It's highly recommended to secure the CA key with some
  passphrase to protect against a filesystem compromise.  A more secure system
  would put the EasyRSA PKI CA on an offline system (can use the same Docker
  image and the script [`ovpn_copy_server_files`](/docs/clients.md) to accomplish this).
* It would be impossible for an adversary to sign bad or forged certificates
  without first cracking the key's passphase should the adversary have root
  access to the filesystem.
* The EasyRSA `build-client-full` command will generate and leave keys on the
  server, again possible to compromise and steal the keys.  The keys generated
  need to signed by the CA which the user hopefully configured with a passphrase
  as described above.
* Assuming the rest of the Docker container's filesystem is secure, TLS + PKI
  security should prevent any malicious host from using the VPN.


## Benefits of Running Inside a Docker Container

### The Entire Daemon and Dependencies are in the Docker Image

This means that it will function correctly (after Docker itself is setup) on
all distributions Linux distributions such as: Ubuntu, Arch, Debian, Fedora,
etc.  Furthermore, an old stable server can run a bleeding edge OpenVPN server
without having to install/muck with library dependencies (i.e. run latest
OpenVPN with latest OpenSSL on Ubuntu 12.04 LTS).

### It Doesn't Stomp All Over the Server's Filesystem

Everything for the Docker container is contained in two images: the ephemeral
run time image (kylemanna/openvpn) and the data image (using busybox as a
base).  To remove it, remove the two Docker images and corresponding containers
and it's all gone.  This also makes it easier to run multiple servers since
each lives in the bubble of the container (of course multiple IPs or separate
ports are needed to communicate with the world).

### Some (arguable) Security Benefits

At the simplest level compromising the container may prevent additional
compromise of the server.  There are many arguments surrounding this, but the
take away is that it certainly makes it more difficult to break out of the
container.  People are actively working on Linux containers to make this more
of a guarantee in the future.

## Differences from kylemanna/docker-openvpn

* expose port 443/tcp
* fix the bug where tcp port was not written to configuration properly
* move away from data containers to mounted volumes

## Tested On

* Docker hosts:
  * server a [Digital Ocean](https://www.digitalocean.com/?refcode=d19f7fe88c94) Droplet with 512 MB RAM running Ubuntu 14.04
* Clients
  * Android App OpenVPN Connect 1.1.14 (built 56)
     * OpenVPN core 3.0 android armv7a thumb2 32-bit
  * OS X Mavericks with Tunnelblick 3.4beta26 (build 3828) using openvpn-2.3.4
  * ArchLinux OpenVPN pkg 2.3.4-1
