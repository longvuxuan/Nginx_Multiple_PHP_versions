# Nginx + Mysql + Multiple PHP Version on MacOS + Magento 2

Use brew to install all the components:
- Nginx + Server Blocks (Virtualhost)
- PHP versions: 8.1, 7.3 and 7.4
- Dnsmasq
- Mysql + phpmyadmin
- Mkcert
- Mailhog
- Elasticsearch
- Xdebug
- SSH Keys

## Installation

### BREW

Install xcode tools and brew.

```sh
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew analytics off
brew install openssl
brew install wget
brew doctor
```

### Nginx

Install and configure nginx

```sh
brew install nginx
sudo vi /usr/local/etc/nginx/nginx.conf
whoami
```

Get current user from whoami command and edit the user line in the nginx.conf file.
Uncomment user line:

```sh
user <user_from_whoami> staff;
```
Add more configurations inside http {} section

```sh
http {
......
    # allow for many servers
    server_names_hash_bucket_size 512;
    client_max_body_size 100M;
    fastcgi_buffers 16 16k;
    fastcgi_buffer_size 32k;
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    
.....
}
```

Change listen from 8080 to 80 under server{} section
```sh
server {
......
        listen       80;
        .....
        location / {
            root   html;
            index  index.php index.html index.htm;
        }
        
        ....
        location ~ \.php$ {
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9100;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            add_header X-Frame-Options "SAMEORIGIN";
            add_header X-XSS-Protection "1; mode=block";
            add_header X-Content-Type-Options "nosniff";
            charset utf-8;
        }
.....
}
```
### Configure Server Blocks (Virtualhost)

Go to servers folder in /usr/local/etc/nginx/servers.

```sh
cd /usr/local/etc/nginx/servers
sudo vi <your_custom_domain>.conf
```

Copy and paste the example content:

```sh
server {
    listen 443 ssl http2;
    
    ssl_certificate /Users/<user>/certs/<your_custom_domain>.pem;
    ssl_certificate_key /Users/<user>/certs/<your_custom_domain>-key.pem;

    server_name  demo.test;

    set $MAGE_ROOT <your_root_path>;
    include <your_root_path>/nginx.conf.sample;
}
```

Restart nginx
```sh
nginx -t
brew services restart nginx
brew services list

sudo nginx -s reload
```

Go to https://demo.test link for testing the Virtualhost

### PHP versions

Install multiple versions via brew

```sh
brew tap shivammathur/php
brew install shivammathur/php/php@7.3
brew install shivammathur/php/php@7.4
brew install shivammathur/php/php@8.1
```

Change user and group to avoid the issue of permission while running the PHP versions.

```sh
sudo vi /usr/local/etc/php/8.1/php-fpm.d/www.conf

user <user_from_whoami>
group staff

listen = 127.0.0.1:9081

listen.owner = <user_from_whoami>
listen.group = staff
listen.mode = 0660
```

Make it the same for other PHP versions.
Note: Remember the port at listen line. For example,
- PHP 8.1 with port: 9081
- PHP 7.3 with port: 9073
- PHP 7.4 with port: 9074

Change some configurations in the php.ini file

```sh
sudo vi /usr/local/etc/php/8.1/php.ini

memory_limit = -1
max_execution_time = 3000
upload_max_filesize = 200M
post_max_size = 800M
file_uploads = On
```

Restart php
```sh
brew services restart php@8.1
brew services restart php@7.3
brew services list
```

Verify that you have processes running and validate your ports
```sh
sudo lsof -i -n -P | grep php-fpm
```

Optionally, add aliases and replace <your_version> with the version homebrew installs.
```sh
alias php81="/usr/local/opt/php@8.1/bin/php"
alias php73="/usr/local/opt/php@7.3/bin/php"
alias php74="/usr/local/opt/php@7.4/bin/php"
alias mage="bin/magento"
alias mage73="php73 bin/magento"
alias mage74="php74 bin/magento"
alias composer73="php73 /usr/local/bin/composer"
alias composer81="php81 /usr/local/bin/composer"
alias composer74="php74 /usr/local/bin/composer"

# Make switching versions easy
function phpv() {
    brew unlink php
    brew link --overwrite --force "php@$1"
    export PATH="/usr/local/opt/php@$1/bin:$PATH" >> ~/.zshrc
    export PATH="/usr/local/opt/php@$1/sbin:$PATH" >> ~/.zshrc
    source ~/.zshrc
    php -v
}
export PATH="/usr/local/sbin:$PATH"
```

To switch between PHP version:
```sh
phpv 7.4
or
phpv 8.1
```

