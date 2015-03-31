## Speed up REAL hello WORLD application (Laravel 5)

## Preface

This guide will help you to build environment that could be used for profiling performance of your Laravel 5 application and show a few examples of how to use it in practice.
We will build Laravel 'Hello World' application, make it faster and make it even faster with deeper optimizations of Laravel framework itself.

The purposes of this article are

* Show and remind a few simple steps that will help you to optimize your application
* Measure how these changes actually impact the application and how you can profile your applications
* Make a few step-by-step performance enhancements to Laravel 5, encourage and inspire you to research and contribute to Laravel

## Prerequisites

I will use Ubuntu Server 14.04 (LTS) on Virtual Box. The virtual machine configuration is

* 2048 RAM
* 8 GB HDD
* 2 Cores CPU

Installing Ubuntu is a 'click next' process so I won't describe it here. I will use default settings for everything except selecting additional software. OpenSSH server will be installed however it's only for my convenience and you can skip selecting it.

## Install server software

When Ubuntu is installed we will add PHP, web server (NGINX), profiling tools.

### Web server and PHP

```
neomerx@L5:~$ sudo apt-get update && sudo apt-get dist-upgrade && sudo apt-get install -y nginx php5 php5-fpm php5-cli php5-json php5-mcrypt php5-curl php-pear git curl apache2-utils && sudo php5enmod -s ALL mcrypt
```

### Install composer

```
neomerx@L5:~$ sudo curl -sS https://getcomposer.org/installer | php && sudo mv composer.phar /usr/local/bin/composer
```

### Install profiling tool

I will use Blackfire. That's a nice tool with friendly UI and sharing profile reports feature. You can find more at [https://blackfire.io/](https://blackfire.io/)

```
neomerx@L5:~$ wget -O - https://packagecloud.io/gpg.key | sudo apt-key add - && echo "deb http://packages.blackfire.io/debian any main" | sudo tee /etc/apt/sources.list.d/blackfire.list && sudo apt-get update && sudo apt-get install blackfire-agent && sudo apt-get install blackfire-agent && sudo blackfire-agent --register && sudo /etc/init.d/blackfire-agent start && sudo apt-get install blackfire-agent && blackfire config && sudo apt-get install blackfire-php
```

