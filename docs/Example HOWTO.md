# Example HOWTO
{:.no_toc}

- This will be replaced with a table of contents
{:toc}

## Introduction

This is a practical example of how to implement DNS-SD on a Linux system using the BIND9 DNS server.
It is meant to provide a relatively simple, easy-to-follow "recipe" for getting DNS-SD up and running in a short period of time.
It is not intended to be a tutorial article, and therefore, it excludes explanation regarding DNS parameters, or why they are being used.
A search of the Internet will turn up many excellent tutorials for DNS and DNS-SD, and of course, a number of excellent books have been written on the subject of DNS and specifically BIND9.
We hope you find this how-to guide to be useful.

There are many Domain Name Service (DNS) server options, but for the purposes of this guide we will use BIND on Linux, a very popular open source solution.

What services does the Networked Media Open Specifications (NMOS) Registration and Discovery Service (RDS) server need to provide?
From the [AMWA NMOS IS-04 specification](https://specs.amwa.tv/is-04/releases/v1.3.1/docs/3.0._Discovery.html), the following services should be configured:

- `_nmos-register._tcp`: A logical host which advertises a Registration API.
- `_nmos-query._tcp`: A logical host which advertises a Query API.

The following service does need not to be configured:

- `_nmos-node._tcp`: A logical host which advertises a Node API.

## DNS Build / Configuration

This guide has been tested on CentOS 7, Ubuntu 16.04, and Raspbian (Buster).
These instructions should work for Debian and may other versions of Linux as well.

We assume that the Linux installation has been completed, the server has a static IP address, and that it has access to the Internet.
(The Internet connection may be disconnected once the BIND installation is complete.)

### Installing BIND

BIND9 can be installed from the Linux command line (as root), with the following:

CentOS 7
```
sudo yum install bind bind-utils
```

Ubuntu
```
apt update && apt install bind9 bind-utils
```

Raspbian
```
apt update && apt install bind9
```

### Adding the example zone to the DNS server

In this example, we will be working with the domain `example.com`.
We must add this domain to the configuration file of the DNS server so that it knows it is responsible for responding to queries for this domain (referred to as a 'zone' in BIND).
Zone information is kept in the following directories, depending upon your Linux version:

CentOS 7 `/etc/bind/named.conf`
Ubuntu/Raspbian `/etc/bind/named.conf.local`

To add the zone, enter the information below in the appropriate configuration file for your distribution.

```
zone "example.com" {
        type master;
        file "/etc/bind/zones/db.example.com";
};
```

### Configuring the Zone file

You will notice the line `file "/etc/bind/zones/db.example.com";` in the configuration above.
This is a pointer to the zone file for the domain `example.com`.
In order for the DNS server to work properly, you will need to create this zone file.
But first create the directory `/etc/bind/zones/`.

```
# cd /etc/bind
# mkdir zones
# cd zones
```

Next, use any text editor to create the file `db.example.com` in the `/etc/bind/zones` directory based upon the directions below.

### Configuring the hosts / TXT / SRV file

We begin by defining several required DNS parameters.

```
$TTL 3600
@       IN      SOA     dns1.example.com. admin.example.com. (
           20210713     ; Serial
               3600     ; Refresh
                600     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```

Then we define our DNS server for this zone. Endpoints should be configured with this DNS server.

```
; DNS server
example.com.        IN      NS      dns1.example.com.
```

We add the following PTR records to indicate the server supports Service Discovery.
Clients looking for DNS-SD support will query the server for these specific records (`b` is for "browse", `lb` is for "legacy browse".)

```
; These lines indicate to clients that this server supports DNS Service Discovery.
b._dns-sd._udp  IN      PTR     @
lb._dns-sd._udp IN      PTR     @
```

Next we define the NMOS services provided by this server.

```
; These lines indicate to clients which NMOS service types this server advertises.
_services._dns-sd._udp  PTR     _nmos-register._tcp
_services._dns-sd._udp  PTR     _nmos-query._tcp
```

There should be one `PTR` record for each instance of the service you wish to advertise.
Here we have one instance of the Registration API service and one instance of the Query API service.

```
_nmos-register._tcp     PTR     reg-api-1._nmos-register._tcp
_nmos-query._tcp        PTR     qry-api-1._nmos-query._tcp
```

Now we add `SRV` records that specify the target for the registration and query services.
In this case, both of the records point to `rds1.example.com`.

```
; NMOS Registration and Query Services        Type   Priority  Weight   Port   Host 
reg-api-1._nmos-register._tcp.example.com.     SRV     10        10      80    rds1.example.com.
qry-api-1._nmos-query._tcp.example.com.        SRV     10        10      80    rds1.example.com.
```

We add `TXT` records which provide information relevant to the IS-04 specification.

```
reg-api-1._nmos-register._tcp.example.com.     TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=0" "api_auth=false"
qry-api-1._nmos-query._tcp.example.com.        TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=0" "api_auth=false"
```

In both cases above the `SRV` records tell clients to access the server using port `80`.
This would suit default HTTP access, but if HTTPS is used, this would need to be changed to `443`.

Lastly we provide the IP addresses for the hosts in the system.
This file can of course be expanded to contain names for all the hosts, endpoints, and switches in the system, making debugging simpler.

```
; Nameserver records    Class  Type     Target
dns1.example.com.         IN     A      192.168.0.18
rds1.example.com.         IN     A      192.168.0.50
```

### Starting the service

Once the files are in place, the service can be started and made permanent (will run after a reload/reboot of the server):

CentOS 7
```
systemctl restart named
systemctl enable named
```

Ubuntu/Raspbian
```
systemctl restart bind9
systemctl enable bind9
```

### Enabling DNS through the Linux firewall

Often, Linux default will prevent DNS through the firewall, so you have issues with connectivity to the DNS server now running.
The following will enable DNS through the CentOS 7 firewall, and make this permanent:

```
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```

## Testing

You can test the functionality of the new DNS server right from the server itself.
You can also run these tests from a Linux box or Mac by setting the `nameserver` portion of the Linux or Mac box network configuration to point to your new DNS server.
If you run the tests below from a separate computer, omit `localhost` or `@localhost` from the commands below.

There are a couple of tools that allow the DNS operation to be tested.

We can verify that the host names of the RDS servers are configured:

```
# nslookup rds1.example.com localhost
Server:         192.168.0.18
Address:        192.168.0.18#53

Name:           rds1.example.com
Address:        192.168.0.50
```

We can see that the lookup was resolved by `192.168.0.18`, and resulted in the address for the `rds1` server being returned as `192.168.0.50`.

The `dig` tool provides a little more info:

```
# dig @localhost rds1.example.com

; <<>> DiG 9.10.6 <<>> rds1.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12178
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;rds1.example.com.                IN      A

;; ANSWER SECTION:
rds1.example.com. 3600    IN      A       192.168.0.50

;; AUTHORITY SECTION:
example.com.      3600    IN      NS      dns2.example.com.
example.com.      3600    IN      NS      dns1.example.com.

;; ADDITIONAL SECTION:
dns1.example.com. 3600    IN      A       192.168.0.18
dns2.example.com. 3600    IN      A       192.168.0.20

;; Query time: 43 msec
;; SERVER: 192.168.0.18#53(192.168.0.18)
;; WHEN: Tue Jul 13 19:58:50 IST 2021
;; MSG SIZE  rcvd: 134
```

### Using Dig to Browse NMOS Related DNS Items

We can use `dig` to do a top down browse of the NMOS services being served by BIND9.
First we simply check if our DNS is offering up the NMOS services by looking at all the pointers to services our DNS is providing:

```
# dig _services._dns-sd._udp.example.com PTR
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> PTR _services._dns-sd._udp.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4132
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;_services._dns-sd._udp.example.com. IN	PTR

;; ANSWER SECTION:
_services._dns-sd._udp.example.com. 3600 IN PTR	_nmos-query._tcp.example.com.
_services._dns-sd._udp.example.com. 3600 IN PTR	_nmos-register._tcp.example.com.

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns1.example.com.

;; ADDITIONAL SECTION:
dns1.example.com.	3600	IN	A	192.168.0.18

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Fri Sep 02 13:02:59 UTC 2022
;; MSG SIZE  rcvd: 158
```

We see in the ANSWER section that our DNS has the two entries we included in our BIND9 database file for our NMOS registry and NMOS query type services.

### Checking the `PTR` records

Next we use dig to query the DNS for PTR records associated with the entries obtained in our last step. 

```
# dig _nmos-register._tcp.example.com PTR

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> _nmos-register._tcp.example.com PTR
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60489
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;_nmos-register._tcp.example.com. IN	PTR

;; ANSWER SECTION:
_nmos-register._tcp.example.com. 3600 IN PTR	reg-api-1._nmos-register._tcp.example.com.

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns1.example.com.

;; ADDITIONAL SECTION:
dns1.example.com.	3600	IN	A	192.168.0.18

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Fri Sep 02 13:18:03 UTC 2022
;; MSG SIZE  rcvd: 119
```

### Checking the `SRV` records

Finally we can resolve the actual host IP and port for the returned registry using

```
# dig reg-api-1._nmos-register._tcp.example.com SRV

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> reg-api-1._nmos-register._tcp.example.com SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25267
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;reg-api-1._nmos-register._tcp.example.com. IN SRV

;; ANSWER SECTION:
reg-api-1._nmos-register._tcp.example.com. 3600	IN SRV 10 10 80 rds1.example.com.

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns1.example.com.

;; ADDITIONAL SECTION:
rds1.example.com.	3600	IN	A	192.168.0.50
dns1.example.com.	3600	IN	A	192.168.0.18

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Fri Sep 02 13:19:23 UTC 2022
;; MSG SIZE  rcvd: 157
```

The Additional Section contains the resolved IP address for the host while the answer section shows we have the correct port that we included in the BIND9 database.

## Providing Back-up Servers

To provide resilience, a secondary DNS server and a secondary RDS server should be provisioned.
Endpoints should have both DNS IP addresses configured, allowing them to use the secondary DNS server, if the primary is no longer available.

BIND allows primary / secondary pairing, so that the zones and hosts configuration can be automatically updated on the secondary device, reducing the amount of duplication.

There should be one `PTR` record for each instance of the service you wish to advertise.
Here we have two instances of the Registration API service and one instance of the Query API service.

```
_nmos-register._tcp     PTR     reg-api-1._nmos-register._tcp
_nmos-register._tcp     PTR     reg-api-2._nmos-register._tcp
_nmos-query._tcp        PTR     qry-api-1._nmos-query._tcp
```

If more than one RDS server can be used, then two records can be provided, with a priority setting to enable endpoints to make a preferred decision.
In this case, the first is prioritized, with the `10` beating `20`.

```
; NMOS Registration and Query Services        Type   Priority  Weight   Port   Host 
; High Priority Registration Service
reg-api-1._nmos-register._tcp.example.com.     SRV     10        10      80    rds1.example.com.

; Lower Priority Registration Service
reg-api-2._nmos-register._tcp.example.com.     SRV     20        10      80    rds2.example.com.
```

We also provide `TXT` record for the both the primary and secondary service instances, with information relevant to the IS-04 specification.
NMOS also includes priority information here as many DNS-SD clients ignore the priority and weight information in `SRV` records.

```
; High Priority Registration Service
reg-api-1._nmos-register._tcp.example.com.     TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=10" "api_auth=false"

; Lower Priority Registration Service
reg-api-2._nmos-register._tcp.example.com.     TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=20" "api_auth=false"
```

Take advice from the RDS vendor about how to set the priority and weight.
If active-active is available in the RDS servers, then these records can be used to provide load-balancing.

In all cases above the `SRV` records are identifying a port number of `80`.
This would suit default HTTP access.
For HTTPS access (see [BCP-003-01](https://specs.amwa.tv/bcp-003-01)) this would be set to `443`, and the `TXT` records would include `api_proto=https`.
Again, this would be a question for the RDS vendor.

Lastly we provide the IP addresses for the hosts in the system.
This file can of course be expanded to contain names for all the hosts, endpoints, and switches in the system, making debugging simpler.

```
; Nameserver records    Class  Type     Target
dns1.example.com.         IN     A      192.168.0.18
dns2.example.com.         IN     A      192.168.0.20
rds1.example.com.         IN     A      192.168.0.50
rds2.example.com.         IN     A      192.168.0.51
```

### Checking the DNS records for Multiple RDSs

```
# dig _nmos-register._tcp.example.com PTR
 <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> _nmos-register._tcp.example.com PTR
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48397
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;_nmos-register._tcp.example.com. IN	PTR

;; ANSWER SECTION:
_nmos-register._tcp.example.com. 3600 IN PTR	reg-api-2._nmos-register._tcp.example.com.
_nmos-register._tcp.example.com. 3600 IN PTR	reg-api-1._nmos-register._tcp.example.com.

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns2.example.com.
example.com.		3600	IN	NS	dns1.example.com.

;; ADDITIONAL SECTION:
dns1.example.com.	3600	IN	A	192.168.0.18
dns2.example.com.	3600	IN	A	192.168.0.20

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Tue Sep 06 07:29:22 UTC 2022
;; MSG SIZE  rcvd: 178
```

As can be seen the DNS server answers with two PTR records to this query.
Details on the priority and other metadata for each service instance can be checked as well.

First we query for the SRV record:

```
# dig reg-api-2._nmos-register._tcp.example.com SRV

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> reg-api-2._nmos-register._tcp.example.com SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5862
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 4

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;reg-api-2._nmos-register._tcp.example.com. IN SRV

;; ANSWER SECTION:
reg-api-2._nmos-register._tcp.example.com. 3600	IN SRV 20 10 80 rds2.example.com.

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns1.example.com.
example.com.		3600	IN	NS	dns2.example.com.

;; ADDITIONAL SECTION:
rds2.example.com.	3600	IN	A	192.168.0.51
dns1.example.com.	3600	IN	A	192.168.0.18
dns2.example.com.	3600	IN	A	192.168.0.20

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Tue Sep 06 07:32:09 UTC 2022
;; MSG SIZE  rcvd: 192
```

And now the TXT record for the same service:

```
# dig reg-api-2._nmos-register._tcp.example.com TXT

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.amzn2.5.2 <<>> reg-api-2._nmos-register._tcp.example.com TXT
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41920
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;reg-api-2._nmos-register._tcp.example.com. IN TXT

;; ANSWER SECTION:
reg-api-2._nmos-register._tcp.example.com. 3600	IN TXT "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=20" "api_auth=false"

;; AUTHORITY SECTION:
example.com.		3600	IN	NS	dns2.example.com.
example.com.		3600	IN	NS	dns1.example.com.

;; ADDITIONAL SECTION:
dns1.example.com.	3600	IN	A	192.168.0.18
dns2.example.com.	3600	IN	A	192.168.0.20

;; Query time: 0 msec
;; SERVER: 10.0.50.59#53(10.0.50.59)
;; WHEN: Tue Sep 13 09:59:12 UTC 2022
;; MSG SIZE  rcvd: 217
```

### Complete BIND9 Database File

Below is the complete BIND9 domain file described above.
Configuring your BIND9 domain with this file should allow you to verify your setup for an example.com domain.
To set up for a different domain, the changes to the file would be minimal.

```
; We begin by defining several required DNS parameters.

$TTL 3600
@       IN      SOA     dns1.example.com. admin.example.com. (
           20210713     ; Serial
               3600     ; Refresh
                600     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL

; Then we define our DNS servers for this zone.
; Endpoints should be configured with one or both of these DNS servers.

; DNS servers
example.com.        IN      NS      dns1.example.com.
example.com.        IN      NS      dns2.example.com.

; We add the following PTR records to indicate the server supports Service Discovery.
; Clients looking for DNS-SD support will query the server for these specific records (`b` is for "browse", `lb` is for "legacy browse".)

; These lines indicate to clients that this server supports DNS Service Discovery.
b._dns-sd._udp  IN      PTR     @
lb._dns-sd._udp IN      PTR     @

; Next we define the NMOS services provided by this server.

; These lines indicate to clients which NMOS service types this server advertises.
_services._dns-sd._udp  PTR     _nmos-register._tcp
_services._dns-sd._udp  PTR     _nmos-query._tcp

; There should be one `PTR` record for each instance of the service you wish to advertise.
; Here we have two instances of the Registration API service and one instance of the Query API service.

_nmos-register._tcp     PTR     reg-api-1._nmos-register._tcp
_nmos-register._tcp     PTR     reg-api-2._nmos-register._tcp
_nmos-query._tcp        PTR     qry-api-1._nmos-query._tcp

; Now we add `SRV` records that return the targets for the two registration service instances and the query service instance. 

; NMOS Registration and Query Services        Type   Priority  Weight   Port   Host 
; High Priority Registration Service
reg-api-1._nmos-register._tcp.example.com.     SRV     10        10      80    rds1.example.com.

; Lower Priority Registration Service
reg-api-2._nmos-register._tcp.example.com.     SRV     20        10      80    rds2.example.com.

; High Priority Query Service
qry-api-1._nmos-query._tcp.example.com.        SRV     10        10      80    rds1.example.com.

; We also provide `TXT` record for the both the primary and secondary service instances, with information relevant to the IS-04 specification.
; NMOS also includes priority information here as many DNS-SD clients ignore the priority and weight information in `SRV` records.

; High Priority Registration Service
reg-api-1._nmos-register._tcp.example.com.     TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=10" "api_auth=false"

; Lower Priority Registration Service
reg-api-2._nmos-register._tcp.example.com.     TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=20" "api_auth=false"

; High Priority Query Service
qry-api-1._nmos-query._tcp.example.com.        TXT     "api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=0" "api_auth=false"

; In all cases above the `SRV` records are identifying a port number of `80`.
; This would suit default HTTP access.
; For HTTPS access this would be set to `443`, and the `TXT` records would include `api_proto=https`.
; Again, this would be a question for the RDS vendor.

; Lastly we provide the IP addresses for the hosts in the system.
; This file can of course be expanded to contain names for all the hosts, endpoints, and switches in the system, making debugging simpler.

; Nameserver records    Class  Type     Target
dns1.example.com.         IN     A      192.168.0.18
dns2.example.com.         IN     A      192.168.0.20
rds1.example.com.         IN     A      192.168.0.50
rds2.example.com.         IN     A      192.168.0.51
```
