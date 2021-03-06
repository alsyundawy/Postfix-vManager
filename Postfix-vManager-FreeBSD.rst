==========================================================
  Postfix vManager FreeBSD
==========================================================

:Version: 2.0
:Source: https://app.box.com/s/lsz0b4w8cmu46qaz02w39far0djk1uny
:Keywords: Postfix Mail server management, Cyrus-SASL, Dovecot, MySQL, Virtual Domains, Alias Domains

Author
==========

Copyright (C) Umar Draz <umar_draz@yahoo.com>

Table of Contents
=================

::

  1. Requirements
  2. MySQL Server Installation
  3. Postfix and Dovecot
  4. Web Server Installation
  5. Postfix vManager Installation
  6. OpenDKIM (Domain Keys)

1. Requirements
===============

* Postfix which supports virtual domains and mailboxes including vda patch.
* a compatible IMAP/POP3 server (such as Dovecot and Courier), but in this howto we will only discuss Dovecot
* PHP v4 and above with PDO and mysqli supported.

2. MySQL Server Installation
============================

Installing MySQL 5 Server on FreeBSD is a quick and easy process. In classic fashion let’s get the process underway by updating our system.

::

  cd /usr/ports/databases/mysql51-server/
  make install clean

Add the following line to /etc/rc.conf:

::

  echo 'mysql_enable="YES"' >> /etc/rc.conf

And start the mysql server:

::

  /usr/local/etc/rc.d/mysql-server start
  
Then set a password for the MySQL root user:

::

  /usr/local/bin/mysqladmin -u root password 'your-password'

That's All MySQL server has been installed, now lest install Postfix and Dovecot.

3. Postfix and Dovecot
======================

Let's start the Mail server Installation.

::

  cd /usr/ports/mail/postfix
  make install clean

In the option list window you must select these options.

::

  [ .. ] MYSQL
  [ .. ] SASL2
  [ .. ] SPF
  [ .. ] TLS
  [ .. ] VDA
  [ .. ] DOVECOT2

And on Dovecot installation, when window appear, you must also select [ .. ] MYSQL option.

This will install the Postfix mail server, the Dovecot IMAP and POP daemons, and several supporting packages that provide services related to authentication.

After sucessfully installation of Postfix, in the next step will create database for our Postfix vManager.

2. Set up MySQL database for Virtual Domains and Users
-----------------

Start the MySQL shell by issuing the following command. You'll be prompted to enter the root password for MySQL that you assigned during the initial setup.

::

  mysql -u root -p

You'll be presented with an interface similar to the following:

::

  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 48
  Server version: 5.5.31-0ubuntu0.12.04.1 (Ubuntu)

  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

  mysql>

Issue the following command to create a database for your mail server and switch to it in the shell:

::

  CREATE DATABASE vmanager;
  USE vmanager;

Create a mail administration user called vadmin and grant it permissions on the mail database with the following commands. Please be sure to replace "vadmin_password" with a password you select for this user.

::

  GRANT SELECT, INSERT, UPDATE, DELETE ON vmanager.* TO 'vadmin'@'localhost' IDENTIFIED BY 'vadmin_password';
  FLUSH PRIVILEGES;

That's all we have sucessfully create database for our application, latter on we will restore our database schema into vmanager database when we will install Postfix vManager.

3.3. Configure Postfix to work with MySQL
-----------------

Create a virtual forwarding file called /etc/postfix/mysql_virtual_forwarders_maps.cf for forwarding emails from one email address to another, with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_forwarders_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT goto FROM forwarders WHERE address='%s' AND active = '1'

Create a virtual domain configuration file for Postfix called /etc/postfix/mysql_virtual_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT domain FROM domain WHERE domain='%s' and active='1'

Create a virtual mailbox configuration file for Postfix called /etc/postfix/mysql_virtual_mailbox_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_mailbox_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT CONCAT(domain,'/',maildir) FROM mailbox WHERE username='%s' AND active = '1'

Create a mailbox quota limit configuration file for Postfix called /etc/postfix/mysql_virtual_mailbox_limit_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_mailbox_limit_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT quota FROM mailbox WHERE username='%s'

Create a sender check configuration file called /etc/postfix/mysql_sender_check.cf so after smtp authentication senders can not use our mail server as open relay.