During installation you will be asked for server ID, server token, client ID and client token which could be found at [https://blackfire.io/account/credentials](https://blackfire.io/account/credentials) (registration required (which is free at the moment of writing)).

### Tweaking Linux Kernel

Note: Further Linux kernel, php and NGINX performance and security settings are not 'best practices'. Consider before applying to other projects.

```
neomerx@L5:~$ sudo nano /etc/sysctl.conf
```

Append the following

>net.ipv4.tcp_fin_timeout = 5  
>net.ipv4.tcp_rfc1337 = 1  
>net.ipv4.tcp_keepalive_intvl = 5  
>net.ipv4.tcp_keepalive_probes = 3  
>net.ipv4.tcp_keepalive_time = 15  
>net.ipv4.tcp_sack = 0  
>net.core.netdev_max_backlog = 2000  
>net.core.somaxconn = 4096  

Save and exit (Ctrl+O, Ctrl+X)

### Install Laravel 5

```
neomerx@L5:~$ composer create-project laravel/laravel --prefer-dist ~/laravel && chmod a+w -R ~/laravel/storage/ && chmod a+w ~/laravel/vendor/
```

### Set up NGINX

```
sudo nano /etc/nginx/sites-enabled/default
```

Replace default config with (hint: Ctrl+K removes current line)

>server {  
>	listen 80 default_server;  
>	listen [::]:80 default_server ipv6only=on;  
>	server_name localhost;  
>  
>	root /home/neomerx/laravel/public/;  
>	index index.php;  
>  
>	location / {  
>		try_files $uri $uri/ /index.php?$query_string;  
>	}  
>  
>	location ~* \.php$ {  
>		fastcgi_pass					unix:/var/run/php5-fpm.sock;  
>		fastcgi_param SCRIPT_FILENAME	$document_root$fastcgi_script_name;  
>		fastcgi_index					index.php;  
>		include						fastcgi_params;  
>	}  
>}  

Save and exit (Ctrl+O, Ctrl+X)

### Remember server IP address

```
neomerx@L5:~$ ifconfig | grep "inet addr"
```

### Reboot the system

```
neomerx@L5:~$ sudo reboot
```

## Speed up application

Open in browser http://<server IP address>/ and check that Laravel works correctly.

### First run

#### ab test

We will create a performance baseline for future comparisons when tweaks done. Let's start from all time favorite [Apache HTTP server benchmarking tool](http://httpd.apache.org/docs/2.4/programs/ab.html) 

```
neomerx@L5:~$ ab -c 10 -t 3 http://localhost/
```

After 3 runs I got numbers for Requests per second (RPS): 205.23, 200.98, 201.55

##### Note

ab will tell you there are failed requests. It happens because Laravel welcome screen generates random text and responses differ by length. You should check that all requests failed due to length only. For example

>Concurrency Level:      10  
>Time taken for tests:   3.000 seconds  
>Complete requests:      628  
>Failed requests:        **498**  
>   (Connect: 0, Receive: 0, Length: **498**, Exceptions: 0)  

#### Blackfire test

Save initial profile

```
neomerx@L5:~$ blackfire --slot 1 --samples 10 curl http://localhost/
```

to see the result [at slot 1](https://blackfire.io/slots) Or you can see [mine here](https://blackfire.io/profiles/be7d0588-9454-4a5c-aff8-d36774b8376d/graph)

|Slot                 |Wall Time|IO    |CPU Time|Memory |Network|
|---------------------|---------|------|--------|-------|-------|
|Blackfire #1. Initial|87.6 ms  |260 µs|87.3 ms |3.19 MB|0 B    |

### First optimization

Laravel comes with a few built-in optimizations. Run the following command

```
neomerx@L5:~$ cd ~/laravel/ && php artisan config:cache && php artisan route:cache && php artisan optimize --force
```

Check that the following files were created

* storage/framework/config.php
* vendor/compiled.php
* vendor/routes.php

#### ab test

```
neomerx@L5:~$ ab -c 10 -t 3 http://localhost/
```

After 3 runs I got numbers (RPS): 215.18, 207.91, 210.88

Well we have some improvement but more interesting numbers are at application profile.

#### Blackfire test

```
neomerx@L5:~$ blackfire --slot 2 --samples 10 curl http://localhost/
```

You can see [mine here](https://blackfire.io/profiles/fea1b426-246b-4647-882d-f1c0dc40576e/graph)

|Slot                           |Wall Time|IO     |CPU Time|Memory |Network|
|-------------------------------|---------|-------|--------|-------|-------|
|#1. Initial                    |87.6 ms  |260 µs |87.3 ms |3.19 MB|0 B    |
|#2. More built-in optimizations|60.9 ms  |121 µs |60.8 ms |3.04 MB|0 B    |
|Comparison from #1 to #2       |-27 ms   |-140 µs|-27 ms  |-146 KB|0 B    |

Detailed comparison is [here](https://blackfire.io/profiles/compare/4b84af7b-78ad-490f-a7bb-57f9e4179134/graph)

As you can see Wall Time dropped 31% and CPU Time 30% however we haven't seen such gains in ab test. What it might mean? Overall performance of the application depends not only from application itself but from OS configuration, web server configuration and etc.

Application performance testing gives more accurate view of how application is optimized without unnecessary influence of other parts of the system. For this reason I would rely mostly on application testing alone without ab tool.

### Real Hello World

If you look closer at profile report you might conclude that main slowest parts of the application are

* registering and boot various providers
* Usage of IoC container
* pipelining through middleware and routs (with a rather scary long stack trace)

Before we go further we should remember that we are developing 'Hello World' application and Laravel comes with a ton of features right out of the box. Do we need them all? How do they impact performance of our application? Let's try to answer

* Do we need cookies, session and CSRF protection? Probably not. So remove not needed middleware at app/Http/Kernel.php@$middleware
* While handling the request session is activated and by default is configured in 'file' mode which leads to tons of files in ```storage/framework/sessions/``` folder and IO load. Session cannot be switched off but we can lower impact by setting drivers to 'array' mode in ```.env``` file
* We don't need authentication so remove 'auth' and 'password' routes and auth middleware in controller constructors (```app/Http/Controllers/WelcomeController.php``` and ```app/Http/Controllers/HomeController.php```)
* We will reduce number of loaded providers which we don't use by removing them from ```config/app.php```.

#### Blackfire test

```
neomerx@L5:~$ blackfire --slot 3 --samples 10 curl http://localhost/
```

You can see [mine here](https://blackfire.io/profiles/91b5c46f-91f2-4577-9eaf-0111ce2fc76b/graph)

|Slot                           |Wall Time|IO     |CPU Time|Memory |Network|
|-------------------------------|---------|-------|--------|-------|-------|
|#1. Initial                    |87.6 ms  |260 µs |87.3 ms |3.19 MB|0 B    |
|#2. More built-in optimizations|60.9 ms  |121 µs |60.8 ms |3.04 MB|0 B    |
|#3. Real Hello World           |41.6 ms  |56  µs |41.5 ms |2.80 MB|0 B    |
|Comparison from #2 to #3       |-19 ms   |-65 µs |-19 ms  |-252 KB|0 B    |

Detailed comparison is [here](https://blackfire.io/profiles/compare/a23127ed-20a0-4fff-b307-4e63eff32c61/graph)

#### Summary

Having implemented a few simple steps we improved performance of our 'Hello World' for more than 2 times. Our application supports vital Laravel features such as routing, controllers, views, templates, localization and etc.

Note: I also checked how it impacted RPS and got: 330.92, 289.66, 305.99 which is 50% improvement from stock Laravel 5.

### Do you want even more RPS?

I'm sure you do otherwise you wouldn't be here. Let's install [HHVM](http://hhvm.com/) and see how it works

```
neomerx@L5:~$ sudo apt-get install software-properties-common && sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0x5a16e7281be7a449 && sudo add-apt-repository 'deb http://dl.hhvm.com/ubuntu trusty main' && sudo apt-get update && sudo apt-get install hhvm && sudo service nginx restart
```

Configure NGINX for using HHVM

```
neomerx@L5:~$ sudo nano /etc/nginx/sites-enabled/default
```

And append the following

>server {  
>		listen 8080 default_server;  
>		listen [::]:8080 default_server ipv6only=on;  
>		server_name localhost;  
>  
>		root /home/neomerx/laravel/public/;  
>		index index.php;  
>  
>		location / {  
>				try_files $uri $uri/ /index.php?$query_string;  
>		}  
>  
>		location ~* \.php$ {  
>		fastcgi_keep_conn	on;  
>		fastcgi_pass		127.0.0.1:9000;  
>		fastcgi_param		SCRIPT_FILENAME $document_root$fastcgi_script_name;  
>		include			fastcgi_params;  
>		}  
>}  

Save and exit (Ctrl+O, Ctrl+X)

Restart NGINX

```
neomerx@L5:~$ sudo service nginx restart
```

Open in browser http://<server IP address>**:8080**/ and check that Laravel works correctly.

#### ab test

```
neomerx@L5:~$ ab -c 10 -t 3 http://localhost:8080/
```

After a few runs I got numbers (RPS): 374.73, 682.83, 584.68, **771.33, 729.74, 734.95**

**From 200 RPS to 750 RPS. Not bad huh?**

### Summary

That's the end of 'Speed up application' part. We have played with stock Laravel 5 application, performance tools/options and found simple and effective ways to make our application faster.

In the next part we go deeper to hack Laravel 5 framework using [Blackfire](https://blackfire.io/).