# DNS Setup
In Order for Direct IPv6 Connectivity to work, an appropriate AAAA (IPv6) Record Must be present for the Application Hostname:
```
whoami          IN      AAAA    2001:db8:0000:0001:0000:0000:0001:0001
```

For IPv4 Connectivity, it is the IPv4 Address of the `snid` Host that must be Entered.
In this Tutorial, since `snid` is assumed to be running on the Podman Host itself, this simply means creating an A Record for the Podman Host itself:
```
whoami          IN      A    198.51.100.10
```

> **Warning**  
> 
> Issues can arise due to DNSSec and/or DNS over TLS Configuration related Issues. Try to disable those, flush caches with `resolvectl flush-caches` and issue `systemctl restart systemd-resolved` to see if the Issue disappears.

> **Note**  
> 
> I am NOT 100% sure that this is required. However, if you want to perform Host Overrides on DNS Level and still want to use DNSSec, you MUST have a Nameserver that is correctly registered with DNSSec in your Registrar Control Panel.

In case of DNS with Cloudflare DNS Proxy (or similar by another Provider) furthermore, `snid` probably will NOT work, since the IPv6 Addresses on Cloudflare DNS Servers will NOT match the CIDR Backend for `snid` at least. Furthermore, I am NOT sure how these in-out-in Connections Loops will be handled by both `snid` and the Firewall.

In order to mitigate against this, I ended up running an LXC Container in the DMZ Zone, i.e. in the same IPv4 (and IPv6) Subnet as the Podman Host, which acts as an Authoritative Name Server.

These are just some Notes. For a full-fledged BIND Configuration, please refer to the relevant Tutorials.

BIND Local DNS Server:
- Public IPv4: `198.51.100.10`
- Private IPv4: `172.16.1.3`
- Public+Private IPv6: `2001:db8:0000:0001:0000:0000:0000:0003/128`

As explained previously, detailed Setup Instructions are outside of the Purpose of this Tutorial. To enable DNSSec you can however follow this [Excellent Tutorial](https://www.talkdns.com/articles/a-beginners-guide-to-dnssec-with-bind-9/).

DNS Over TLS can be Enabled in `/etc/bind/named.conf` by adding a few Lines and Certificates (I manage them using `certbot`, your Mileage might vary):
```
// TLS Certificates and Key
tls letsencrypt-tls {
        cert-file "/etc/letsencrypt/fullchain.pem";
        key-file "/etc/letsencrypt/privkey.pem";
};

...

// Unencrypted
listen-on-v6 port 53 { ::1; };
listen-on port 53 { localhost; 127.0.0.1; 172.16.1.3; any; };

// DNS over TLS
listen-on-v6 port 853 tls letsencrypt-tls { ::1; };
listen-on port 853 tls letsencrypt-tls { localhost; 127.0.0.1; 172.16.1.3; any; };
```

Set BIND to be Authoritative Nameserver for the Domain, while forwarding everything else to Hosting Provider DNS Servers:
```
view "internal" {
        match-clients { 172.16.1.0/24; localhost; 127.0.0.1; };
        recursion yes;

        allow-recursion {
                /* Only trusted addresses are allowed to use recursion. */
                trusted;
        };

        zone "MYDOMAIN.TLD" IN {
                type master;
                file "/etc/bind/pri/MYDOMAIN.TLD.internal";
                notify yes;

                // Instruct BIND to sign the Zone File
                dnssec-policy default;                   # this enables BIND's fully automated key and signing policy
                                                         #  - (ISC's recommended way to manage DNSSEC)
                key-directory "/etc/bind/keys";          # this sets the directory in which this zone's DNSSEC keys will be stored
                inline-signing yes;                      # this allows BIND to transparently update our signed zone file
                                                         # whenever we change the unsigned file
        };


        zone "." {
                type forward;
                forward only;
                forwarders {
			                  // Use Google Public Recursive Name Servers
                        8.8.8.8;
			                  8.8.4.4;
			                  2001:4860:4860::8888;
			                  2001:4860:4860::8844;
                };
        };


        // Default Zones
        include "/etc/bind/named.conf.default-zones";
};
```

Then it's pretty much a standard BIND Zone Configuration File:
```
; MYDOMAIN.TLD.

$ORIGIN MYDOMAIN.TLD.
$TTL    5m
@       IN      SOA	ns1.MYDOMAIN.TLD. ns2.MYDOMAIN.TLD. (
	                YYYYMMDD01				; Serial (10 digits)
                        604800					; Refresh
                        3600					; Retry
			;86400					; Retry
                        2419200					; Expire
                        604800)					; Negative Cache TTL
;

; Authoritative nameservers
			IN              NS              ns1
			IN	        NS	        ns2

; Nameservers
ns1                     IN              A               198.51.100.10
ns1			IN	        AAAA	        2001:db8:0000:0001:0000:0000:0000:0003

; For now NS2 is just the same as NS1
ns2                     IN              A               198.51.100.10
ns2			IN	        AAAA	        2001:db8:0000:0001:0000:0000:0000:0003

; ##########################################################################################

; Defaults 
                        IN              A               198.51.100.10

; ##########################################################################################

; Podman Host

podmanhost01		IN	        A	        198.51.100.10
podmanhost01        	IN	        AAAA	        2001:db8:0000:0001:0000:0000:0000:0100

; ##########################################################################################

; Applications

; ...
; ...

whoami			IN	        A               198.51.100.10
whoami			IN              AAAA            2001:db8:0000:0001:0000:0000:0001:0001

```

Add to `/etc/systemd/resolved.conf` on the DNS Server Host:
```
[Resolve]
# Use local BIND Name Server, which will forward to Hetzner Recursive Name Servers for external (non-managed) Domains
DNS=127.0.0.1
Domains=MYDOMAIN.TLD
DNSSEC=yes
DNSOverTLS=opportunistic

# Set to NO since there is already a DNS Server on the System (BIND/NAMED)
DNSStubListener=no
```

Add to `/etc/systemd/resolved.conf` on the Podman Host:
```
[Resolve]
DNS=172.16.1.3 2001:db8:0000:0001:0000:0000:0000:0003
#FallbackDNS=
Domains=MYDOMAIN.TLD
DNSSEC=yes
DNSOverTLS=opportunistic
```

On both Hosts, run:
```
systemctl restart systemd-resolved
```

In this way, the correct IPv4 <-> IPv6 Translation should be performed and the correct Application should now be able to be reacheable.