**File:** /usr/local/etc/postfix/mysql_sender_check.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT username FROM ( SELECT username as username FROM mailbox UNION ALL SELECT address FROM alias_domain) a where username = '%s'

Create a transport map configuration file called /etc/postfix/mysql_transport.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_transport.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT destination FROM transport where domain = '%s'

Create an alias domains configuration file called /etc/postfix/mysql_virtual_alias_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_alias_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT target_domain FROM alias_domain WHERE address = '%s' OR address = concat('@', SUBSTRING_INDEX('%s', '@', -1)) AND concat('@', alias_domain) = '%s' AND active = '1'

Create a parking domain configuration file called /etc/postfix/mysql_parking_domains_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_parking_domains_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT domain FROM parking_domains WHERE domain='%s' and active = '1'

Create a virtual groups configuration file called /etc/postfix/mysql_virtual_groups_maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_virtual_groups_maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT goto FROM groups WHERE address='%s' AND active = '1'

Create an alias domains relay configuration file called /etc/postfix/mysql_alias_domains.maps.cf with the following contents. Be sure to replace "vadmin_password" with the password you chose earlier for the MySQL mail administrator user.

**File:** /usr/local/etc/postfix/mysql_alias_domains.maps.cf

::

  user = vadmin
  password = vadmin_password
  hosts = localhost
  dbname = vmanager
  query = SELECT DISTINCT alias_domain FROM alias_domain WHERE alias_domain='%s' and active = '1'
  
Set proper permissions and ownership for these configuration files by issuing the following commands:

::

  chmod o= /usr/local/etc/postfix/mysql_*
  chgrp postfix /usr/local/etc/postfix/mysql_*

Next, we'll create a user and group for mail handling. All virtual mailboxes will be stored under this user's home directory.

::

  pw groupadd vmail -g 150
  pw useradd vmail -g vmail -u 150 -d /home/vmail -m

Now create /etc/postfix/main.cf with the following contents Please be sure to replace "example.yourdomain.com" with the fully qualified domain name you used for your system mail name.

**File:** /usr/local/etc/postfix/main.cf

::

  soft_bounce = no
  smtpd_banner = $myhostname
  biff = no
  append_dot_mydomain = no
  inet_interfaces = all
  myhostname = example.yourdomain.com
  myorigin = $myhostname
  mydomain = yourdomain.com
  mynetworks = 127.0.0.0/8
  mynetworks_style = host
  mydestination = $myhostname, localhost.$mydomain, localhost
  alias_maps = $virtual_alias_maps
  local_transport = local
  transport_maps = proxy:mysql:$config_directory/mysql_transport.cf
  debug_peer_level = 2
  debugger_command =
         PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
         ddd $daemon_directory/$process_name $process_id & sleep 5
  html_directory = /usr/local/share/doc/postfix
  disable_vrfy_command = yes
  mailbox_size_limit = 0
  owner_request_special = no
  recipient_delimiter = +
  home_mailbox = Maildir/
  mail_owner = postfix
  command_directory = /usr/local/sbin
  daemon_directory = /usr/local/libexec/postfix
  data_directory = /var/db/postfix
  queue_directory = /var/spool/postfix
  sendmail_path = /usr/local/sbin/sendmail
  newaliases_path = /usr/local/bin/newaliases
  mailq_path = /usr/local/bin/mailq
  mail_spool_directory = /var/spool/mail
  manpage_directory = /usr/local/man
  setgid_group = maildrop
  unknown_local_recipient_reject_code = 450

  # Virtual Domains and Users
  virtual_transport = virtual
  virtual_alias_maps =
    proxy:mysql:$config_directory/mysql_virtual_forwarders_maps.cf,
    proxy:mysql:$config_directory/mysql_virtual_groups_maps.cf,
    proxy:mysql:$config_directory/mysql_virtual_alias_domains_maps.cf
  virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
  virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
  virtual_mailbox_limit_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_limit_maps.cf
  virtual_mailbox_base = /home/vmail
  relay_domains =
    proxy:mysql:$config_directory/mysql_parking_domains_maps.cf,
    proxy:mysql:$config_directory/mysql_alias_domains.maps.cf
  proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_mailbox_maps $virtual_mailbox_domains $relay_domains $virtual_mailbox_limit_maps $transport_maps
  virtual_minimum_uid = 150
  virtual_uid_maps = static:150
  virtual_gid_maps = static:150

  # Additional for quota support
  virtual_mailbox_limit_override = yes
  virtual_maildir_limit_message = Sorry, the user's mail quota has exceeded.
  virtual_overquota_bounce = yes

  # SMTP Authentication 
  smtpd_sasl_auth_enable = yes
  smtpd_sasl_security_options = noanonymous
  broken_sasl_auth_clients = yes
  smtpd_sasl_authenticated_header = yes
  smtpd_sasl_type = dovecot
  smtpd_sasl_path = private/auth

  # TLS/SSL
  smtpd_use_tls = yes
  smtpd_tls_auth_only = no
  smtpd_tls_cert_file = /usr/local/etc/postfix/smtpd.cert
  smtpd_tls_key_file = /usr/local/etc/postfix/smtpd.key

  # Other Configurations
  strict_rfc821_envelopes = yes
  smtpd_soft_error_limit = 10
  smtpd_hard_error_limit = 20
  smtpd_data_restrictions = reject_unauth_pipelining, reject_multi_recipient_bounce
  smtpd_etrn_restrictions = reject
  smtpd_helo_required = yes
  smtpd_recipient_limit = 25
  #smtpd_sender_login_maps = mysql:$config_directory/mysql_sender_check.cf

  smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_invalid_hostname,
    reject_unauth_pipelining,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain,
    permit

  smtpd_sender_restrictions =
    permit_mynetworks,
    reject_unverified_sender,
    #reject_sender_login_mismatch,
    #reject_unauthenticated_sender_login_mismatch,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit

