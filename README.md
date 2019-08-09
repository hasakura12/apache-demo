# Get Started with Apache2 for Ubuntu
Index
- [Install](#install)
- [Restrict access to authorized users](#restrict_users)
- [Restrict access to group members](#restrict_group)
- [Add TLS/HTTPS support](#https_support)
- [What's next (Nginx)](#whats_next)


## Install <a name="install"></a>
```
apt update
apt install apache2

# make sure nothing is listening on port 80 (like Nginx)
netstat -anltp

# make sure apache2 service is running
systemctl status apache2

# verify apache2 returns a homepage
curl localhost
```


## Restrict access to authorized users <a name="restrict_users"></a>
```
# create custom dir for this demo
mkdir /etc/apache2/apache_demo.com

# change user/group ownership of the custom dir to www-data, recursively
chown www-data:www-data -R /etc/apache2/apache_demo.com/
chmod 2755 -R /etc/apache2/apache_demo.com/

# copy existing apache conf to the custom dir
cp /etc/apache2/sites-enabled/000-default.conf /etc/apache2/apache_demo.com/apache2.conf

# unlink symlink of existing conf because we'll use custom conf
unlink /etc/apache2/sites-enabled/000-default.conf

# create symlink from /etc/apache2/apache_demo.com/apache2.conf to ../site-enabled
ln -s /etc/apache2/apache_demo.com/apache2.conf /etc/apache2/sites-enabled/


# create html contents 
mkdir /var/www/apache_demo.com
chown www-data:www-data -R /var/www/apache_demo.com
chmod 2755 -R /var/www/apache_demo.com

# create a simple index.html
echo "hello from apache_demo.com" > /var/www/apache_demo.com/index.html

# create a private dir
mkdir /var/www/apache_demo.com/private

# create a simple index.html
echo "hello from apache_demo.com/private" > /var/www/apache_demo.com/private/index.html

# verify symlink created
ls -al /etc/apache2/sites-enabled/

# test apache working
systemctl restart apache2
curl localhost
```

Now create `.htpasswd`
```
# set username and password "user1", -c to overwrite
htpasswd -c /etc/apache2/apache_demo.com/.htpasswd user1
```

Edit apache.conf
```
# add basic auth
<Directory /var/www/apache_demo.com/private>
    AllowOverride AuthConfig
    AuthType Basic
    AuthName "Password protected"
    AuthUserFile /etc/apache2/apache_demo.com/.htpasswd
    Require valid-user
</Directory>
```
Verify it
```
curl localhost/private -u user1:user1
```

## Restrict access to group members <a name="restrict_group"></a>
Enable a required module
```
a2enmod authz_groupfile
```

Create a group and add an user to the group
```
groupadd apache_group

useradd apache_user1 -G apache_group

# verify it
id apache_user1
```

Add the group user to `.htpasswd`
```
# append by not specifying -c option
htpasswd /etc/apache2/apache_demo.com/.htpasswd apache_user1
```

Add the group user to `.apache_group`
```
echo "apache_group: apache_user1" > /etc/apache2/apache_demo.com/.apache_group
```

Now modify apache.conf
```
# create private dir for group
<Directory /var/www/apache_demo.com/group>
  AllowOverride AuthConfig
  AuthType Basic
  AuthName "Password protected"
  AuthUserFile "/etc/apache2/apache_demo.com/.htpasswd"
  AuthGroupFile "/etc/apache2/apache_demo.com/.apache_group"
  Require valid-user
</Directory>
```
Notice the two lines below - you need both AuthUserFile `.htpasswd` and AuthGroupFile `.apache_group`
```
AuthUserFile "/etc/apache2/apache_demo.com/.htpasswd"
AuthGroupFile "/etc/apache2/apache_demo.com/.apache_group"
```

Also create an index.html for the group
```
# create a group dir
mkdir /var/www/apache_demo.com/group

# create a simple index.html
echo "hello from apache_demo.com/group" > /var/www/apache_demo.com/group/index.html
```

Test and reload the apache server
```
apache2ctl configtest
service apache2 restart
```

Hit the group-restricted endpoint
```
curl localhost/group -u apache_user1:apache_user1 -L
```
Should return
```
hello from apache_demo.com/group
```

If you add a new user to `.htpasswd` but not in `.apache_group`, then the user can't be authorized to access the endpoint
```
htpasswd /etc/apache2/apache_demo.com/.htpasswd apache_user2

url localhost/group -u apache_user2:apache_user2 -L
```
And you will get
```
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
```


## Add TLS/HTTPS support <a name="https_support"></a>
Enable module
```
a2enmod ssl
```

Create a self-signed cert and a private key
```
openssl req -x509 -days 365 -newkey rsa:2048 -keyout /etc/apache2/apache_demo.com/self.key -out /etc/apache2/apache_demo.com/self.crt
```

Create a virtual host and add paths to the cert and key
```
<VirtualHost *:443>
	ServerName www.apache_demo.com:443
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/apache_demo.com

  SSLCertificateFile /etc/apache2/apache_demo.com/self.crt
  SSLCertificateKeyFile /etc/apache2/apache_demo.com/self.key
</VirtualHost>
```

Test and reload the apache server
```
apache2ctl configtest
service apache2 restart
```

Hit the Https endpoint to verify it
```
curl https://localhost:443 -k
hello from example.com in apache
```


## What's next (Nginx) <a name="whats_next></a>
I have another [Nginx demo](https://github.com/hasakura12/nginx-demo) (reverse proxy, load balancing, Nginx basics, HTTPs support, and more).

Also [AWS EKS (Elastic Kubernetes Service) demo](https://github.com/hasakura12/aws-eks-demo).