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

When we have a running dns server, we can try enumerate it, *especially* if it's running on TCP (Usually it means that zone transfers are enabled, dnssec is used and etc'). Let's use host (there are planty other tools such as dnsmap, dnsrecon etc') first to check for dns listing. Host is a dns lookup tool (you can read more about it in `man host`):
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
using the -l option, we want to see all the hosts in the domain using our dns server (10.10.10.123) -> This is called axfr, an option that allows dns server to copy the zone files from one server to another (_"AXFR is a mechanism for replicating DNS data across DNS servers."_). Basically what we can see here are the different sub-domains our domain has.
What that information tells us is there are a few more dns servers we can use. Let's add our dns server:
```
root@kali:~/Downloads# vim /etc/resolv.conf
root@kali:~/Downloads# ping admin.friendzoneportal.red
PING admin.friendzoneportal.red (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=8.40 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.088 ms

```
Great, now we can access the domain. But wait, it doesn't work. Entering "admin.frienzoneportal.red" doesn't give us anything. Well, here comes the tricky parts: our webserver is running virtual hosts (MORE here: https://en.wikipedia.org/wiki/Virtual_hosting under name based or https://httpd.apache.org/docs/trunk/vhosts/examples.html)
We can use BurpSuite with repeater to change the host we are sending the request to from 10.10.10.123 to admin.friendzoneportal.red with *HTTPS ENABLED*. Then, we get a different response:
```
HTTP/1.1 200 OK
Date: Tue, 14 May 2019 12:07:47 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Fri, 05 Oct 2018 23:11:41 GMT
ETag: "17b-5778364ed4075-gzip"
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Length: 379
Connection: close
Content-Type: text/html

<title>Admin Page</title>

<center><h2>Login and break some friendzones !</h2></center>

<center><h2>Spread the love !</h2></center>

<center>
<form name="login" method="POST" action="login.php">

<p>Username : <input type="text" name="username"></p>
<p>Password : <input type="password" name="password"></p>
<p><input type="submit" value="Login"></p>

</form>
</center>

<form>

```

From here, I was stuck, I don't really understand much in domain even after reading a lot about dns and dns records. I tried a different approach: Let's see what is SMB and what can we do with it. I read the following:
https://www.itprotoday.com/linux/linuxs-smbclient-command

Okay, so we have a smb server, let's try to get something interesting from it:
```
root@kali:~/Downloads# smbclient -L 10.10.10.123
Enter WORKGROUP\root's password: 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	Files           Disk      FriendZone Samba Server Files /etc/Files
	general         Disk      FriendZone Samba Server Files
	Development     Disk      FriendZone Samba Server Files
	IPC$            IPC       IPC Service (FriendZone server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            FRIENDZONE

```
Great, something to work with:
```
root@kali:~/Downloads# smbclient //10.10.10.123/Files
Enter WORKGROUP\root's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
root@kali:~/Downloads# smbclient //10.10.10.123/general
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jan 16 22:10:51 2019
  ..                                  D        0  Wed Jan 23 23:51:02 2019
  creds.txt                           N       57  Wed Oct 10 01:52:42 2018

		9221460 blocks of size 1024. 6373644 blocks available
smb: \> cat creds.txt
cat: command not found
smb: \> creds.txt
creds.txt: command not found
smb: \> echo creds.txt
echo <num> <data>
smb: \> file
file: command not found
smb: \> help
?              allinfo        altname        archive        backup         
blocksize      cancel         case_sensitive cd             chmod          
chown          close          del            deltree        dir            
du             echo           exit           get            getfacl        
geteas         hardlink       help           history        iosize         
lcd            link           lock           lowercase      ls             
l              mask           md             mget           mkdir          
more           mput           newer          notify         open           
posix          posix_encrypt  posix_open     posix_mkdir    posix_rmdir    
posix_unlink   posix_whoami   print          prompt         put            
pwd            q              queue          quit           readlink       
rd             recurse        reget          rename         reput          
rm             rmdir          showacls       setea          setmode        
scopy          stat           symlink        tar            tarmode        
timeout        translate      unlock         volume         vuid           
wdel           logon          listconnect    showconnect    tcon           
tdis           tid            utimes         logoff         ..             
!              
smb: \> open creds.txt
open file \creds.txt: for read/write fnum 1
smb: \> print creds.txt
NT_STATUS_ACCESS_DENIED opening remote file creds.txt
smb: \> ls
  .                                   D        0  Wed Jan 16 22:10:51 2019
  ..                                  D        0  Wed Jan 23 23:51:02 2019
  creds.txt                           N       57  Wed Oct 10 01:52:42 2018

		9221460 blocks of size 1024. 6373620 blocks available
smb: \> get creds.txt
getting file \creds.txt of size 57 as creds.txt (0.2 KiloBytes/sec) (average 0.2 KiloBytes/sec)

```
As you can see, files was not accessible with blank password. Then general contained a creds.txt file, I transfered it into my machine and we have a tail!
```
root@kali:~/Downloads# cat creds.txt
creds for the admin THING:

admin:****
```
Now we can access files or any other admin 'thing'. Let's get back to our login page. Trying to login will give a rabbit hole -> "ZZzzZzZ". I forgot to add the domain names to hosts file, so I added them:
(/etc/resolv.conf specifies nameservers in order of search preference.
/etc/hosts overrides all nameservers by mapping urls/shortnames to IPs)
```
127.0.0.1	localhost
127.0.1.1	kali
10.10.10.123 admin.friendzoneportal.red
10.10.10.123 files.friendzoneportal.red
10.10.10.123 imports.friendzoneportal.red
10.10.10.123 vpn.friendzoneportal.red
10.10.10.123 development.friendzoneportal.red
```
Now admin.friendzoneportal.red:*443* is accessible. But we still stuck! Let's try to run nikto:
```
root@kali:~/Downloads# nikto -host admin.friendzoneportal.red -p 443
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.10.123
+ Target Hostname:    admin.friendzoneportal.red
+ Target Port:        443
---------------------------------------------------------------------------
+ SSL Info:        Subject:  /C=JO/ST=CODERED/L=AMMAN/O=CODERED/OU=CODERED/CN=friendzone.red/emailAddress=haha@friendzone.red
                   Ciphers:  ECDHE-RSA-AES256-GCM-SHA384
                   Issuer:   /C=JO/ST=CODERED/L=AMMAN/O=CODERED/OU=CODERED/CN=friendzone.red/emailAddress=haha@friendzone.red

```
aha! the certificate is written on the email address haha@friendzone.red. Let's check https://friendzone.red (after adding it's address to the hosts file)
Okay, we get the following text:
```
Testing some functions !

I'am trying not to break things !
MDlPdDFEbWNMWjE1NTkzOTk4OTZJbUxYUDdWOUJx
```

