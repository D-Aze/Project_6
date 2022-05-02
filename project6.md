# Documentation of Project 6: Web Solution with WordPress

## Step 1: Prepare a Web Server

1. Launch a Red Hat EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.

2. Attach all three volumes one by one to your Web Server EC2 instance

3. Open up the Linux terminal to begin configuration

4. Use lsblk command to inspect what block devices are attached to the server.
![volumes-attached](./images/volumes-attached.PNG)

5. Use gdisk utility to create a single partition on each of the 3 disks: `sudo gdisk /dev/xvdf`
![partition-1](./images/partition-1.PNG)
![partition-2](./images/partition-2.PNG)
![partition-3](./images/partition-3.PNG)

6. Use lsblk utility to view the newly configured partition on each of the 3 disks
![partitions-configured](./images/partitions-configured.PNG)

7. Install lvm2 package using `sudo yum install lvm2`. Run `sudo lvmdiskscan` command to check for available partitions
![lvm2](./images/lvm2.PNG)

8. Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

9. Verify that your Physical volume has been created successfully by running `sudo pvs`

10. Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg: `sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

11. Verify that your VG has been created successfully by running `sudo vgs`

12. Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs:

`sudo lvcreate -n apps-lv -L 14G webdata-vg`

`sudo lvcreate -n logs-lv -L 14G webdata-vg`

13. Verify that your Logical Volume has been created successfully by running `sudo lvs`

14. Verify the entire setup:

`sudo vgdisplay -v #view complete setup - VG, PV, and LV`

`sudo lsblk`

15. Use `mkfs.ext4` to format the logical volumes with ext4 filesystem

`sudo mkfs -t ext4 /dev/webdata-vg/apps-lv`

`sudo mkfs -t ext4 /dev/webdata-vg/logs-lv`

16. Create /var/www/html directory to store website files: `sudo mkdir -p /var/www/html`

17. Create /home/recovery/logs to store backup of log data: `sudo mkdir -p /home/recovery/logs`

18. Mount /var/www/html on apps-lv logical volume: `sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

19. Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system): `sudo rsync -av /var/log/. /home/recovery/logs/`

20. Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very
important): `sudo mount /dev/webdata-vg/logs-lv /var/log`

21. Restore log files back into /var/log directory: `sudo rsync -av /home/recovery/logs/. /var/log`

22. Update /etc/fstab file so that the mount configuration will persist after restart of the server:

a. The UUID of the device will be used to update the /etc/fstab file: `sudo blkid`
![uuid](./images/uuid.PNG)
b. Run `sudo vi /etc/fstab` and update as follows:
![uuid-updated](./images/uuid-updated.PNG)
c. Test the configuration and reload the daemon

 sudo mount -a
 sudo systemctl daemon-reload
d. Verify setup by running `df -h`
![setup-verified](./images/setup-verified.PNG)

## Step 2: Prepare a Database Server
Launch a second RedHat EC2 instance that will have a role – ‘DB Server’. Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.

## Step 3: Install WordPress on Your Web Server EC2
1. Update the repository: `sudo yum -y update`

2. Install wget, Apache and it’s dependencies: `sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

3. Start Apache: 

`sudo systemctl enable httpd`

`sudo systemctl start httpd`

![apache-started](./images/apache-started.PNG)


4. To install PHP and it’s depemdencies

`sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`

`sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`

`sudo yum module list php`

`sudo yum module reset php`

`sudo yum module enable php:remi-7.4`

`sudo yum install php php-opcache php-gd php-curl php-mysqlnd`

`sudo systemctl start php-fpm`

`sudo systemctl enable php-fpm`

`sudo setsebool -P httpd_execmem 1`

5. Restart Apache: `sudo systemctl restart httpd`

6. Download wordpress and copy wordpress to var/www/html

`mkdir wordpress`
 
`cd wordpress`

`sudo wget http://wordpress.org/latest.tar.gz`

`sudo tar xzvf latest.tar.gz`

`sudo rm -rf latest.tar.gz`

`sudo cp wp-config-sample.php wp-config.php`

`cp -R wordpress /var/www/html/`

Enter the wp-config.php file via `sudo vi wp-config.php` and edit the DB_NAME, DB_USER, DB_PASSWORD, and DB_HOST.
![wp-config](./images/wp-config.PNG)

7. Configure SELinux Policies

`sudo chown -R apache:apache /var/www/html/wordpress`

`sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R`

`sudo setsebool -P httpd_can_network_connect=1`

## Step 4: Install MySQL on your DB Server EC2
`sudo yum update`

`sudo yum install mysql-server`

Verify that the service is up and running by using `sudo systemctl status mysqld`. If it is not running, restart the service and enable it so it will be running even after reboot:

`sudo systemctl restart mysqld`

`sudo systemctl enable mysqld`

![mysql-running](./images/mysql-running.PNG)

## Step 5: Configure DB to Work with WordPress

`sudo mysql`

`CREATE DATABASE wordpress;`

`CREATE USER 'myuser'@'< Web-Server-Private-IP-Address >' IDENTIFIED BY 'mypass';`

`GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';`

`FLUSH PRIVILEGES;`

`SHOW DATABASES;`

`exit`

![database-created](./images/database-created.PNG)

## Step 6: Configure WordPress to connect to remote database

1. Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client

`sudo yum install mysql`

`sudo mysql -u admin -p -h <DB-Server-Private-IP-address>`
![connected](./images/connected.PNG)

2. Verify if you can successfully execute `SHOW DATABASES;` command and see a list of existing databases.

3. Change permissions and configuration so Apache could use WordPress

4. Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)
![red-hat](./images/red-hat.PNG)

5. Try to access from your browser the link to your WordPress: http://< Web-Server-Public-IP-Address >/wordpress/
![wordpress-1](./images/wordpress-1.PNG)
![wordpress-2](./images/wordpress-2.PNG)
![wordpress-3](./images/wordpress-3.PNG)