This completes the configuration for Postfix. Next, you'll make an SSL certificate for the Postfix server that contains values appropriate for your organization.

Create an SSL Certificate for Postfix
-----------------

Issue the following commands to create the SSL certificate

::

  cd /usr/local/etc/postfix
  openssl req -new -outform PEM -out smtpd.cert -newkey rsa:2048 -nodes -keyout smtpd.key -keyform PEM -days 365 -x509

You will be asked to enter several values similar to the output shown below. Be sure to enter the fully qualified domain name you used for the system mailname in place of "example.yourdomain.com".

::

  Country Name (2 letter code) [AU]:PK
  State or Province Name (full name) [Some-State]:Punjab
  Locality Name (eg, city) []:Lahore
  Organization Name (eg, company) [Internet Widgits Pty Ltd]:MyComapny
  Organizational Unit Name (eg, section) []:Email Services
  Common Name (eg, YOUR name) []:example.yourdomain.com
  Email Address []:webmaster@yourdomain.com

Set proper permissions for the key file by issuing the following command:

::

  chmod o= /usr/local/etc/postfix/smtpd.key

This completes SSL certificate creation for Postfix. Next, you'll need to configure Dovecot for imap service.

3.4. Configure Dovecot
-----------------

Replace the contents of the file with the following example, substituting your system's domain name for yourdomain.com.

**File:** /usr/local/etc/dovecot/dovecot.conf

::

  auth_mechanisms = plain login
  base_dir = /var/run/dovecot/
  disable_plaintext_auth = no
  first_valid_gid = 150
  first_valid_uid = 150
  last_valid_gid = 150
  last_valid_uid = 150
  log_path = /var/log/maillog
  log_timestamp = %Y-%m-%d %H:%M:%S
  auth_username_format = %Lu
  mail_access_groups = mail
  mail_location = maildir:~/Maildir
  passdb {
    args = /usr/local/etc/dovecot/dovecot-mysql.conf
    driver = sql
  }
  protocols = imap
  service auth {
    unix_listener /var/spool/postfix/private/auth {
      group = postfix
      mode = 0660
      user = postfix
    }
  }
  service imap-login {
    inet_listener imap {
      address = *
      port = 143
    }
  }
  service pop3-login {
    inet_listener pop3 {
      address = *
      port = 110
    }
  }

  ssl = yes
  ssl_cert = </usr/local/etc/postfix/smtpd.cert
  ssl_key = </usr/local/etc/postfix/smtpd.key

  userdb {
    args = /usr/local/etc/dovecot/dovecot-mysql.conf
    driver = sql
  }

MySQL will be used to store password information, so /etc/dovecot/dovecot-mysql.conf must be edited. Replace the contents of the file with the following example, making sure to replace "vadmin_password" with your mail password.

