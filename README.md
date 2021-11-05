# docker-nginx-mysql-php-phpmyadmin

## Abstract
In this repository you will find a description how to dockerize nginx, php and mysql. phpMyAdmin is only on board to provide a convenient administration interface.
<br/><br/>

## Intended use
This Docker image is to be used for m< vulnerable web store called "mallow" zthat is built in Python. Until that happens, the repository will serve as a think tank so I don't forget over time how to set up this project.
<br/><br/>

## Structure
Prebuilt Docker images exist in Docker Hub for both phpMyAdmin, mySQL, and nginx. These images are assembled via docker-compose.
<br/><br/>

## Docker Compose File
The docker-compose file assembles the whole application stack. Before we go into detail, let's have a look at the whole file. Be warned - this app builds a vulnerable application, so the dockercompose file has some vulnerabilities itself.

```
version: '3'

services:
    web:
        image: nginx:alpine
        container_name: webserver_mallowz_php
        restart: always
        ports:
            - 8000:80     
        volumes:
            - "./web:/var/www/html"
            - "./etc/nginx/default.conf:/etc/nginx/conf.d/default.conf"
        depends_on:
            - database
            - php

    php:
        image: bitnami/php-fpm
        container_name: php_mallowz_php
        restart: always
        volumes:
            - "./web:/var/www/html"

    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: phpmyadmin_mallowz_php
        restart: always
        ports:
            - 8080:80
        environment: 
            PMA_HOST: mysql_mallowz_php
            PMA_ARBITRARY: 1
            MYSQL_ROOT_PASSWORD: "root"
        depends_on:
            - database

    database:
        image: mysql:5.7.22
        container_name: mysql_mallowz_php
        restart: always
        ports:
            - 8081:3306
        volumes:
            - ./data/db/mysql:/var/lib/mysql
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: mallowz
            MYSQL_USER: mallowz
            MYSQL_PASSWORD: mallowz
```

<br/><br/>
### The Web Server
A nginx server is used as the web server. I also thought about using an Apache web server, but after some research I decided that nginx is the better option. 
The nginx:aplpine image from Docker Hub is used. I give it the name "webserver_mallowz_php" so that I can find the corresponding container in Docker later. 
The webserver runs on port 80, this port is forwarded to the outside via port 8000 - so the webserver can be reached under localhost:8000. 
The next section in the configuration deals with the sharing or mounting of some directories of the installation, which should be able to be used and reached 
by the web server. I went the way of including directories on the local machine into the image. This allows me - at least during development - to store files 
locally and still be able to access them via the web server without having to copy files in a complicated way. 
The depends section indicates that the web server should wait for the php and database installation. 
Thus, it specifies the order in which the individual containers must be started. This makes sense, because if php is not yet started, but the web server already wants to 
access Django directories, this can lead to unsightly errors. Here again the section from the compose file for the web server installation. 

```
    web:
        image: nginx:alpine
        container_name: webserver_mallowz_php
        restart: always
        ports:
            - 8000:80     
        volumes:
            - "./web:/var/www/html"
            - "./etc/nginx/default.conf:/etc/nginx/conf.d/default.conf"
        depends_on:
            - database
            - php

```

### php
PHP is used as the CGI interface. I deployed another docker image which will use django and gunicorn as wsgi, maybe you want to have a look at that repository, too. The PHP installation uses the bitnami php image and exposes the volume /var/www/html to the same directory the web server uses.

```
    php:
        image: bitnami/php-fpm
        container_name: php_mallowz_php
        restart: always
        volumes:
            - "./web:/var/www/html"
```

### pypMyAdmin
As mentioned above, the pypMyAdmin instance is only for more convenient maintenance of the database. Instead of having to log in to the container and enter console commands, 
I allowed myself the luxury of accessing the database via a web interface. The official Docker Hub image of phpmyadmin is used, which usually makes the service 
available via port 80. I changed this port to port 8080, so that the installation can now be reached under localhost:8080. PMA_HOST is used to specify the name of 
the database instance that phpMyAdmin should access, which in my case is mysql_mallowz_php. As you can see very nicely, in the Docker Compose file you have to pass a 
password with which you can log in to phpMyAdmin. In my case the password is in plain text in the file - clearly a security hole. Similar to the web server, there is 
also a Depends statement. Here it waits for the successful start of the database instance before phpMyAdmin itself can be started and called.

```
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: phpmyadmin_mallowz_php
        restart: always
        ports:
            - 8080:80
        environment: 
            PMA_HOST: mysql_mallowz_php
            PMA_ARBITRARY: 1
            MYSQL_ROOT_PASSWORD: "root"
        depends_on:
            - database
```

### The mySQL / mariaDB database
The database is also available for download as a ready-made image in Docker Hub. Similar to the web server, a volume was also released here - all mySQL configurations 
and database files are stored on the local computer and included in the image. The access data for the database must be stored in the environment variables. In this case, 
too, these are available in plain text and are thus to be classified as a security vulnerability. A better way is to either set sensitive information as environment 
variables in advance or to use secrects.

```
    database:
        image: mysql:5.7.22
        container_name: mysql_mallowz_php
        restart: always
        ports:
            - 8081:3306
        volumes:
            - ./data/db/mysql:/var/lib/mysql
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: mallowz
            MYSQL_USER: mallowz
            MYSQL_PASSWORD: mallowz
```

# Nginx configuration
I also faced some challenges with nginx. For example, I had to figure out that a config file needs to be created in the first place. However, after I had implemented the basic configuration of the web server correctly, the Django installation would not start. After some experimentation, I came up with this solution:

```
# Nginx configuration

server {
    listen 80 default_server;
    server_name localhost

    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html;
   
    set $virtualdir "";
    set $realdir "";

    location / {
        try_files $uri $uri/ $realdir/index.php?$args;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

The file is - as specified in the dockercompose file - to be saved in the directory "etc/nginx/default.conf"

# Project installation and configuration

1. Open up command prompt
2. Create a new project folder and cd into it
3. Create etc/nginx folder and place default.conf inside it
4. Execute a "docker-compose up -d"
5. Open up the browser and navigate to "localhost:8000". You should see the nginx start page.
