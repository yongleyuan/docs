Load Balancing GridFTP
======================

!!! warning
    This document is for software that will no longer be supported after the OSG 3.5 retirement (February 2022).
    See the [Release Series Support Policy](https://opensciencegrid.org/technology/policy/release-series/) for details.

GridFTP is designed for high throughput data transfers and in many cases can handle all of the transfers for a site. However, in some cases it may be useful to run multiple GridFTP servers to distribute the load. For such sites, we recommend using a [load balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing)) to distribute requests and present the appearance of a single high-throughput GridFTP server.

One general-purpose technology for implementing a load balancer on Linux is [Linux Virtual Server](http://www.linuxvirtualserver.org/whatis.html) (LVS). To use it with GridFTP, a single load balancer listens on a virtual IP address, monitors the health of the set of real GridFTP servers, and forwards requests to available ones. Optionally, there can be one or more inactive, backup load balancers that can activate and take over the virtual IP address in case the primary load balancer fails, resulting in a system that is more resilient to failure. LVS is implemented by the [IP Virtual Server](http://www.linuxvirtualserver.org/software/ipvs.html) kernel module, which can be managed by userspace services on the load balancers such as [keepalived](http://www.keepalived.org).

This guide explains how to install, configure, run, test, and troubleshoot the `keepalived` service on a load balancing host for a set of [GridFTP](gridftp.md) servers.

Before Starting
---------------

Before starting the installation process, consider the following requirements:

-   There must be a shared file system for file propagation across GridFTP servers
-   You must have reserved a virtual IP address and associated virtual hostname

As with all OSG software installations, there are some one-time (per host) steps to prepare in advance:

-   Ensure the host has [a supported operating system](../release/supported_platforms.md)
-   Obtain root access to each host

Designing Your Load-Balanced GridFTP System
-------------------------------------------

Before beginning the installation process, you will need to plan the overall architecture of your load-balanced GridFTP system: the number of GridFTP servers, the type of shared file system to run on the GridFTP servers, whether or not backup load balancers are required, and hardware requirements.

### GridFTP servers

The number of GridFTP servers that you should run is determined first and foremost by the expected GridFTP transfer load at your site and the speed of the links available to each server. For example, if you expect a 20Gbps peak transfer load and have 10Gb links with 80–90% efficiency, you would need a minimum of 4 GridFTP servers: 3 to satisfy your desired throughput + 1 for failover or growth.

#### Shared file system

The number of GridFTP servers can also be determined by your hardware needs and by your choice of shared file system. If you choose a POSIX-based shared file system, plan for machines with more cores, or more GridFTP hosts to distribute the CPU load. If you are running [GridFTP with Hadoop](install-hadoop.md#gridftp-configuration), plan for machines with more memory, or more GridFTP hosts to distribute the memory load.

!!! note
    If you determine that you need only a single GridFTP host, you do not need load balancing. Instead, follow the [standalone-GridFTP installation guide](gridftp.md).

### Load balancer(s)

In the recommended direct routing mode, load balancers simply rewrite initial packets from a given request so the hardware requirements are minimal. When choosing load balancer hosts, aim for stability. If your chosen host is unstable or if you do not want to introduce downtime for operating system or hardware updates, at least one additional load balancer will be needed as a backup.

Preparing the GridFTP Servers
-----------------------------

Before adding your GridFTP hosts to the load-balanced system, each host requires the following:

* GridFTP software
* Special host certificates
* Load-balancing configuration

### Acquiring host certificate(s)

When authenticating with a GridFTP server, clients verify that the server's host certificate matches the hostname of the server. In the case of a load-balanced GridFTP system, clients contact the GridFTP server through the virtual hostname, so the GridFTP server will have to present a certificate containing the virtual hostname as well as the GridFTP server's hostname. Use the [OSG host certificate reference](../security/host-certs/overview.md) for more information on how to request these types of certificates.  Additionally, a special procedure is available to acquire [Let's Encrypt certificates](#with-lets-encrypt) with the load balanced gridftp.

If your GridFTP servers are also running XRootD, you will need unique certificates for each GridFTP server. Otherwise, you can request a single certificate that can be shared among the GridFTP servers.

#### Without XRootD

The single shared certificate must have the hostname associated with the load-balanced GridFTP system as its [common name](https://en.wikipedia.org/wiki/X.509_attribute_certificate) and each GridFTP servers hostname listed as [subject alternative names](https://en.wikipedia.org/wiki/Subject_Alternative_Name).

1.  Request and generate the shared certificate:

        :::console
        user@host $ osg-cert-request --hostname <VIRTUAL-HOSTNAME> \
                    --country <COUNTRY> \
                    --state <STATE> \
                    --locality <LOCALITY> \
                    --organization <ORGANIZATION>
                    --altname <GRIDFTP-SERVER-#1-HOSTNAME> \
                    --altname <GRIDFTP-SERVER-#2-HOSTNAME>

1.  Take the resulting CSR and get it signed by the appropriate authority.
    Most institutions can use InCommon as outlined [here](../security/host-certs/incommon.md).
2.  Create a directory to contain the shared certificate:

        :::console
        root@host # mkdir /etc/grid-security/gridftp

3.  Place the shared certificate-key pair in the newly created directory:

        :::console
        root@host # mv <PATH-TO-SERVICE-CERT> /etc/grid-security/gridftp/gridftp-hostcert.pem
        root@host # mv <PATH-TO-SERVICE-KEY> /etc/grid-security/gridftp/gridftp-hostkey.pem

1.  Edit `/etc/sysconfig/globus-gridftp-server` to identify the shared certificate-key pair:

        export X509_USER_CERT=/etc/grid-security/gridftp/gridftp-hostcert.pem
        export X509_USER_KEY=/etc/grid-security/gridftp/gridftp-hostkey.pem

#### With XRootD

XRootD requires that the certificate's [common name](https://en.wikipedia.org/wiki/X.509_attribute_certificate) refers specifically to the host it resides on. To ensure each GridFTP server can authenticate using the virtual hostname, add it as the [subject alternative name](https://en.wikipedia.org/wiki/Subject_Alternative_Name) for each certificate.

1.  Create a list of GridFTP server hostnames in `load-balanced-hosts.txt`:

        <GRIDFTP-SERVER-#1-HOSTNAME> <VIRTUAL-HOSTNAME>
        <GRIDFTP-SERVER-#2-HOSTNAME> <VIRTUAL-HOSTNAME>
        [...]

1.  Submit a batch request for the per-GridFTP server certificates:

        :::console
        user@host $ osg-gridadmin-cert-request -f load-balanced-hosts.txt

1.  Copy the resulting certificates and keys to their corresponding GridFTP servers in `/etc/grid-security/hostcert.pem` and `/etc/grid-security/hostkey.pem`, respectively.

#### With Let's Encrypt

The certificate provided to the clients needs to have the virtual host address of the load balancer, as well as the hostname of each of the worker nodes.  Additionally, LetsEncrypt contacts the requested domains to verify ownership.  Therefore, each domain requested must be available to respond to HTTP requests at the same time.  The easiest method for this is to use a shared directory for Let's Encrypt's `certbot` to install the secrets.

The procedure to acquire Let's Encrypt certificates for multiple hosts is as follows:

1. Create or use a shared directory that each of the data transfer nodes can read, for example a simple NFS share.  The steps in creating a NFS shared directory is outside the scope of this guide.  In this guide, the shared directory will be referred as `/mnt/nfsshare` . 

2. Install `httpd` on each of the data transfer nodes:

        :::console
        root@host $ yum install httpd

    Create a webroot directory within the shared directory on one of the nodes:

        :::console
        root@host $ mkdir /mnt/nfsshare/webroot

3. Configure `httpd` to export the same webroot on each of the data transfer nodes:

        <VirtualHost *:80>
            DocumentRoot "/mnt/nfsshare/webroot"
            <Directory "/mnt/nfsshare/webroot">
                Require all granted
            </Directory>
        </VirtualHost>


4. Configure `keepalived` to virtualize port 80 to at least one of your data transfer nodes.
    Add to your configuration:


        virtual_server <VIRTUAL-IP-ADDRESS> 80 {
            delay_loop 10
            lb_algo wlc
            lb_kind DR
            protocol tcp

            real_server <GRIDFTP-SERVER-#1-IP ADDRESS> {
                TCP_CHECK {
                    connect_timeout 3
                    connect_port 80
                }
            }
        }


5. Run `certbot` with the webroot options on only 1 of the data nodes.  The first domain in the command line should be the virtual hostname:

        root@host $ certbot certonly -w /mnt/nfsshare/webroot -d <VIRTUAL_HOSTNAME> -d <DATANODE_1> -d <DATANODE_N>...

For XRootD certificates, the real hostname of the XRootD node is required to be the first hostname in the `certbot` command.  You may run the `certbot` command several times on the same host, replacing the `VIRTUAL_HOSTNAME` with the real hostname of the XRootD servers, and placing the `VIRTUAL_HOSTNAME` in in the list of other domains in the certificate.

### Installing GridFTP

Whether you are starting from scratch or adding more GridFTP servers to your load-balanced GridFTP system, follow the documentation for [installing a standalone GridFTP server](gridftp.md) for each of your intended GridFTP servers (skip section 2.2, requesting a certificate). For hosts with GridFTP already installed, skip this section.

### Configuring your GridFTP servers

Each GridFTP server requires changes to its IP configuration and potentially its arptables:

-   [Adding your virtual IP address](#adding-your-virtual-ip-address)
-   [Disabling ARP](#disabling-arp) − if your GridFTP servers are on the same network segment as the virtual IP

#### Adding your virtual IP address

Use the virtual IP address of your load balancer(s) as the secondary IPs of each of your GridFTP servers.

1.  Add the virtual IP using the `ip` tool:

        :::console
        root@host # ip addr add <VIRTUAL-IP-ADDRESS>/<SUBNET-MASK> dev <NETWORK-INTERFACE>

1.  To persist the virtual IP changes across reboots, edit `/etc/rc.d/rc.local`, and add the same command as used above.
1.  Make sure that `/etc/rc.d/rc.local` is executable:

        :::console
        root@host # chmod u+x /etc/rc.d/rc.local

#### Disabling ARP

If your GridFTP servers and load balancer(s) are on the same network segment, you will have to disable ARP on the GridFTP servers to avoid [ARP race conditions](http://kb.linuxvirtualserver.org/wiki/ARP_Issues_in_LVS/DR_and_LVS/TUN_Clusters). Otherwise, skip to [the section on preparing keepalived](#preparing-keepalived-load-balancers).

1. Install the arptables software:

        :::console
        root@host # yum install arptables

1. Disable ARP:

        :::console
        root@host # arptables -F 
        root@host # arptables -A IN -d <VIRTUAL-IP-ADDRESS> -j DROP 
        root@host # arptables -A OUT -s <VIRTUAL-IP-ADDRESS> -j mangle --mangle-ip-s <GRIDFTP-REAL-IP-ADDRESS>
        
1. Save ARP tables to survive reboots:

        :::console
        root@host # arptables-save > /etc/sysconfig/arptables

Preparing Keepalived Load Balancer(s)
-------------------------------------

### Installing Keepalived

Whether you run a single load balancer, or have one active load balancer and some inactive backups, each load balancer host must have the `keepalived` software installed, configured, and running.

!!! note
    Do not install `keepalived` on the GridFTP servers themselves.

The `keepalived` package is available from standard operating system repositories. Install it on each load balancer host using the following commands:

1. Clean yum cache:

        ::console
        root@host # yum clean all --enablerepo=*

1. Update software:

        :::console
        root@host # yum update

    This command will update **all** packages

1. Install the `keepalived` package:

        root@host # yum install keepalived

### Required configuration

On the primary load balancer, edit `/etc/keepalived/keepalived.conf`:

``` file
global_defs {
   router_id <STRING-LABEL-FOR-YOUR-LOAD-BALANCED-SYSTEM>
}

vrrp_instance VI_gridftp {
    state MASTER
    interface <NETWORK-INTERFACE>
    virtual_router_id <INTEGER-BETWEEN-0-AND-255>
    priority 100
    virtual_ipaddress {
        <VIRTUAL-IP-ADDRESS>/<SUBNET-MASK> dev <NETWORK-INTERFACE>
    }
}

virtual_server <VIRTUAL-IP-ADDRESS> 2811 {
    delay_loop 10
    lb_algo wlc
    lb_kind DR
    protocol tcp

    real_server <GRIDFTP-SERVER-#1-IP ADDRESS> {
        TCP_CHECK {
            connect_timeout 3
            connect_port 2811
        }
    }
    real_server <GRIDFTP-SERVER-#2-IP-ADDRESS> {
        TCP_CHECK {
            connect_timeout 3
            connect_port 2811
        }
    }
    [...]
}
```

!!! note
    Use the same `VIRTUAL-IP-ADDRESS` throughout the configuration of your load-balanced GridFTP system.

!!! note
    In the `virtual_server` section, write one `real_server` subsection for each GridFTP server behind the load balancer.

### Optional configuration

The following configuration steps are optional and will likely not be required for setting up a small cluster of GridFTP hosts. If you do not need any of the following special configurations, skip to [the section on using keepalived](#using-keepalived).

-   [Adding backup load balancers](#adding-backup-load-balancers)
-   [Enabling e-mail notifications](#enabling-e-mail-notifications)

#### Adding backup load balancers

If you need to add backup load balancers, copy `/etc/keepalived/keepalived.conf` from your primary load balancer and change the `state` and `priority` attributes under your `vrrp_instance VI_gridftp` section:

!!! note
    Priority specifies the order of preferred load balancer fallback, where larger values corresponds to a higher preference.

``` file
vrrp_instance VI_gridftp {
    state BACKUP
    interface <NETWORK-INTERFACE>
    virtual_router_id <SAME-ID-AS-MASTER-LOAD-BALANCER>
    priority <PRIORITY-INTEGER>
    virtual_ipaddress {
        <VIRTUAL-IP-ADDRESS>/<SUBNET-MASK> dev <NETWORK-INTERFACE>
    }
}
```

#### Enabling e-mail notifications

To receive e-mails when the state of your load-balanced system changes, update the `global_defs` section of `/etc/keepalived/keepalived.conf` for each of your load balancer nodes:

``` file
notification_email {
<NOTIFY-EMAIL-ADDRESS-#1>
<NOTIFY-EMAIL-ADDRESS-#2>
[...]
}
notification_email_from <FROM-EMAIL-ADDRESS>
smtp_server <SMTP-SERVER-IP-ADDRESS>
smtp_connect_timeout 60
router_id <MACHINE-IDENTIFYING-STRING>
```

Using Your Load Balanced GridFTP System
---------------------------------------

### Using GridFTP

On the GridFTP servers, arptables is the only additional service required for running a load-balanced GridFTP system.
Manage the service with the following commands:

| **To ...**                                    | **Run the command...**        |
| --------                                      | ----------------------------- |
| Start the service                             | `systemctl start arptables`   |
| Stop theservice                               | `systemctl stop arptables`    |
| Enable the service to start during boot       | `systemctl enable arptables`  |
| Disable the service from starting during boot | `systemctl disable arptables` |

For information on how to use your individual GridFTP servers, please refer to the [Managing GridFTP section](gridftp.md#managing-gridftp) of the GridFTP installation guide.

### Using Keepalived

On the load balancer nodes, `keepalived` is the only additional service required for running a load-balanced GridFTP system. As a reminder, here are common service commands (all run as `root`):

| To...                                       | Run the command...             |
|:--------------------------------------------|:-------------------------------|
| Start a service                             | `systemctl start keepalived`   |
| Stop a service                              | `systemctl stop keepalived`    |
| Enable a service to start during boot       | `systemctl enable keepalived`  |
| Disable a service from starting during boot | `systemctl disable keepalived` |

Getting Help
------------

To get assistance with `keepalived` in front of OSG Software services, please use the [this page](../common/help.md).

---------

-   [Linux Virtual Server homepage](http://www.linuxvirtualserver.org/whatis.html)
-   [Keepalived homepage](http://www.keepalived.org/index.html)
-   [RHEL 7 Load Balancer Administration Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/load_balancer_administration/index)
-   [T2 Nebraska LVS installation notes](https://github.com/gattebury/gridftp-with-lvs)


