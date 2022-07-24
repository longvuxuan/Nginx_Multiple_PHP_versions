# Step by step to install a new instance of Magento 2 on the local

Environments:
- Nginx
- PHP: 8.1, 7.3 or 7.4
- Dnsmasq
- Mysql + phpmyadmin
- Mkcert

## 1. Create a new SSL for the site

Use `mkcert` to create the SSL for custom domain. You can create the folder anywhere that will store the .pem files.

Here, I will create a folder like "certs" in `/Users/<your-user-name>/certs` and create the SSL for custom domain like `magento.test`

```sh
mkdir -p /Users/<your-user-name>/certs
mkcert magento.test
```

## 2. Configure Server Blocks in Nginx

It is the same Virtualhost in Apache. We need to create them in Nginx, too.
The `.conf` file format like: `<your_custom_domain>.conf`.

For example, I will create it for above custom domain.

```sh
cd /usr/local/etc/nginx/servers/
sudo vi magento.test.conf
```

The default root folder of Nginx: `/usr/local/var/www/`.
You can change it to any folders as you want.

Copy and paste the example content:

```sh
server {
    listen 443 ssl http2;
    
    ssl_certificate /Users/<your-user-name>/certs/<your_custom_domain>.pem;
    ssl_certificate_key /Users/<your-user-name>/certs/<your_custom_domain>-key.pem;

    server_name  <your_custom_domain>;

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

## 3. Import the database

Use command to import .gz file to the database

```sh
gunzip -c <your_path_to_gz_file> | sed -e 's/DEFINER=[^*]*\*/\*/g' | mysql -u <your_database_user_name> -p <your_database_name>
```

Update the configurations in the `core_config_data` table with `<your_custom_domain>` URL and Elasticsearch, too.

The default configuration of Elasticsearch:
`port: 9200`
`host: localhost`

## 4. Configure nginx.conf.sample file in Magento

Each Magento version will run different PHP version. We need to point it to run correctly PHP version.

Edit the `nginx.conf.sample` file in the Magento root folder.

Find `fastcgi_backend` and replace it with `127.0.0.1:<your_php_version_port>`.

You can find php version and port in the link:
https://github.com/longvx2017/Nginx_Multiple_PHP_MacOS#php-versions

For example: I set up the Magento 2.3 with PHP 7.3 and port 9173: `127.0.0.1:9173`

```sh
Replace fastcgi_backend to 127.0.0.1:9173
```

- To run the composer, we need to point the correct PHP version.

As above example with Magento 2.3 and php 7.3. To run the composer command we will:

```sh
which composer
=> /usr/local/bin/composer

For PHP 7.3:
/usr/local/opt/php@7.3/bin/php /usr/local/bin/composer --version

For PHP 7.4:
/usr/local/opt/php@7.4/bin/php /usr/local/bin/composer --version

For PHP 8.1:
/usr/local/opt/php@8.1/bin/php /usr/local/bin/composer --version
```

You can change it with PHP version as you want.

Go to: https://magento.test/ on your browser to see the results.

















