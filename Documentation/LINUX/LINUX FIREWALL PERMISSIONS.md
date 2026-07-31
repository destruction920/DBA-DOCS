#LINUX FIREWALL PERMISSIONS

# To Check OS Firewall status in Linux

systemctl status firewalld

# To check existing rules

firewall-cmd --list-all

#To allow ip address sequence and 5432 port in firewall settings execute
following command: firewall-cmd --permanent --zone=public
--add-rich-rule=' rule family="ipv4" source address="10.1.0.0/24" port
protocol="tcp" port="5432" accept'

# To Stop and Start firewall service

systemctl stop firewalld systemctl start firewalld

# To check existing rules (Newly added rules should be visible)

firewall-cmd --list-all

# To remove rich rule from firewall

firewall-cmd --permanent --zone=public --remove-rich-rule='rule
family="ipv4" source address="10.1.0.0/24" port protocol="tcp"
port="5432" accept'