### Dnsmasq

Save yourself the fuss of editing your hosts file constantly you can use dnsmasq.

```sh
brew install dnsmasq
```

Then set up a custom hosts TLD *.x  (or other hosts TLD like .test) that point to 127.0.0.1.
```sh
echo 'address=/.test/127.0.0.1' > /usr/local/etc/dnsmasq.conf

sudo brew services start dnsmasq
sudo mkdir -v /etc/resolver 
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/test'

ping demo.test
```

### MailHog

Install it to test emails on the local site.
```sh
brew install mailhog
brew services start mailhog

# start with
mailhog
```

Now, you can access MailHog at http://localhost:8025/. However, you still need to connect MailHog to PHP and the mail mac command used by Postfix (Postfix comes with macOS 12 Monterey).

```sh
sudo vi /etc/postfix/main.cf
```

Add the following to the end of the file to connect MailHog to Postfix.

```sh
# MailHog
myhostname = localhost
relayhost = [127.0.0.1]:1025
```
Send a test email and check MailHog.

```sh
echo "Test email from Postfix" | mail -s "Test Email" hi@example.com
```

Next, update each php.ini file with the following, if you have multiple versions of PHP, and then restart php-fpm.

```sh
sudo vi /usr/local/etc/php/8.1/php.ini

# Edit sendmail line
sendmail_path = /usr/local/opt/mailhog/bin/MailHog sendmail test@localhost
```


### Mysql + PhpMyadmin

Install mysql and phpmyadmin

```sh
brew install mysql
brew services start mysql
brew services list
```

Edit the my.cnf file

```sh
sudo vi /usr/local/etc/my.cnf

# Default Homebrew MySQL server config
[mysqld]
# Only allow connections from localhost
bind-address = 127.0.0.1
mysqlx-bind-address = 127.0.0.1

# Add mode only if needed
sql_mode = "ONLY_FULL_GROUP_BY,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
```

Now, secure using the password password and then restart.
```sh
mysql_secure_installation
brew services restart mysql

mysql -uroot -ppassword
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';

brew services restart mysql
```
### PHPMYADMIN

Install and symlink phpmyadmin

```sh
brew install phpmyadmin

ln -sfv /usr/local/share/phpmyadmin /usr/local/var/www/
```

Go to http://localhost/phpmyadmin link for testing again.

### MKCERT

Install mkcert on the local to use https
```sh
brew install mkcert

mkcert -install

mkdir -p ~/certs

cd ~/certs

mkcert damda.skus.test
```

### Elasticsearch

Install the Elasticsearch

```sh
brew tap elastic/tap
brew install elastic/tap/elasticsearch-full
```

### Xdebug

Install Xdebug extension

```sh
brew link --overwrite --force php@7.3
pecl uninstall -r xdebug 
pecl install xdebug


brew link --overwrite --force php@7.4
pecl uninstall -r xdebug 
pecl install xdebug

brew link --overwrite --force php@8.1
pecl uninstall -r xdebug
pecl install xdebug
```

Edit the php.ini file

```sh
sudo vi /usr/local/etc/php/8.1/php.ini

[xdebug]
zend_extension="xdebug.so"
xdebug.remote_handler=dbgp
xdebug.log_level=0
xdebug.start_with_request=yes
xdebug.idekey=PHPSTORM
xdebug.mode=debug
xdebug.discover_client_host=1
xdebug.client_port=9001
```

Restart php
```sh
brew services restart php@8.1
brew services restart php@7.3
brew services list
```

### Configure Bitbucket or Github with multiple SSH Keys

Generate the SSH keys by running command:
```sh
ssh-keygen
```

Add the public keys to the accounts on the Bitbucket.

Configure the ssh config file in `~/ssh/config`.

```sh
vi ~/.ssh/config

Host bitbucket_work
     HostName bitbucket.org
     User git
     IdentityFile ~/.ssh/id_rsa_work
     IdentitiesOnly yes

Host bitbucket_personal
     HostName bitbucket.org
     User git
     IdentityFile ~/.ssh/id_rsa_personal
     IdentitiesOnly yes
```

When you clone the repositories on the Bitbucket.org, you only need to change the host again.

```sh
For example:

git clone git@bitbucket.org:*****/*****.git

To:

=> for Work
git clone git@bitbucket_work:*****/*****.git

=> for Personal
git clone git@bitbucket_personal:*****/*****.git

```

Notice: use `ssh-copy-id` command with -i parameter to determine default ssh key file.

```sh
ssh-copy-id -i ~/.ssh/id_rsa.pub username@hostname
```

DONE !!!


















