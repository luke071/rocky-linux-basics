# Introduce to Rocky Linux 9
Rocky Linux 9 is built directly from RHEL 9 sources and has identical features, fixes and behavior.  


# 1. Introductory scenario
1.1 Check the existing existence of the host. Change the host to web-server . Then check if Apache is running and set static IP addresses on the interface.  
```bash
su -
hostnamectl status
hostnamectl set-hostname web-server
systemctl status httpd
nmcli connection show
sudo nmcli con modify "enp0s2" ipv4.method manual ipv4.addresses "192.168.0.50/24,192.168.0.51/24" ipv4.gateway 192.168.0.1 ipv4.dns 8.8.8.8
nmcli con up "enp0s2"
```
1.2 Create the main directory /www and subpages in this directory /wimbledon and /french-open
```bash
mkdir -p /www/wimbledon
mkdir -p /www/french-open
```
1.3 Create index.html files in the main directory and subpages.
```bash
echo "<h1>Wimbledon</h1>" | tee /www/wimbledon/index.html
echo "<h1>French Open</h1>" | tee /www/french-open/index.html
echo "<h1>Home page</h1>" | tee /www/index.html
```
1.4 Set permissions.
```bash
chown -R apache:apache /www
chmod -R 755 /www
```
1.5 Configure Apache VirtualHosts on port 100.  
Creating a custom-sites.conf file with an entry.  
```bash
nano /etc/httpd/conf.d/custom-sites.conf
```
```bash
<VirtualHost 192.168.0.50:100>
    DocumentRoot "/www/wimbledon"
    <Directory "/www/wimbledon">
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost 192.168.0.51:100>
    DocumentRoot "/www/french-open"
    <Directory "/www/french-open">
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:80>
    DocumentRoot "/www"
    ServerName 192.168.0.50
    <Directory "/www">
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```
1.6 Set up Apache on port 100.   
```bash
nano /etc/httpd/conf/httpd.conf
```
Added Listen 100 listening. 
Listen 80
Listen 100 
1.7 Allow access through port 100 in firewalld.  
```bash
firewall-cmd --permanent --add-port=100/tcp
firewall-cmd --reload  
``` 
1.8 Configure SELinux to allow Apache to use non-standard ports.  
Adding TCP port 100 to the SELinux context as allowed for the HTTP server.  
```bash
semanage port -a -t http_port_t -p tcp 100
```
Assign the SELinux label httpd_sys_content_t to the /www directory and all its subdirectories and files.  
```bash
semanage fcontext -a -t httpd_sys_content_t "/www(/.*)?"
```
Restore SELinux contexts to files in /www, according to previously assigned rules.  
```bash
restorecon -Rv /www
```
Restarting Apache.  
```bash
systemctl restart httpd
```

Browser accessibility test.  

http://192.168.0.50:100 - will show the French Open website  
![alt text](./assets/1.png)  
http://192.168.0.51:100 - will show the Wimbledon website  
![alt text](./assets/2.png)  
http://192.168.0.50 - show home page  
http://192.168.0.51 - show home page   
![alt text](./assets/3.png)  
1.9 Configure access to the website located in the user's directory via the userdir module.  
Enabling the UserDir module by editing the /etc/httpd/conf.d/userdir.conf file.  
```bash
<IfModule mod_userdir.c>
    UserDir public_html
    UserDir enabled student
</IfModule>

<Directory "/home/*/public_html">
    AllowOverride None
    Require all granted
</Directory>
```
Execution of the command  
```bash
httpd -M | grep userdir
```
should return userdir module (shared). Allow Apache to access user directories.  
```bash
setsebool -P httpd_enable_homedirs on
```
Set Linux context for ~/public_html directory.  
```bash
semanage fcontext -a -t httpd_user_content_t "/home/student/public_html(/.*)?"
restorecon -Rv /home/student/public_html
```
Configuring the user's home directory.  
```bash
mkdir ~/public_html
echo "<h1>User directory</h1>" > ~/public_html/index.html
chmod 711 ~
chmod 711 ~
```
1.10 Test access to websites.  
Browser accessibility test of the user's home directory. 
http://192.168.0.50/~student - will show the page in the student user directory  
![alt text](./assets/4.png)  

1.11 After adding a 2 GB disk to the machine, create two partitions of 1 GB each with automatic punting after reboot.
After adding a new hard drive in Virtualbox (with the Pre-allocate Full Size option).  
```bash
lsblk
```
![alt text](./assets/disc-1.png)  
```bash
fdisk /dev/sdb
```
First partition  
n - new partition   
p - primary type  
1 - partition number   
first sector - press enter   
+1G
  
First partition  
n - new partition   
p - primary type  
2 - partition number   
first sector - press enter   
+1024MB
  
w - save changes  

Once the partitions are created, they must be formatted using the appropriate file system. Then create folders disc1 and disc2.   

```bash
mkfs.ext4 /dev/sdb1
mkfs.ext4 /dev/sdb2
mkdir /mnt/disc1 /mnt/disc2
```
Check UUID.
```bash
blkid
```
Add in file fstab:  
UUID="XXX-XXX-XXX" /mtn/disc1 ext4 defaults 0 2  
UUID="XXX-XXX-XXX" /mtn/disc2 ext4 defaults 0 2  
```bash
nano /etc/fstab
```
Mount and restart the system.    
```bash
mount -a
restart
```
Checking mounting after restart.  
```bash
df - h
```
Test:  
![alt text](./assets/disc-2.png)  

