---
layout: default
title: How to install BookStack in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 2
---

# How to install BookStack
{: .no_toc }
Wanted to document and gather all of my findings in a central place.


## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---
## Getting started


## Requirements
* FreeNAS 11.2

## Install

Check the FreeBSD version.
```bash
uname -ro
FreeBSD 11.2-STABLE
```
Ensure that your FreeBSD system is up to date.
```bash
root@BookStack:~ # pkg update && pkg upgrade -y
```
Install the necessary packages.
```bash
pkg install -y sudo nano unzip curl wget bash socat git
```
Create a new user account with your preferred username, we will use bookstack
```bash
root@BookStack:~ # adduser
Username: bookstack
Full name: Book Stack
Uid (Leave empty for default): 
Login group [bookstack]: 
Login group is bookstack. Invite bookstack into other groups? []: wheel
Login class [default]: 
Shell (sh csh tcsh bash rbash git-shell nologin) [sh]: 
Home directory [/home/bookstack]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: yes
Use an empty password? (yes/no) [no]: no       
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]: no
Username   : bookstack
Password   : <random>
Full Name  : Book Stack
Uid        : 1001
Class      : 
Groups     : bookstack wheel
Home       : /home/bookstack
Home Mode  : 
Shell      : /bin/sh
Locked     : no
OK? (yes/no): y
adduser: INFO: Successfully added (bookstack) to the user database.
adduser: INFO: Password for (bookstack) is: zd0myhtaIP.W56
Add another user? (yes/no): 
```
Run the visudo command and uncomment the %wheel ALL=(ALL) ALL line to allow members of the wheel group to execute any command.
```bash
visudo

# Uncomment by removing hash (#) sign
# %wheel ALL=(ALL) ALL
```

Navigate down to the line, press i when hovering over `#`, then press delete. Then press shift + ., so you have a colon `:` - and then write `wq` and press enter. 

Now, switch to your newly created user with su.
```bash
su - bookstack
```
NOTE: Replace bookstack with your username.


## Install PHP

Install PHP, as well as the necessary PHP extensions.

```bash
sudo pkg install -y php72 php72-mbstring php72-tokenizer php72-pdo php72-pdo_mysql php72-openssl php72-hash php72-json php72-phar php72-filter php72-zlib php72-dom php72-xml php72-xmlwriter php72-xmlreader php72-pecl-imagick php72-curl php72-session php72-ctype php72-iconv php72-gd php72-simplexml php72-zip php72-filter php72-tokenizer php72-calendar php72-fileinfo php72-intl php72-mysqli php72-phar php72-opcache php72-tidy
```
Check the version.
```bash
$ php --version
PHP 7.2.24 (cli) (built: Oct 27 2019 01:11:29) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.24, Copyright (c) 1999-2018, by Zend Technologies

```
Soft-link php.ini-production to php.ini.
```bash
$ sudo ln -s /usr/local/etc/php.ini-production /usr/local/etc/php.ini
Password:
$ 
```
Enable and start PHP-FPM.
```bash
sudo sysrc php_fpm_enable=yes
sudo service php-fpm start

$ sudo sysrc php_fpm_enable=yes
php_fpm_enable:  -> yes
$ sudo service php-fpm start
Performing sanity check on php-fpm configuration:
[05-Nov-2019 20:02:13] NOTICE: configuration file /usr/local/etc/php-fpm.conf test is successful

Starting php_fpm.
```

## Install MariaDB

Install MariaDB.
```bash
sudo pkg install -y mariadb102-client mariadb102-server
```

Check the version.
```bash
mysql --version
mysql  Ver 15.1 Distrib 10.2.26-MariaDB, for FreeBSD11.2 (amd64) using readline 5.1
```
Start and enable MariaDB.
```bash
sudo sysrc mysql_enable="yes" 
sudo service mysql-server start
```
Run the mysql_secure_installation script to improve the security of your installation.
```bash
sudo mysql_secure_installation



$ sudo mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
$ 

```
Log into MariaDB as the root user.
```bash
mysql -u root -p
# Enter password:
```
Create a new database and user. Remember the credentials for this new user.
```bash
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 16
Server version: 10.2.26-MariaDB FreeBSD Ports

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE bookstackdb;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL ON bookstackdb.* TO 'stackbook' IDENTIFIED BY '9HF733Hz';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit;
Bye
$ 
```

## Install Nginx

Install Nginx.

```bash
sudo pkg install -y nginx
```
Check the version.
```bash
$ nginx -v
nginx version: nginx/1.16.1
```
Enable and start Nginx.
```bash
sudo sysrc nginx_enable=yes
sudo service nginx start
```

Run `sudo nano /usr/local/etc/nginx/bookstack.conf` and set up Nginx for BookStack.

```bash
server {
  listen 80;
  listen [::]:80;
  server_name example.com;
  root /usr/local/www/bookstack/public;

  index index.php index.html;

  location / {
    try_files $uri $uri/ /index.php$is_args$args;
  }

  location ~ \.php$ {
    try_files $uri =404;
    include fastcgi_params;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass 127.0.0.1:9000;
  }
}

```
Save the file and exit with :+W+Q.

Now we need to include bookstack.conf in the main nginx.conf file.

