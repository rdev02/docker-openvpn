# Advanced Configurations

The [`ovpn_genconfig`](/bin/ovpn_genconfig) script is intended for simple configurations that apply to the majority of the users.  If your use case isn't general, it likely won't be supported.  This document aims to explain how to work around that.

## Create host volume mounts rather than data volumes

* Refer to the Quick Start document, and substitute `--volumes-from $OVPN_DATA` with `-v /path/on/host/openvpn0:/etc/openvpn`
* Quick example that is likely to be out of date, but here's how to get started:

        mkdir openvpn0
        cd openvpn0
        docker run --rm -v $PWD:/etc/openvpn rdev02/docker-openvpn ovpn_genconfig -u udp://VPN.SERVERNAME.COM:1194
        docker run --rm -v $PWD:/etc/openvpn -it rdev02/docker-openvpn ovpn_initpki
        vim openvpn.conf
        docker run --rm -v $PWD:/etc/openvpn -it rdev02/docker-openvpn easyrsa build-client-full CLIENTNAME nopass
        docker run --rm -v $PWD:/etc/openvpn rdev02/docker-openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn

* Start the server with:

        docker run -v $PWD:/etc/openvpn -d -p 1194:1194/udp --privileged rdev02/docker-openvpn
