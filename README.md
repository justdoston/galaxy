# [Galaxy]

## Introduction

I made linux machine box I wanted to make it realistic as much as possible. This machine requieres critical thinking which is usefull for hackers

## Info for HTB

### Access

Passwords:

| User  | Password                            |
| ----- | ----------------------------------- |
| master | master6633 |
| doston | Ab4018618c |
| root  | root8618 |

### Key Processes

1) SSH server on 22 port
2) Apache2 server on 80 port which is vulnerable to http host header
3) There is custom copy.sh file in doston user's home directory
4) There is infodevs.zip file in /var/www/html/developers directory with password blink182 (which is crackable under 3 minute)
5) There is /usr/bin/php suid bit set which only master user can run

### Automation / Crons
There is no automation

### Firewall Rules

There is no firewall rule

### Docker

There is no docker

### Other

1)I made my own ip address to redirect to galaxy.htb in /etc/apache2/sites-avialabel/000-default.conf
2) I specifically made website unable to see .zip file
3) I specifically made doston user unable to see following directory and files: /etc/passwd, /etc/shells/, /home (but can access it's own directory), /var


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
