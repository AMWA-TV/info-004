# Practical Implementation Example
{:.no_toc}

- This will be replaced with a table of contents

{:toc}

There are many DNS server options, but for the purposes of this doc, for the MDS (Media DNS Server), we’ll use BIND on Linux, a very popular open source solution.

What services does the NMOS RDS need to serve? From the [AMWA NMOS IS-04 specification](https://specs.amwa.tv/is-04/releases/v1.3.1/docs/3.0._Discovery.html), the following services should be configured:

- `_nmos-register._tcp`: A logical host which advertises a Registration API.
- `_nmos-query._tcp`: A logical host which advertises a Query API.

The following service does need not to be configured:

- `_nmos-node._tcp`: A logical host which advertises a Node API.

## DNS Build / Configuration

We’ll be running BIND on a CentOS 7 installation. We’ll assume that the CentOS 7 installation has been completed, and has a static IP address, with access to the Internet, which we’ll use for initial installation of BIND (it can be disconnected later)


### Installing BIND

BIND9 can be installed from the linux command line (as root), with the following:

```
sudo yum install bind bind-utils
```

### Configure BIND - General configuration

The basic BIND install provides a general config file which you can find at `/etc/named.conf`

Edit this file to add the following ACL (Access Control List) above the config section, defining who can access the DNS server; Devices on 192.168.0.0/24, locally attached networks, and the local machine. Add more networks as needed.

```
acl AllowQuery {
        192.168.0.0/24;
        localhost;
        localnets;
};
```

Then adjust the `allow-query` section within the `options` section to reference this ACL.

```
options {
allow-query     { AllowQuery; };
```

Also within the `options` section, define which network interfaces to allow DNS requests on:

```
listen-on port 53 { 127.0.0.1; 192.168.0.18; };
```

If desired, configure the next upstream DNS server - this is likely to be a corporate DNS service on the Internet the internet (`8.8.8.8`, or `1.1.1.1` for example) or might be in the local lab.

```
forwarders { 192.168.0.19; }; #IP of upstream nameserver(s)
recursion yes;
```

### Configure the Zones file

This file provides information about the zones (domains) that you want the DNS server to support - for this example, we’ll assume a single zone.

Create and edit the file we previously referenced in the /etc/named.conf file ( `include "/etc/named/named.conf.local";)`

The file should look something like the following, which can be used for the domain “gplab.home.com”, and references a file for this zone: `/etc/named/zones/db.gplab.home.local`

### Configure the hosts / TXT / SRV file

Now we create the file that contains nameserver, hosts, SRV and TXT records for the `gplab.home.com` domain, which needs to be located and called the same filename, as specified above in the zone config: `/etc/named/zones/db.gplab.home.local`

We define the global TTL (Time to live is seconds) for this zone (`gplab.home.com`), as 3600 seconds. `Serial` provides a timestamp that will be used when we synchronize a secondary DNS server later.

```
TTL 3600
@       IN      SOA     dns1.gplab.home.com. admin.gplab.home.com. (
           20210713     ; Serial
               3600     ; Refresh
                600     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```

Then we define our main and backup DNS server for this zone/domain. End-points should be configured with these DNS servers.

```
; DNS servers
        IN      NS      dns1.gplab.home.com.
        IN      NS      dns2.gplab.home.com.
```

The NMOS register services are defined:

```
; These lines indicate to clients that this server supports DNS Service
; Discovery
b._dns-sd._udp	IN	PTR	@
lb._dns-sd._udp	IN	PTR	@
```

These lines indicate to clients which service types this server may advertise:

```
_services._dns-sd._udp	PTR	_nmos-register._tcp
_services._dns-sd._udp	PTR	_nmos-query._tcp
```

There should be one PTR record for each instance of the service you wish to advertise. Here we have one Registration API and one Query API:

```
_nmos-register._tcp	PTR	reg-api-1._nmos-register._tcp
_nmos-query._tcp		PTR	qry-api-1._nmos-query._tcp
```

If more than one RDS server can be used, then two records can be provided, with a priority setting to enable end-points to make a preferred decision. In this case, the first is prioritized, with the “10” beating “20”.

The TXT records indicate additional metadata relevant to the IS-04 spec.
