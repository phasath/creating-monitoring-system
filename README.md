# Creating a Monitoring System in 10 minutes

Created by Raphael Sathler, on Dec 21, 2018.


# Why this tutorial?

When I needed to create a monitoring system, I found many tutorials but none of them teached how to do it out of blue to an actual monitoring system. I had to search through lots of tutorials, documentation and reading the source code to understand the basis.
So I decided to share what I discovered so far. This way, nobody would have to go through this again. 


# What is this?

This is a Monitoring System composed from 3 applications:

[Grafana](https://grafana.com/): An open platform for analytics and monitoring. This platform can collect data from many places and show in highly flexibles customizeds dashboards.
[Graphite] (https://graphiteapp.org/): Graphite can store and server metrics. Here is where we'll send our servers data.
[Collectd] (https://collectd.org/): Collectd is a daemon which collects data from the servers and sends it to somewhere. In our case, it will be the graphite instance. There are bunches of plugins that you can make use of to collect almost any desired metric.


# Requirements 

We **will need** docker, so, if you didn't have it, please, install it easily following the [official site](https://www.docker.com/). 
If you want to deploy applications on your own server, you might want to take a look on [dokku](https://github.com/dokku/dokku) which I highly recommend.


# Installation

So, first things first, we'll setup Grafana first, and configure our access.

## Grafana

### Creating volume

Using [docker volume](https://docs.docker.com/storage/volumes/), we will create a permanent storage for our configuration in the host. This will make it easier to configure the stuff we need. 

```
$ docker volume create grafana-storage
$ docker volume create grafana-config-storage
```
### Deploying grafana

Once created, we can deploy our grafana instance on docker. This can be simply done using this command:

```
$ docker run -d -p 3000:3000 --name=grafana -v grafana-storage:/var/lib/grafana-v grafana-config-storage:/etc/grafana grafana/grafana
```

### Checking ports

We don't want to let this port open, so make sure this port has access only through localhost. You can check it using [nmap](https://nmap.org/) and [netstat](https://linux.die.net/man/8/nestat)

`Netstat` will show your ports "internally", and `nmap` will show it "externally", so make sure it will show up on `netstat` but not on `nmap`. If it does appear, I suggest you take a look on your firewall to block it.

```
$ netstat -tupln 
```

```
$ nmap -n localhost
```

### Serving it with Nginx

Once the container is properly deployed, now we'll configure nginx to serve it. You might want to use it on docker, and create a port forwarding to yours server, like, forward the port 80 from your server to the nginx container on docker, or install nginx on your server. That's up to you and I'll not explain how to do so as it's not the purpose of this tutorial. A quickly search on google will give you hundreds of links. I recommend [this one](https://www.digitalocean.com/community/tutorials/how-to-run-nginx-in-a-docker-container-on-ubuntu-14-04)

If you don't know your container IP, you can get it under NetworkSettings using:

```
$ docker inspect grafana
```

The config file is bellow. This should be placed inside `sites-available` - and linked to `sites-enabled` - or `conf.d`, depending on your `nginx` and `os` version.

> You must adjust to your settings with your docker container's ip and your website. 

```
grafana.conf

server {
  listen      [::]:80;
  listen      80;
  server_name [your-website]; 
  access_log  /var/log/nginx/grafana-access.log;
  error_log   /var/log/nginx/grafana-error.log;

  location    / {

    gzip on;
    gzip_min_length  1100;
    gzip_buffers  4 32k;
    gzip_types    text/css text/javascript text/xml text/plain text/x-component application/javascript application/x-javascript application/json application/xml  application/rss+xml font/truetype application/x-font-ttf font/opentype application/vnd.ms-fontobject image/svg+xml;
    gzip_vary on;
    gzip_comp_level  6;
   
    client_max_body_size 100M;
    proxy_pass  http://grafana-3000;
    http2_push_preload on; 
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header X-Request-Start $msec;
  }
}

upstream grafana-3000 {

  server [your-docker-container-ip]:3000;
}
```

Check to see if it works after reloading `nginx`. 

After logging for the first time on `grafana`, you must set a new admin password.

Under the volume created on the host, grafana-config-storage, usually in `/var/lib/docker/volumes`, you'll find the `grafana.ini` configuration file that you must adjust with your personal settings and whatever you use, like smtp and etc.

We'll come back here later. Now, we'll proceed to Graphite.

## Graphite 

The docker setup to graphite is just simply deploy it on docker. Some changes can be done, like using another database and etc. For the purpose of this example, we'll stick with the default configuration.

```
$ docker run -d --name graphite --restart=always -p 81:80 -p 2003-2004:2003-2004 -p 2023-2024:2023-2024 -p 8125:8125/udp -p 8126:8126 graphiteapp/graphite-statsd
```

Note that we are using now the port 81 on our server as we have the port 80 used by nginx.

Again, have the graphite container's ip noted down:

```
$ docker inspect graphite
```

## Collectd

This is the part where we give life to all of this: a Collector Daemon of Metrics to the servers.

You can install it using the package manager of your server. I have created an [auto-install](https://github.com/phasath/collectd_auto_install) for CentOS, Ubuntu and OracleLinux, including the most basic configuration for monitoring a server, like disks, cpu, memory and etc.

In this repository, there's also two templates for grafana to get the metrics of your server.

Follow the instructions from this repository to install Collectd.

The collectd has a config file, located under `/etc` or `/etc/collectd.d` depending upon your OS that holds the configuration for the server. It can be changed to properly reflects your system. The version from this repository is a very wide one to summarize the principal components of a system.

## Setting graphite as datasource

Once all the configuration is done, you must create a datasource on grafana pointing to your graphite server - where the collectd will send the metrics. 
To do that, simply press on `add datasource`, select `graphite` and configure it to point to your container graphite's ip.

## Creating a dashboard

Now you can create a dashboard to display the metrics from graphite. On the repository from the [collectd auto-install](https://github.com/phasath/collectd_auto_install), there are 2 templates. They MUST be imported and changed - specially the disks area - to reflect your servers configuration. 
You can also browser for other dashboards, as long as they use graphite as datasource.

# Issues

If you couldn't understand something, raise an [issue](https://github.com/phasath/creating-monitoring-system/issues/new) and I'll gladly help you. 
