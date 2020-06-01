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

`/etc/unbound/unbound.conf.d/pi-hole.conf` - Use the conf file for settings

Start your local recursive server and test that it's operational:

```
sudo service unbound start
dig pi-hole.net @127.0.0.1 -p 5335
```
The first query may be quite slow, but subsequent queries, also to other domains under the same TLD, should be fairly quick.

### DNSSEC validation
You can test DNSSEC validation using
```
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
```

The first command should give a status report of SERVFAIL and no IP address. The second should give NOERROR plus an IP address.

Configure Pi-hole
Finally, configure Pi-hole to use your recursive DNS server by specifying 127.0.0.1#5335 as the Custom DNS (IPv4)

## Setting up a cron job to update root hints file
Create a monthly cron job with following settings from following file to update root hints file automatically

`/etc/cron.monthly/update-unbound-hints`

