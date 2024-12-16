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

[

Provide an in-depth explanation of the steps it takes to complete the box from start to finish. Divide your walkthrough into the below sections and sub-sections and include images to guide the user through the exploitation. 

Please also include screenshots of any visual elements (like websites) that are part of the submission. Our review team is not only evaluating the technical path, but the realism and story of the box.

Show **all** specific commands using markdown's triple-backticks (```` ```bash ````) such that the reader can copy/paste them, and also show the commands' output through images or markdown code blocks (```` ``` ````). 

**A reader should be able to solve the box entirely by copying and pasting the commands you provide.**

]

# Enumeration

[Describe the steps that describe the box's enumeration. Typically, this includes a sub-heading for the Nmap scan, HTTP/web enumeration, etc.]

# Foothold

[Describe the steps for obtaining an initial foothold (shell/command execution) on the target.]

# Lateral Movement (optional)

[Describe the steps for lateral movement. This can include Docker breakouts / escape-to-host, etc.]

# Privilege Escalation

[Describe the steps to obtaining root/administrator privileges on the box.]
