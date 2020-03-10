#Install Guide for Setting up the Routinator RPKI Validator Software

1	Set the hostname
sudo nmtui-hostname
Hostname will be set

2	Set the interface and connect it

Please note that "Automatically connect" and "Available to all users" must be checked!

sudo nmtui-edit
Interface IP address will be set

3	Install required packages
sudo yum check-update
sudo yum upgrade -y
sudo yum install -y epel-release
sudo yum install -y vim wget curl net-tools lsof bash-completion yum-utils htop nginx httpd-tools tcpdump rust cargo rsync policycoreutils-python


4	Set the timezone to UTC
sudo timedatectl set-timezone UTC

5	Remove postfix, it's unneeded
sudo systemctl stop postfix
sudo systemctl disable postfix

6	Create SSL for nginx
sudo mkdir /etc/ssl/private
sudo chmod 700 /etc/ssl/private
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
Populate the relevant information to generate a self signed certificate
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

7	Add in this file to /etc/nginx/conf.d/ssl.conf and edit ssl.conf file to provide the IP of the host in the "server_name" field.
ssl.conf

8	Replace /etc/nginx/nginx.conf with this file:
nginx.conf

9	Add this file to the /etc/nginx directory:
proxy.conf

10	Set user/pass for web interface authentication
sudo htpasswd -c /etc/nginx/.htpasswd <username>

11	Start nginx and set it up so it starts at boot
sudo systemctl start nginx
sudo systemctl enable nginx

12	Add the user "routinator" and create the /opt/routinator directory and assign it to the 'routinator' user and group
sudo useradd routinator
sudo mkdir /opt/routinator
sudo chown routinator:routinator /opt/routinator

13	Sudo over to the routinator user
sudo su - routinator
"routinator" user will be created and logged in
14	Install the routinator software and add it to the $PATH for user routinator
cargo install routinator
vi /home/routinator/.bash_profile
Edit the PATH line to include "/home/routinator/.cargo/bin"
PATH=$PATH:$HOME/.local/bin:$HOME/bin:/home/routinator/.cargo/bin

15	Initialize the routinator software and accept the ARIN TAL and exit back to the user with sudo
/home/routinator/.cargo/bin/routinator -b /opt/routinator init -f --accept-arin-rpa
exit

16	Create a routinator systemd script from the template below.
Please note that you must populate the IPv4/IPv6 addresses!
Also, the IPv6 address needs to have brackets '[ ]' around it. For example:
/home/routinator/.cargo/bin/routinator -v -b /opt/routinator server --http 127.0.0.1:8080 --rtr 172.16.47.235:8323 --rtr [2600:6ce4:40:11::43]:8323

sudo vi /etc/systemd/system/routinator.service
[Unit]
Description=Routinator RPKI Validator and RTR Server
After=network.target
[Service]
Type=simple
User=routinator
Group=routinator
Restart=on-failure
RestartSec=90
ExecStart=/home/routinator/.cargo/bin/routinator -v -b /opt/routinator server --http 127.0.0.1:8080 --rtr <IPv4 IP>:8323 --rtr [<IPv6 IP>]:8323
TimeoutStartSec=0
[Install]
WantedBy=default.target

17	Reload the systemd daemon and set the routinator service to start at boot
sudo systemctl daemon-reload
sudo systemctl enable routinator.service
sudo systemctl start routinator.service

18	Configure SELinux to allow connections to localhost and to allow rsync to write to the /opt/routinator directory
sudo setsebool -P httpd_can_network_connect 1
sudo semanage permissive -a rsync_t

19	Set up the firewall to permit ssh, https and 8323 (RTR protocol)
sudo firewall-cmd --permanent --remove-service=ssh --zone=public
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" source address="<IPv4 management subnet>" service name=ssh accept'
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" source address="<IPv6 management subnet>" service name=ssh accept'
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" source address="<IPv4 management subnet>" service name=https accept'
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" source address="<IPv6 management subnet>" service name=https accept'
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" source address="<peering router IPv4 loopback subnet>" port port=8323 protocol=tcp accept'
sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" source address="<peering router IPv6 loopback subnet>" port port=8323 protocol=tcp accept'
sudo firewall-cmd --reload





