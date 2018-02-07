---
title: "Setting up Web Page Speed test on Centos 7"
date: 2018-02-01
---

# What the machine currently has

CentOS Linux release 7.4.1708 (Core)  
WebPageTest 17.12
Apache (httpd): 2.4.6 (67.el7.centos.6)
PHP: 5.4.16 (43.el7_4)

# Commands
sudo yum install httpd php php-gd php-mbstring php-pdo

https://www.vultr.com/docs/how-to-install-ffmpeg-on-centos 

# Verify Apache and PHP is working
Start apache service: sudo systemctl start httpd
Create test.php in /var/www/html

```php
<?php
   phpinfo(); 
?>
```

Should get PHP info screen that looks something like this: https://i.imgur.com/Zh9Sk27.jpg

# Download web page speed test
https://github.com/WPO-Foundation/webpagetest/releases

Unpack to /opt/webpagespeedtest

# Connect apache to folder
Going to assume you want this on root

Put /etc/httpd/conf.d/webpagespeedtest.conf
```
Alias "/" "/opt/webpagespeedtest/www/"
<Directory "/opt/webpagespeedtest/www">
  DirectoryIndex index.php
  Order allow,deny
  AllowOverride All
  Require all granted
  Allow from all
</Directory>
```

# Verify that web page speed test works

Go to the site and you should see it working

Next: http://watchtower-speedtest.fedbase.dev.ostk.com/install/

# Setting up the site
Copy all sample files at /opt/webpagespeedtest/www/settings

```
for i in *sample
do
  cp -a $i ${i%%sample}
done
```

# Agent Install

Install chrome

Add yum repo

/etc/yum.repos.d/google-chrome.repo

```
[google-chrome]
name=google-chrome
baseurl=http://dl.google.com/linux/chrome/rpm/stable/$basearch
enabled=1
gpgcheck=1
gpgkey=https://dl-ssl.google.com/linux/linux_signing_key.pub
```

yum install python2-pip gcc gcc-c++ python-devel

pip install --upgrade pip

pip install psutil monotonic ujson

/opt/webpagespeedtest/wptagent
cp browsers.ini.sample browsers.ini

python wptagent.py -vvvv --location ostk --server "http://watchtower-speedtest.fedbase.dev.ostk.com/work/" --name ostk_wptdriver
