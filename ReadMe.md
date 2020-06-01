# PiHole Setup
Setting up a recursive PiHole setup at home

## Installing Unbound
(Using the guide from https://docs.pi-hole.net/guides/unbound/)
We will use unbound, a secure open-source recursive DNS server primarily developed by NLnet Labs, VeriSign Inc., Nominet, and Kirei. The first thing you need to do is to install the recursive DNS resolver:
`sudo apt install unbound`
Important: Download the current root hints file (the list of primary root servers which are serving the domain "." - the root domain). ```
```
wget -O root.hints https://www.internic.net/domain/named.root
sudo mv root.hints /var/lib/unbound/
```
### Setting up DNSSec
Using the guide from (https://nlnetlabs.nl/documentation/unbound/howto-anchor).

We will need following initial files to setup unbound-anchor:

* `/var/lib/unbound/root.key` - The root anchor file, updated with 5011 tracking, and read and written to. The file is created if it does not exist.

* `/etc/unbound/icannbundle.pem` - The trusted self-signed certificate that is used to verify the downloaded DNSSEC root trust anchor. You can update it by fetching it from https://data.iana.org/root-anchors/icannbundle.pem (and validate it). If the file does not exist or is empty, a builtin version is used.
* https://data.iana.org/root-anchors/root-anchors.xml - Source for the root key information.
* https://data.iana.org/root-anchors/root-anchors.p7s - Signature on the root key information.

After getting the certificate file run the following command:

`unbound-anchor -a /var/lib/unbound/root.anchor -c /etc/unbound/icannbundle.pem`

This will update the files.

### Configure unbound
Highlights:
* Listen only for queries from the local Pi-hole installation (on port 5335)
* Listen for both UDP and TCP requests
* Verify DNSSEC signatures, discarding BOGUS domains
* Apply a few security and privacy tricks
```
/etc/unbound/unbound.conf.d/pi-hole.conf 
```
```
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # Suggested by the unbound man page to reduce fragmentation reassembly problems
    edns-buffer-size: 1472

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```
Start your local recursive server and test that it's operational:

```
sudo service unbound start
dig pi-hole.net @127.0.0.1 -p 5335
```
The first query may be quite slow, but subsequent queries, also to other domains under the same TLD, should be fairly quick.

### Test validation
You can test DNSSEC validation using
```
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
```

The first command should give a status report of SERVFAIL and no IP address. The second should give NOERROR plus an IP address.

Configure Pi-hole
Finally, configure Pi-hole to use your recursive DNS server by specifying 127.0.0.1#5335 as the Custom DNS (IPv4)

## Setting up a cron job to update root hints file
Create a monthly cron job with following settings to update root hints file automatically
```
/etc/cron.monthly/update-unbound-hints
```
```
#!/bin/bash
wget -q https://www.internic.net/domain/named.root -O /tmp/root.hints
if grep -q ROOT-SERVERS /tmp/root.hints ;then
  mv -f /tmp/root.hints /etc/unbound/root.hints && chmod a+r /var/lib/unbound/root.hints
  unbound-anchor -a /var/lib/unbound/root.anchor -c /etc/unbound/icannbundle.pem
  unbound-control reload
```

