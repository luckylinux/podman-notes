# DNS Setup
# Introduction
These Notes have been written as part of the Podman Pasta IPv6 + snid NAT45 IPv4 <-> IPv6 Tutorial.

Since the DNS Server Setup is quite a separate Topic from what that Podman Tutorial was meant to address, it has been decided to move the DNS Setup to a different Location.

## SNID Requirements
Read about [How SNID Works](https://github.com/AGWA/snid?tab=readme-ov-file#dns-lookup-behavior).

In Order for Direct IPv6 Connectivity to work, an appropriate AAAA (IPv6) Record Must be present for the Application Hostname:
```
whoami          IN      AAAA    2001:db8:0000:0001:0000:0000:0001:0001
```

For IPv4 Connectivity, it is the IPv4 Address of the `snid` Host that must be Entered.
In this Tutorial, since `snid` is assumed to be running on the Podman Host itself, this simply means creating an A Record for the Podman Host itself:
```
whoami          IN      A    198.51.100.10
```

Therefore, at the end of the Day, it is required that **BOTH** DNS Records are present for `snid` to work correctly, so that an Incoming IPv4 Request looks like this:
```
Remote Client <---> IPv4 Connection to whoami.MYDOMAIN.TLD <---> Contact Server on 198.51.100.10 <---> SNID Server Listening on 198.51.100.10 Port 443 (TLS) <--> DNS Lookup SNI Record (whoami.MYDOMAIN.TLD) <---> Find Target Server IPv6 Address 2001:db8:0000:0001:0000:0000:0001:0001 <--> Perform NAT46 <--> IPv6 Connection to 2001:db8:0000:0001:0000:0000:0001:0001
```

# Potential Issues
## DNSSec and DNS over TLS
Issues can arise due to DNSSec and/or DNS over TLS Configuration related Issues. 

Try to **TEMPORARILY** disable those e.g. in `/etc/systemd/resolved.conf`:

Flush caches with:
```
resolvectl flush-caches
```

Restart Service:
```
systemctl restart systemd-resolved
```

Check if the Issue disappears.

If the Issue disappears, you might need to proceed and setup a separate Authoritative Nameserver to which BIND (and potentially your Home Devices) can talk to.

This is a better Solution compared to turning off DNSSec / DNS over TLS entirely !

## Cloudflare DNS Proxy
In case of DNS with Cloudflare DNS Proxy (or similar by another Provider) furthermore, `snid` probably will NOT work, since the IPv6 Addresses on Cloudflare DNS Servers will NOT match the CIDR Backend for `snid`. 

Furthermore, I am NOT sure how these in-out-in Connections Loops will be handled by both `snid` and the Firewall.

NAT Reflection on the Firewall (e.g. OPNSense) to some extent might help regarding IPv4 NAT.

The fact that the DNS Records however do NOT point to the IP Address of the Server will still most likely be an Issue.

# Setup
As stated previously, I am NOT 100% sure that this is required.

However, if you want to perform Host Overrides on DNS Level and still want to use DNSSec, you MUST have a Nameserver that is correctly registered with DNSSec in your Registrar Control Panel.

These are just some Notes. Detailed Setup Instructions are outside of the Purpose of this Tutorial.

For a full-fledged BIND Configuration, please refer to the relevant Tutorials, which are abundant on the Internet.

## Environment
I ended up running an LXC Container in the DMZ Zone, i.e. in the same IPv4 (and IPv6) Subnet as the Podman Host.

This LXC Container will run `bind` / `named` as an Authoritative Name Server for the configured Domain `DOMAIN="MYDOMAIN.TLD"`.

You might want to do a fully-isolated Virtual Machine (e.g. KVM, ESXI, ...) or you might manage to setup using a Podman Container instead.

BIND Local DNS Server:
- Public IPv4: `198.51.100.10`
- Private IPv4: `172.16.1.3`
- Public+Private IPv6: `2001:db8:0000:0001:0000:0000:0000:0003/128`

## Enable DNSSec
To enable DNSSec you can however follow this [Excellent Tutorial](https://www.talkdns.com/articles/a-beginners-guide-to-dnssec-with-bind-9/).

Remember to register the `DS` Key in your Registrar Control Panel for your Domain !

## Enable DNS over TLS
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

## DNS View and Zone
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

## Configure DNS on the LXC Container Host
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

Also create `/etc/.pve-ignore.resolv.conf` to make sure that Proxmox VE does NOT override `/etc/resolv.conf` (which points to `/run/systemd/resolve/stub-resolv.conf` if `systemd-resolved` is enabled and "in charge"):
```
# https://forum.proxmox.com/threads/how-to-disable-auto-updates-of-etc-hostsname-etc-resolv-conf-maybe-others.76186/
# https://pve.proxmox.com/wiki/Linux_Container#_guest_operating_system_configuration
# DO NOT let Proxmox VE Update /etc/resolv.conf automatically
```

## Configure DNS on the Podman Host

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

# Troubleshooting
From a Remote Client (NOT located withing the same LAN, try to use 4G/LTE Connectivity otherwise) test that Connectivity is working.

It is reccomended to test against something like `traefik/whoami` Application, which can display many Parameters, including HTTP and especially X-Forwarded-For Headers.

This Section assumes:
```
HOSTNAME="whoami.MYDOMAIN.TLD"
```

In order to follow the Redirects, `curl` MUST be invoked with the `-L` Argument.

Thee `-vvv` Argument is only to Obtain more Verbose Output (which can be Useful for Debugging) and can be omitted if everything is working normally.

## IPv6 Testing
IPv6 HTTPS Test (do NOT follow redirects) should yield a `200` Status Response (`OK`):
```
curl -vvv -6 https://${HOSTNAME}
```

IPv6 HTTP Test (do NOT follow redirects) should yield a `308` Status Response (`Permanent Redirect`):
```
curl -vvv -6 http://${HOSTNAME}
```

IPv6 HTTP Test (follow redirects) should yield a `200` Status Response (`OK`):
```
curl -vvv -6 -L http://${HOSTNAME}
```

## IPv4 Testing
IPv4 HTTPS Test (do NOT follow redirects) should yield a `200` Status Response (`OK`):
```
curl -vvv -4 https://${HOSTNAME}
```

IPv4 HTTP Test (do NOT follow redirects) should yield a `308` Status Response (`Permanent Redirect`):
```
curl -vvv -4 https://${HOSTNAME}
```

IPv4 HTTP Test (follow redirects) should yield a `200` Status Response (`OK`):
```
curl -vvv -4 -L http://${HOSTNAME}
```

In case of Issues in the IPv4 Test, check the `snid` Service Status for Clues:
```
systemctl status snid.service
journalctl -xeu snid.service
```