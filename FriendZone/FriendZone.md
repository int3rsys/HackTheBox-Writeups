We start with a regular port scan which reveals:
```
Port		State

21/tcp		open
22/tcp		open
53/tcp		open
80/tcp		open
139/tcp		open
```
If we enter 10.10.10.123 in our browser we get a photo with a domain name: `Email us at: info@friendzoneportal.red`
Next obvious thing was to try to look for known exploits for the services (httpd, apache, ISC BIND for the DNS server, samba server) but it wasn't successful.

When we have a running dns server, we can try enumerate it. Let's use host (there are planty other tools such as dnsmap, dnsrecon etc') first to check for dns listing. Host is a dns lookup tool (you can read more about it in `man host`):
```
root@kali:~/Downloads# host -l friendzoneportal.red 10.10.10.123
Using domain server:
Name: 10.10.10.123
Address: 10.10.10.123#53
Aliases: 

friendzoneportal.red has IPv6 address ::1
friendzoneportal.red name server localhost.
friendzoneportal.red has address 127.0.0.1
admin.friendzoneportal.red has address 127.0.0.1
files.friendzoneportal.red has address 127.0.0.1
imports.friendzoneportal.red has address 127.0.0.1
vpn.friendzoneportal.red has address 127.0.0.1
```
using the -l option, we want to see all the hosts in the domain using our dns server (10.10.10.123) -> This is called axfr, an option that allows dns server to copy the dns tables from one server to another ("AXFR is a mechanism for replicating DNS data across DNS servers."). Basically what we can see here are the different sub-domains our domain has.




