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

If desired, configure the next upstream DNS server - this is likely to be a corporate DNS service on the Internet the internet (`8.8.8.8`, or `1.1.1.1` for example) or might be in the local lab.

```
forwarders { 192.168.0.19; }; #IP of upstream nameserver(s)
recursion yes;
```

### Configure the Zone file

You will notice the line `file "/etc/bind/zones/db.gplab.com";` in the configuration information above.  This is a pointer to the zone file for the domain `gplab.com`.  In order for the DNS server to work properly, you will need to first create the directory `/etc/bind/zones/`, and then you will need to create the file `db.gplab.com`.  The file should look something like the following, which can be used for the domain `gplab.com`.

### Configure the hosts / TXT / SRV file

Now we create the file that contains nameserver, hosts, `SRV` and `TXT` records for the `gplab.com` domain, which needs to be located and called the same filename, as specified above in the zone config: `/etc/named/zones/db.gplab.local`

We define the global TTL (Time to live is seconds) for this zone (`gplab.com`), as 3600 seconds. `Serial` provides a timestamp that will be used when we synchronize a secondary DNS server later.

```
TTL 3600
@       IN      SOA     dns1.gplab.com. admin.gplab.com. (
           20210713     ; Serial
               3600     ; Refresh
                600     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```

Then we define our main and backup DNS server for this zone/domain. End-points should be configured with these DNS servers.

```
; DNS servers
        IN      NS      dns1.gplab.com.
        IN      NS      dns2.gplab.com.
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

There should be one `PTR` record for each instance of the service you wish to advertise. Here we have one Registration API and one Query API:

```
_nmos-register._tcp	PTR	reg-api-1._nmos-register._tcp
_nmos-query._tcp		PTR	qry-api-1._nmos-query._tcp
```

If more than one RDS server can be used, then two records can be provided, with a priority setting to enable end-points to make a preferred decision. In this case, the first is prioritized, with the `10` beating `20`.

The `TXT` records indicate additional metadata relevant to the IS-04 spec.


```
; NMOS RDS services
; Expected RDS
reg-api-1._nmos-register._tcp.gplab.com.     3600    IN SRV  10      10      80      rds1.gplab.com.

; Backup RDS
reg-api-1._nmos-register._tcp.gplab.com.     3600    IN SRV  20      10      80      rds2.gplab.com.

reg-api-1._nmos-register._tcp.gplab.com.	TXT	"api_ver=v1.0,v1.1,v1.2,v1.3" "api_proto=http" "pri=0" "api_auth=false"
```


Take advice from the RDS vendor about how to set the Priority (`10`) and Weight (`20`) for these `SRV` records. If active-active is available in the RDS servers, then these records can be used to provide load-balancing. In the case below, both records would be served with equal weight.


```
; RDS A
_nmos-register._tcp.gplab.com.     3600    IN SRV  10      20      80      rds1.gplab.com.

; RDS B
_nmos-register._tcp.gplab.com.     3600    IN SRV  10      20      80      rds2.gplab.com.
```


In all cases above the `SRV` records are identifying a port number of `80`. This would suit default HTTP access, with `443` needed for HTTPS - but again, this would be a question for the RDS vendor.

Lastly we provide the IP addresses for the hosts in the system. This file can of course be expanded to contain names for all the hosts, end-points, and switches in the system, making debugging simpler 


```
; Nameserver records
dns1.gplab.com.            IN      A       192.168.0.18
dns2.gplab.com.            IN      A       192.168.0.20
rds1.gplab.com.            IN      A       192.168.0.50
rds2.gplab.com.            IN      A       192.168.0.51
```



### Start the service

Once the files are in place, the service can be started and made permanent (will run after a reload of the server):


```
systemctl restart named
systemctl enable named
```

### Enable DNS through the linux firewall

Often, linux default will prevent DNS through the firewall, so you have issues with connectivity to the DNS server now running. The following will enable DNS through the CentOS 7 firewall, and make this permanent:


```
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```

## Testing

From a Linux box / Mac, testing can be carried out once the DNS config on that machine has been updated to point to the new DNS server.

There are a couple of tools that allow the DNS operation to be tested.

We can verify that the host names of the RDS servers are configured:


```
gparistacom:~ gparista.com$ nslookup rds1.gplab.com
Server:		192.168.0.18
Address:	192.168.0.18#53

Name:	rds1.gplab.com
Address: 192.168.0.50
```


We can see that the lookup was resolved by `192.168.0.18`, and resulted in the address for the `rds1` server being returned as `192.168.0.50`.

The `dig` tool provides a little more info:

```
gparistacom:~ gparista.com$ dig rds1.gplab.com

; <<>> DiG 9.10.6 <<>> rds1.gplab.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12178
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;rds1.gplab.com.		IN	A

;; ANSWER SECTION:
rds1.gplab.com.	3600	IN	A	192.168.0.50

;; AUTHORITY SECTION:
gplab.com.		3600	IN	NS	dns2.gplab.com.
gplab.com.		3600	IN	NS	dns1.gplab.com.

;; ADDITIONAL SECTION:
dns1.gplab.com.	3600	IN	A	192.168.0.18
dns2.gplab.com.	3600	IN	A	192.168.0.20

;; Query time: 43 msec
;; SERVER: 192.168.0.18#53(192.168.0.18)
;; WHEN: Tue Jul 13 19:58:50 IST 2021
;; MSG SIZE  rcvd: 134
```

### Checking the SRV records

We can also use `dig` to check the presence of the `_nmos._register_.tcp` record:


```
gparistacom:~ gparista.com$ dig _nmos-register._tcp.gplab.com SRV

; <<>> DiG 9.10.6 <<>> _nmos-register._tcp.gplab.com SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44496
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;_nmos-register._tcp.gplab.com. IN	SRV

;; ANSWER SECTION:
_nmos-register._tcp.gplab.com. 3600 IN SRV	10 10 80 rds1.gplab.com.
_nmos-register._tcp.gplab.com. 3600 IN SRV	20 10 80 rds2.gplab.com.

;; AUTHORITY SECTION:
gplab.com.		3600	IN	NS	dns2.gplab.com.
gplab.com.		3600	IN	NS	dns1.gplab.com.

;; ADDITIONAL SECTION:
rds1.gplab.com.	3600	IN	A	192.168.0.50
rds2.gplab.com.	3600	IN	A	192.168.0.51
dns1.gplab.com.	3600	IN	A	192.168.0.18
dns2.gplab.com.	3600	IN	A	192.168.0.20

;; Query time: 41 msec
;; SERVER: 192.168.0.18#53(192.168.0.18)
;; WHEN: Tue Jul 13 20:00:53 IST 2021
;; MSG SIZE  rcvd: 243
```

## Backup DNS (BIND) Server

To provide resilience, a secondary DNS server should be provisioned. end-points should have both DNS IP addresses configured, allowing them to use the backup DNS server, if the primary is no longer available.

BIND allows primary / secondary pairing, so that the zones and hosts configuration can be automatically updated on the secondary device, reducing the amount of duplication.