1.12 Check the technical condition of this disk with smartctl.
Installing the smartmontools hard drive monitoring and diagnostics toolkit.  
```bash
dnf install smartmontools
```
Performing a short test.  
```bash
smartctl -t short /dev/sdb
```
1.13 Create the Guests group and the Guest-1 user with a home directory for which the maximum number of days between password changes is 5 days. 
```bash
groupadd Guests
useradd -m -G Guests -e "" -s /bin/bash Guest-1
passwd Guest-1
chage -M 5 Guest-1
```
1.14 Assign the Guest-1 account to the Guests group. In the Guest-1 home directory, create a hidden documents directory and give it full rights for the Guests group.  
```bash
sudo -u Guest-1 mkdir /home/Guest-1/.documents
sudo chmod 770 /home/Guest-1/.documents
sudo chown Guest-1:Guests /home/Guest-1/.documents
```
1.15 Create files file-1, file-2 and file-3 in Guest-1's home directory. Create a compressed .tar.gz archive from these files. Then unpack the archive to a folder in home directory /Guest-1/unpacked .
```bash
sudo -u Guest-1 touch /home/Guest-1/file-{1..3}
sudo -u Guest-1 tar -czvf /home/Guest-1/files.tar.gz -C /home/Guest-1 file-1 file-2 file-3
mkdir /home/Guest-1/unpacked
tar -xvzf /home/Guest-1/files.tar.gz -C /home/Guest-1/unpacked
```
1.16 Execute the command alias top -b -n 1 | head -n 17.
```bash
alias toptop='top -b -n 1 | head -n 17'
```
Alias use.
```bash
toptop
```
1.17 Check the complete information about the system kernel.
```bash
uname -a
```
1.18 Check the version and name of the distribution.
```bash
cat /etc/os-release
```
1.19 Check system uptime and load.
```bash
uptime
```
1.20 Check what uses the most RAM.
```bash
ps aux --sort=-%mem | head
```
1.21 Check the parameters of the motherboard, processor, RAM.
```bash
dnf install dmidecode pciutils -y
dmidecode | less
lscpu
free -h
lspci
```
1.22 Check the active connection and listening ports.
```bash
ss -tulnp
```
1.23 View system logs.
```bash
journalctl -xe | less
```
1.24 View kernel messages.
```bash
dmesg | less
```
1.25 Disable ping (ICMP echo).
Permanently disable ping.
```bash
firewall-cmd --add-icmp-block=echo-request --permanent
firewall-cmd --reload
```

Re-enable ping.
```bash
firewall-cmd --remove-icmp-block=echo-request --permanent
firewall-cmd --reload
```


# 2. How to secure Rocky Linux 9
## Logging failed login attempts to a text file.  
```bash
su -
grep "Failed password" /var/log/secure > failed-password.txt 
```
### Changing the SSH port  
In any text editor, open the file /etc/ssh/sshd_config.
```bash
nano /etc/ssh/sshd_config
```
Find the line with #Port 22 remove # and change it to example Port 14355 (recommended ports from 1025 to 65535).
Check the current SELinux mode:
```bash
sestatus
```
Be sure to add port 14355 to SELinux. Replace &lt;type&gt;, &lt;protocol&gt;, and &lt;port_number&gt; in command semanage port -a -t &lt;type&gt; -p &lt;protocol&gt; &lt;port_number&gt;.
```bash
semanage port -a -t ssh_port_t -p tcp 14355
```
Change the Firewall
```
sudo firewall-cmd --permanent --add-port=14355/tcp
sudo firewall-cmd --reload
```
Restarting the ssh service.
```bash
systemctl restart sshd
```
### Blocking root login via SSH
In any text editor, open the file /etc/ssh/sshd_config.
```bash
nano /etc/ssh/sshd_config
```
Find the line shown below:  
PermitRootLogin yes  
Change the value from yes to no. Save and close the file. Then restart the SSH service.  
```bash
systemctl restart sshd
```
### Fail2Ban
Fail2Ban is available in the EPEL repository.  
```bash
dnf install epel-release -y
```
Installation, switching on, start-up fail2ban.  
```bash
dnf install fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```
According to domnetacja you should not modify the jail.conf configuration file directly, so a copy of this file will be created.  
```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Ignoring specific ip address and range. Open /etc/fail2ban/jail.local and add after the default loopback address the example ignored IP address and ignored network address.  

ignoreip = 127.0.0.1/8 ::1 192.168.0.100 172.17.0.0/16  

Sshd service blocking policy in fail2ban. Open file /etc/fail2ban/jail.local and add sample settings after 3 failed login attempts within 3600 s (24 h) setting the lock to 10800 (72 h):  

[sshd]  
enabled = true  
port = ssh  
filter = sshd  
logpath = /var/log/auth.log  
maxretry = 3  
findtime = 3600  
bantime = 10800  

Fail2ban configuration reload.  

```bash
fail2ban-client reload
```

# 3. Apache installation

1. Update system then install HTTP packages.  
```bash
su -
dnf update -y
dnf install httpd -y
```
2. Start the service immediately and configure the service to start after each system boot.  
```bash
systemctl start httpd
systemctl enable httpd
```
3. Status check.  
```bash
systemctl status httpd
```
4. Allow HTTP and HTTPS traffic in the firewall.  
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

# 4. Cockpit installation

1. After updating the system, install Cockpit.  
```bash
su -
dnf update -y
dnf install -y cockpit
```
2. Turning on and running Cockpit.  
```bash
systemctl enable --now cockpit.socket
```
3. Open the port in firewalld.  
```bash
firewall-cmd --permanent --add-service=cockpit
firewall-cmd --reload
```
4. Launching Cockpit in the browser.  
https://<server_ip_address>:9090  
![alt text](./assets/cockpit.png)  