Run `sudo nano /usr/local/etc/nginx/nginx.conf` and add the following line to the http {} block.
`include bookstack.conf;``

``bash
http {
    include       bookstack.conf;
    include       mime.types;
    default_type  application/octet-stream;
```

Test our Nginx configuration changes.
```bash
$ sudo nginx -t
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
$ 
```
Reload Nginx.
```bash
$ sudo service nginx reload
Performing sanity check on nginx configuration:
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
$ 
```

## Install Composer

Install Composer globally by running the following script in your terminal.
```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === 'a5c698ffe4b8e829a443b120cd5ca38043260d5c4023dbf93e1548871f1f07f58274fc6f4c93bcfd858c6bd0775cd8d1') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
```

NOTE: In the command block listed above, the hash will change with every version of the installer. Visit https://getcomposer.org/download for the latest Composer installation commands.

Check the version.
```bash
$ composer --version
Composer version 1.9.1 2019-11-01 17:20:17
$ 
```
## Install BookStack

Create a document root folder.
```bash
$ sudo mkdir -p /usr/local/www/bookstack
Password:
$ 

```
Change ownership of the /usr/local/www/bookstack directory to johndoe.
```bash
$ sudo chown -R bookstack:bookstack /usr/local/www/bookstack
```
Clone the release branch of the BookStack GitHub repository into the document root folder.
```bash
$ cd /usr/local/www/bookstack
$ git clone https://github.com/BookStackApp/BookStack.git --branch release --single-branch .
Cloning into '.'...
remote: Enumerating objects: 71, done.
remote: Counting objects: 100% (71/71), done.
remote: Compressing objects: 100% (71/71), done.
remote: Total 21700 (delta 0), reused 71 (delta 0), pack-reused 21629
Receiving objects: 100% (21700/21700), 12.47 MiB | 9.72 MiB/s, done.
Resolving deltas: 100% (14822/14822), done.
$ 
```

Run the composer install command from the /usr/local/www/bookstack directory.
```bash
composer install
```
Copy the .env.example file to .env and populate it with your own database and mail details.
```bash
cp .env.example .env
nano .env 

# Application key
# Used for encryption where needed.
# Run `php artisan key:generate` to generate a valid key.
APP_KEY=SomeRandomString

# Application URL
# Remove the hash below and set a URL if using BookStack behind
# a proxy, if using a third-party authentication option.
# This must be the root URL that you want to host BookStack on.
# All URL's in BookStack will be generated using this value.
#APP_URL=https://example.com

# Database details
DB_HOST=localhost
DB_DATABASE=bookstackdb      
DB_USERNAME=stackbook        
DB_PASSWORD=9HF733Hz

# Mail system to use
# Can be 'smtp', 'mail' or 'sendmail'
MAIL_DRIVER=smtp

# SMTP mail options
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null


# A full list of options can be found in the '.env.example.complete' file.
```
PS: Ensure that the storage, bootstrap/cache and public/uploads folders are writable by the web server.

In the application root, run the following command.
```bash
$ pwd
/usr/local/www/bookstack
$ php artisan key:generate

**************************************
*     Application In Production!     *
**************************************

 Do you really wish to run this command? (yes/no) [no]:
 > y

Application key [base64:80ucFtH1+owZhGN3eGEbWMYxESZBfHld9dQ4ZgwqrMI=] set successfully.
$ 
```

This will update `APP_KEY=SomeRandomString` from `.env`

Run `php artisan migrate` to update the database.
```bash
$ php artisan migrate
**************************************
*     Application In Production!     *
**************************************

 Do you really wish to run this command? (yes/no) [no]:
 > y


                                                                                                                                                                   
  [Illuminate\Database\QueryException]                                                                                                                             
  SQLSTATE[HY000] [1045] Access denied for user 'stackbook'@'localhost' (using password: YES) (SQL: select * from information_schema.tables where table_schema =   
  bookstackdb and table_name = migrations)                                                                                                                         
                                                                                                                                                                   

                                                                                               
  [Doctrine\DBAL\Driver\PDOException]                                                          
  SQLSTATE[HY000] [1045] Access denied for user 'stackbook'@'localhost' (using password: YES)  
                                                                                               

                                                                                               
  [PDOException]                                                                               
  SQLSTATE[HY000] [1045] Access denied for user 'stackbook'@'localhost' (using password: YES)  
                                                                                               

$ 
```
Change ownership of the `/usr/local/www/bookstack` directory to `www`.

```bash
sudo chown -R www:www /usr/local/www/bookstack
Password:
$ 
```

You can now login using the default admin details `admin@admin.com` with a password of `password`. It is recommended to change these details directly after your first login.


## Fault finding
### List users in Mariadb
 ```bash
root@BookStack:~ # mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 60
Server version: 10.2.26-MariaDB FreeBSD Ports

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SELECT User FROM mysql.user;
+-----------+
| User      |
+-----------+
| stackbook |
| root      |
| root      |
| root      |
+-----------+
4 rows in set (0.00 sec)
```

### Change password MariaDB
```bash
MariaDB [(none)]> SET PASSWORD FOR 'techonthenet'@'localhost' = PASSWORD('newpassword');
```

## Authors
Mr. Johnson

## Acknowledgments
* [https://www.vultr.com/docs/how-to-install-bookstack-on-freebsd-12](https://www.vultr.com/docs/how-to-install-bookstack-on-freebsd-12)


