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
```
root@kali:~/Documents# mdb-export backup.mdb auth_user
id,username,password,Status,last_login,RoleID,Remark
25,"admin","**",1,"08/23/18 21:11:47",26,
27,"engineer","**",1,"08/23/18 21:13:36",26,
28,"backup_admin","**",1,"08/23/18 21:14:02",26,
```
I thought these credentials would log me into the ftp server, but it seems that the ftp server is configured to anonymous logins only. I guess we are left only with the second file, so I downloaded it. It is encrypted with AES and the default linux unzip program doesn't support password extraction of this kind (the file was probably encrypted in Windows system). Therefore, one can download a tool called 7z. Kali comes with it preloaded. I tried engineer's password (the file was located in Engineer) and it worked:
```
ftp>get 'Access Control.zip'
root@kali:~/Documents# 7z x control.zip

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,1 CPU Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz (806E9),ASM,AES-NI)

Scanning the drive for archives:
1 file, 10870 bytes (11 KiB)

Extracting archive: control.zip
--
Path = control.zip
Type = zip
Physical Size = 10870

    
Enter password (will not be echoed):
Everything is Ok         

Size:       271360
Compressed: 10870
```
Now we have a .pst file, which is Microsoft Outlook email folder. Googling for it leads to a program named readpst:
```
root@kali:~/Documents# readpst 'Access Control.pst' 
Opening PST file and indexes...
Processing Folder "Deleted Items"
	"Access Control" - 2 items done, 0 items skipped.
```
Now we have the email file itself. It contains a username and a password. It's probably for the telnet. Let's proceeed:
```
C:\Users\security\Desktop>type user.txt
***
```
we also have a password for root.txt.
From here, I was stuck, so I took a few hints from hackthebox.eu. As it seems, there is a program called cmdkey.exe which stores all domain user credentials for use on a specific target machines. 



