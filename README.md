General Information:
--------------------

This is a setup for a TOR based shared hosting server. It is provided as is and before putting it into production you should make changes according to your needs. This is a work in progress and you should carefully check the commit history for changes before updating.

Installation Instructions:
--------------------------

The configuration was tested with a standard Debian buster and Ubuntu 18.04 LTS installation. It's recommended you install Debian buster (or newer) on your server, but with a little tweaking you may also get this working on other distributions and/or versions.

Uninstall packages that may interfere with this setup:
```
apt-get purge apache2* resolvconf exim4* && systemctl disable systemd-resolved.service
```

To get the latest tor version, you should follow these instructions to add the official tor repository for your distribution: (https://www.torproject.org/docs/debian)

To get the latest mariadb version, you should follow these instructions to add the official tor repository for your distribution: (https://downloads.mariadb.org/mariadb/repositories/)

Add yarn + nodejs to our repositories:
```
curl -sSL https://deb.nodesource.com/gpgkey/nodesource.gpg.key | apt-key add -
curl -sSL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" >> /etc/apt/sources.list
echo "deb https://deb.nodesource.com/node_11.x sid main" >> /etc/apt/sources.list
```

The following command will install all required packages:
```
apt-get --no-install-recommends install apt-transport-tor bzip2 clamav-daemon clamav-freshclam clamav-milter curl dovecot-imapd dovecot-pop3d git dnsmasq haveged iptables libsasl2-modules locales locales-all logrotate mariadb-server nano nodejs postfix postfix-mysql quota quotatool rsync sasl2-bin ssh subversion tor unzip vim vsftpd wget yarn zip
```
The following command will install all required build dependencies for nginx and php:
```
apt-get --no-install-recommends install -y autoconf bison g++ gcc ghostscript libargon2-dev libbz2-dev libbrotli-dev libc-client2007e-dev libcurl4-openssl-dev libedit-dev libenchant-dev libffi-dev libgd-dev libgmp-dev libkrb5-dev libldap2-dev liblmdb-dev libmagickwand-dev libmariadb-dev libonig-dev libsasl2-dev libpcre3-dev libpng-dev libpspell-dev libqdbm-dev libreadline-dev libsasl2-dev libsodium-dev libsqlite3-dev libssh2-1-dev libssl-dev libsystemd-dev libtidy-dev libwebp-dev libxml2-dev libxpm-dev libxslt1-dev libzip-dev make poppler-utils re2c zlib1g-dev
```

Note that both, debian and the torproject have hidden service package archives, so you may want to edit /etc/apt/sources.list to load from those instead:
```
deb tor://vwakviie2ienjx6t.onion/debian sid main
deb tor://sdscoq7snqtznauu.onion/torproject.org sid main
```

Copy (and modify according to your needs) the site files in `var/www` to `/var/www` and the configuration files in `etc` to `/etc` after installation has finished. Then restart some services:
```
systemctl daemon-reload && service tor restart && service dnsmasq restart
```

Now there should be an onion domain in `/var/lib/tor/hidden_service/hostname`:
```
cat /var/lib/tor/hidden_service/hostname
```

Replace the default domain with your domain in the following files:
```
/etc/postfix/sql/alias.cf
/etc/postfix/sender_login_maps
/etc/postfix/main.cf
/var/www/skel/www/index.hosting.html
/var/www/common.php
/etc/postfix/canonical
/etc/postfix-clearnet/canonical
```

In `/etc/postfix(-clearnet)/canonical` don't change the line that has `hosting.danwin1210.me` in it. It is a clearnet/tor address rewriting rule, and if you have your own clearnet domain, you should copy this and modify your copy to preserve sending mail to my host via tor and not via clearnet:

To allow sasl authentication add the `postfix` user to the `sasl` group:
```
usermod -aG sasl postfix
```

This setup has two postfix instances, one for receiving and sending mail to other .onion services and one for rewriting addresses to pass them on to a clearnet facing mail relay. You may or may not want to create the second instance by running
```
postmulti -e init
postmulti -I postfix-clearnet -e create
postmulti -i clearnet -e enable
postmulti -i clearnet -p start
```
If you created an instance, uncomment the clearnet relay related config in etc/postfix/main.cf and make sure to copy and modify the configuration files from etc/postfix-clearnet too

If you encountered the following issue: `postfix: fatal: chdir(/var/spool/postfix-clearnet): No such file or directory` you can just copy the chroot from the default postfix instance like this `cd /var/spool/ && cp -a postfix/ postfix-clearnet/`

After copying (and modifying) the posfix configuration, you need to create databases out of the mapping files (also each time you update those files):
```
postalias /etc/aliases
postmap /etc/postfix/canonical /etc/postfix/sender_login_maps /etc/postfix/transport
postmap /etc/postfix-clearnet/canonical /etc/postfix-clearnet/sasl_password /etc/postfix-clearnet/transport #only if you have a second instance
```

To save temporary files in memory, add the following to `/etc/fstab`:
```
tmpfs /tmp tmpfs defaults,noatime 0 0
tmpfs /var/log/nginx tmpfs rw,user,noatime 0 0
```

As time syncronisation is important, you should configure ntp servers in `/etc/systemd/timesyncd.conf` and make them match with the entries in `/etc/rc.local` iptables configuration

Enable the PHP-FPM default instances and nginx:
```
systemctl enable php7.2-fpm@default
systemctl enable php7.3-fpm@default
systemctl enable php7.4-fpm@default
systemctl enable nginx
```

Edit `/etc/fstab` and add the `noatime,usrjquota=aquota.user,jqfmt=vfsv1` option to the `/home` mountpoint and `noatime`to `/`. Then initialize quota:
```
mount -o remount /home
quotacheck -cu /home
quotaon /home
```

Install custom optimized binaries
```
./install_binaries.sh
```

Install composer and sodium_compat for v3 hidden_service support
```
curl -sSL https://github.com/composer/composer/releases/download/1.9.1/composer.phar > /usr/bin/composer && chmod +x /usr/bin/composer
cd /var/www && composer install
```

For web base database administration, check out the latest phpmyadmin and adminer:
```
cd /var/www/html/ && git clone -b STABLE https://github.com/phpmyadmin/phpmyadmin/ && cd phpmyadmin && composer install --no-dev && yarn
cd /var/www/html/ && git clone https://github.com/vrana/adminer/ && cd adminer && git submodule update --init
```

Once installed create a mysql user for phpmyadmin and cofigure it in `/var/www/html/phpmyadmin/config.inc.php` and fill `$cfg['blowfish_secret']` with random characters:
```
mysql
CREATE USER 'phpmyadmin'@'%' IDENTIFIED BY 'MY_PASSWORD';
CREATE DATABASE phpmyadmin;
GRANT ALL PRIVILEGES ON phpmyadmin.* TO 'phpmyadmin'@'%';
FLUSH PRIVILEGES;
quit
```

For web based mail management grab the latest squirrelmail and install it in `/var/www/html/squirrelmail`:
```
cd /var/www/html/ && svn checkout https://svn.code.sf.net/p/squirrelmail/code/trunk/squirrelmail && cd squirrelmail && ./configure && mkdir /var/local/squirrelmail /var/local/squirrelmail/data /var/local/squirrelmail/attach && chown www-data:www-data /var/local/squirrelmail /var/local/squirrelmail/data /var/local/squirrelmail/attach
```

Once it is downloaded, it will ask you for configuration. Things to change are:
```
D. > select dovecot
2. Server Settings > 1. Domain > Set your own .onion domain here
2. Server Settings > B. Update SMTP settings > 7. SMTP Authentication -> y -> plain -> n User are authenticated using their username + password
4. General Options > 9. Allow editing of identity > n Users should not be able to fake email addresses > y They should be able to change display name > y They should be able to set a reply to mail > y additional headers are not required
10. Language settings > 4. Enable aggressive decoding
11. Tweaks > 2. Ask user info on first login > n (commonly confuses users)
11. Tweaks > 5. Use php iconv functions > y
```

Create a mysql user with all permissions for our hosting management:
```
mysql
CREATE USER 'hosting'@'%' IDENTIFIED BY 'MY_PASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'hosting'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
quit
```

Then edit the database configuration in `/var/www/common.php` and `/etc/postfix/sql/alias.cf`

Last but not least setup the database by running
```
php /var/www/setup.php
``` 

Enable systemd timers to regularly run various managing tasks:
```
systemctl enable hosting-del.timer && systemctl enable hosting.timer
```

Final step is to reboot wait about 5 minutes for all services to start and check if everything is working by creating a test account.

Live demo:
----------

If you want to see the setup in action or create your own site on my server, you can visit my [TOR hidden service](http://dhosting4xxoydyaivckq7tsmtgi4wfs3flpeyitekkmqwu4v4r46syd.onion) or via [my clearnet proxy](https://hosting.danwin1210.me) if you don't have TOR installed.