**File:** /usr/local/etc/dovecot/dovecot-mysql.conf

::

  driver = mysql
  connect = host=localhost user=vadmin password=vadmin_password dbname=vmanager
  default_pass_scheme = MD5-CRYPT
  password_query = SELECT password FROM mailbox WHERE username = '%u'
  user_query = SELECT '/home/vmail/%d/%n/Maildir' as home, 'maildir:/home/vmail/%d/%n/Maildir' as mail, 150 AS uid, 150 AS gid, concat('dirsize:storage=',quota) AS quota FROM mailbox WHERE username ='%u' AND active ='1'

Postfix and Dovecot has now been configured, add the following lines to /etc/rc.conf so that the Postfix and Dovecot will start automatically ast system boot.

::

  echo 'postfix_enable="YES"' >> /etc/rc.conf
  echo 'dovecot_enable="YES"' >> /etc/rc.conf

You must restart Postfix and Dovecot to make sure both work properly.

::

  service dovecot restart
  service postfix restart
  
Thats all Postfix and Dovecot installation is completed. Now let's install Apache and PHP for Postfix vManager Application.

Testing TLS
===========

To verify Postfix supports TLS, it has to be displaying STARTTLS when you connect to port 25 with telnet and run the EHLO command. We set this up in a previous step.

To verify the SSL certificate is working and Postfix can negotiate the SSL encryption you can use the openssl command.

::

  openssl s_client -starttls smtp -crlf -connect mail.example.com:25

Substitute mail.example.com with the hostname of your mail server.

4. WebServer Installation
=========================

Apache is easily installed by entering the following command.

::

  cd /usr/ports/www/apache22
  make install clean

Once Apache has been successfully installed, add the following line to /etc/rc.conf so that the Apache server will start automatically at system boot.

::

  echo 'apache22_enable="YES"' >> /etc/rc.conf

**Configure Name-based Virtual Hosts**

Now we will create virtual host entries for example.yourdomain.com site that we need to host with this server. Here is this.

**File:** /usr/local/etc/apache22/httpd.conf

::

  NameVirtualHost *:80
  <VirtualHost *:80>
    ServerAdmin webmaster@yourdomain.com
    ServerName yourdomain.com
    ServerAlias example.yourdomain.com
    DocumentRoot /usr/local/www/vmanager
    ErrorLog /var/log/vmanager-error.log
    CustomLog /var/log/vmanager-access.log combined
  </VirtualHost>

Before you can use the above configuration you'll need to create the specified directories. For the above configuration, you can do this with the following commands:

::

  mkdir /usr/local/www/vmanager

Postfix vManager depends on url rewriting for SEO purpose. In order to take advantage of this feature we need to edit httpd.conf file as follows.

Edit /usr/local/etc/apache22/httpd.conf file and change **AllowOverride None** to **AllowOverride All** under / directory e.g.

::

  <Directory />
    Options FollowSymLinks
    AllowOverride All
  </Directory>

Installing PHP
-----------------

We will therefore install PHP with the following command.

::

  cd /usr/ports/lang/php5
  make install cleean

A menu should come up allowing you to select/deselect various build options. You should select “Build Apache module” by highlighting the option with the arrow keys and hitting the space bar, then hit Enter.

After PHP installation we need add the requisite extensions to PHP for Postfix vManager. 

::

  cd /usr/ports/lang/php5-extensions/
  make install clean

In the menu list you must select these extensions. Don't uncheck other selected options.

::

  [ .. ] MBSTRING
  [ .. ] MYSQL
  [ .. ] MYSQLI
  [ .. ] PDO_MYSQL

**Enable PHP into Apache**

Now that we have the requisite ports built and installed it’s time to configure them.

Before going online with your site, you should consider copying /usr/local/etc/php.ini-production into php.ini

::

  cp /usr/local/etc/php.ini-development /usr/local/etc/php.ini

Now enable php into apache, open the file /usr/local/etc/apache22/httpd.conf in your favorite editor and look for the following line:

::

  DirectoryIndex index.html

And change it so it reads as follows:

::

  DirectoryIndex index.html index.htm index.php
  
Then append the following lines to the end of the file in order to support PHP files as well as Postfix vManager

::

  AddType application/x-httpd-php .php
  AddType application/x-httpd-php-source .phps

Here we need to restart apache server.

::

  service apache22 restart

If everything has gone according to plan you should be able to open a browser and navigate to **example.yourdomain.com**

5. Postfix vManager
===================

First download postfix vmanager source from this url :Source: https://app.box.com/s/lsz0b4w8cmu46qaz02w39far0djk1uny

After downloading the postfix-vmanager-2.0.tar.gz just extract the source. 

Then first remove the /usr/local/www/vmanager directory and move extracted source into /usr/local/www/vmanager/ let's do it.

::

  tar xzvpf postfix-vmanager-2.0.tar.gz
  rm -rf /usr/local/www/vmanager
  mv postfix-vmanager-2.0 /usr/local/www/vmanager
  
Next restore the database, with the following command

::

  cd /usr/local/www/vmanager/
  mysql -uroot -proot_pass vmanager < setup/vmanager.sql

5.1. Configure Postfix vManager
----------------------

Edit the inc/config.inc.php file and add your settings there. The most important settings are those for your database server.

::

  $CONF['database_host'] = 'localhost';
  $CONF['database_user'] = 'vadmin';
  $CONF['database_password'] = 'vadmin_password';
  $CONF['database_name'] = 'vmanager';
  $CONF['database_port'] = '3306';
  $CONF['database_prefix'] = '';

Postfix vManager require write access to its directory. So you need to change the vmanager directory ownership with that user as web server running.

::

  chown -R www:www /usr/local/www/vmanager/

5.2. Check settings, and create Admin user
------------------------------------------

Hit :Source: https://example.yourdomain.com/ in a web browser. You should see a list of 'OK' messages. Otherwise reslove the issue if found. 

Create the admin user using the form displayed. This is all that is needed.

5.3. Vacations
--------------

The vacation script runs as service within Postfix's master.cf configuration file. Mail is sent to the vacation service via a transport table mapping. When users mark themselves as away on vacation, an alias is added to their account sending a copy of all mail to them to the vacation service.

To use vacation services you need to first create vacation domain. Just login as Super Admin account and then 

5.4. Installing Vacations
-------------------------

Login as Super Admin and then create Vacation domain following this.

::

  Go to Settings -> Vacation Domain.

There are a bunch of Perl modules which we need to install for Vacation setup.

::

  cd /usr/ports/mail/p5-MIME-EncWords ; make install clean
  cd /usr/ports/mail/p5-Email-Valid ; make install clean
  cd /usr/ports/mail/p5-Email-Sender ; make install clean
  cd /usr/ports/mail/p5-Mail-Sender ; make install clean
  cd /usr/ports/devel/p5-Log-Log4perl ; make install clean
  cd /usr/ports/devel/p5-Log-Dispatch ; make install clean
  cd /usr/ports/databases/p5-DBI ; make install clean
  cd /usr/ports/databases/p5-DBD-mysql ; make install clean

**Create Vacation Account:**

Create a dedicated local user account called "vacation". This user handles all potentially dangerous mail content - that is why it should be a separate account.

Do not use "nobody", and most certainly do not use "root" or "postfix". The user will never log in, and can be given a "*" password and non-existent shell and home directory.

Create the user with the following command.

::

  pw useradd vacation -c "Vacation Owner" -d /nonnonexistent -s /usr/bin/false

**Create a directory:**

Create a directory, for example  /var/spool/vacation, that is accessible only to the "vacation" user. This is where the vacation script is supposed to store its temporary files. 

::

  mkdir /var/spool/vacation
  
**Copy Files:**

Copy the vacation.pl file to the directory you created above:

::

  cp setup/vacation.pl /var/spool/vacation/vacation.pl
  chown -R vacation:vacation /var/spool/vacation/
  
Which will then look something like:

::

  -rwx------   1 vacation  vacation  3356 Dec 21 00:00 vacation.pl*

**Setup the transport type:**

Define the transport type in the Postfix /etc/postfix/master.cf file:

::

  vacation    unix  -       n       n       -       -       pipe
    flags=Rq user=vacation argv=/var/spool/vacation/vacation.pl -f ${sender} -- ${recipient}
    
Here we need to restart postfix service.

::

  service postfix restart

**Configure vacation.pl"**

The perl /var/spool/vacation/vacation.pl script needs to know which database you are using, and also how to connect to the database.

Change any variables starting with '$db_' and '$db_type'

Change the $vacation_domain variable to match what you entered through your Super Admin login.

Here is the example of vacatino.pl settings for database and domain name

::

  our $db_type = 'mysql';
  our $db_host = 'localhost';
  our $db_username = 'username';
  our $db_password = 'password';
  our $db_name     = 'dbname';
  our $vacation_domain = 'autoreply.yourdomain.com';

Done! When this is all in place you need to have a look at the Postfix vManager inc/config.inc.php. Here you need to enable Virtual Vacation for the site.

6. Domain Keys
==============

I’m going to show you how to run Postifx with OpenDKIM on a FreeBSD operating system.

Prepare the installation of OpenDKIM adding an opendkim user:

::

  pw useradd -n opendkim -d /var/db/opendkim -g mail -m -s "/usr/sbin/nologin" -w no

Let’s start installing OpenDKIM.

::

  cd /usr/ports/mail/opendkim
  make install clean

Then allow OpenDKIM starting at boot time and executing as opendkim user:

::

  echo 'milteropendkim_enable="YES"' >> /etc/rc.conf
  echo 'milteropendkim_uid="opendkim"' >> /etc/rc.conf

Edit Postfix configuration file.

::

  nano /usr/local/etc/postfix/main.cf

And instruct postfix to use dkim milter:

::

  smtpd_milters = inet:127.0.0.1:8891
  non_smtpd_milters = $smtpd_milters
  milter_default_action = accept

Create configuration file for OpenDKIM

::

  nano /usr/local/etc/opendkim.conf
  
Feel free to use the following one slightly edited to work with **yourdomain.com** domain:

::

  LogWhy            yes
  Syslog            yes
  SyslogSuccess     yes
  Canonicalization  relaxed/simple
  KeyTable          /var/db/opendkim/KeyTable
  SigningTable      /var/db/opendkim/SigningTable
  InternalHosts     /var/db/opendkim/TrustedHosts
  Socket            inet:8891@localhost
  ReportAddress     root
  SendReports       yes

Edit /var/db/opendkim/TrustedHosts

::

  nano /var/db/opendkim/TrustedHosts

Add domains, hostnames and/or ip’s that should be handled by OpenDKIM. Don’t forget localhost.

::

  127.0.0.1
  localhost
  x.253.204.64
  x.253.204.32/27

**Generate keys**

Now generate the keys: one will be used by opendkim to sign your messages and the other to be inserted in your dns zone:

::

  mkdir -p /var/db/opendkim/yourdomain.com
  opendkim-genkey -D /var/db/opendkim/yourdomain.com/ -d yourdomain.com -s default
  
Here you need to move **default.private** to **default**

::

  cd /var/db/opendkim/yourdomain.com/
  mv default.private default
  chown opendkim:mail /var/db/opendkim/
  
Add domain to KeyTable /var/db/opendkim/KeyTable

::

  default._domainkey.yourdomain.com yourdomain.com:default:/var/db/opendkim/yourdomain.com/default

Add domain to SigningTable /var/db/opendkim/SigningTable

::

  yourdomain.com default._domainkey.yourdomain.com

**Add to DKIM public key to DNS**

Add an entry for the public key to the DNS server you are using for your domain. You find the public key here:

::

  cat /var/db/opendkim/yourdomain.com/default.txt
  
The above output should be like.

::

  default._domainkey IN TXT "v=DKIM1; g=*; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQClJj0qvcQvX7ssbGNBqFCTt+Wrh9G15QIXkFPbspt4uUOthLR8yl56CKohRVFfQTjoZjrmxSYDD8ZfV4rnPUu5bz07w7hbL3X1N5rLOM7RTDWU0NrYzGNVS07H4XNUJQRifVULREEqqvjASX6ivp1AH+OvvKn9mQTaSTjviD2cdQIDAQAB"

Now test the key using an OpenDKIM utiliy:

::

  opendkim-testkey -vvv -d yourdomain.com -s default -k /var/db/opendkim/default
  
The above command will verify your zone settings.

Now start both OpenDKIM and Postifix:

::

  /usr/local/etc/rc.d/milter-opendkim start
  /usr/local/etc/rc.d/postfix restart

Look at the DKIM-signature, there it is.

Further check and analysis can be made also on the website http://www.brandonchecketts.com/emailtest.php
