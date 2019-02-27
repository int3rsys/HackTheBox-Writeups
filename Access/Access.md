This is the first writeup in the series.
After accessing to http://10.10.10.98/ we get a picture of servers and a title named "LON-MC6". 
Searching in the internet for the name results in a server name. Basically, we have nothing else in the webiste.
The source code is empty.



Okay, so after searching on the internet, I've figured that the default download way for my ftp client ('ftp' in debian) is ASCII where is the targeted file should be downloaded as binary. 
```ftp> lcd /root/Documents
Local directory now /root/Documents  //Desired destination on local machine
ftp> binary
---> TYPE I
200 Type set to I.
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
---> PORT 10,10,13,77,235,177
200 PORT command successful.
---> RETR backup.mdb
125 Data connection already open; Transfer starting.
226 Transfer complete.
5652480 bytes received in 13.63 secs (405.0628 kB/s)
```
Now in order to handle mdb files, we need to install mdbtools. Kali linux has it preinstall. Let's dive into the file:
```
mdb-schema backup.mdb
.
.
.
(alot of tables)
```
We have alot of data. Obviously we want to have username and password for the ftp server. Let's find the password field:
```
root@kali:~/Documents# mdb-schema backup.mdb | awk -vRS='' '/password/'     //looking for password field.
CREATE TABLE [auth_user]
 (
	[id]			Long Integer, 
	[username]			Text (100), 
	[password]			Text (100), 
	[Status]			Long Integer, 
	[last_login]			DateTime, 
	[RoleID]			Long Integer, 
	[Remark]			Memo/Hyperlink (255)
);
```
Now we know the interesting table name - auth_users.
Let's export it and see what we have:






