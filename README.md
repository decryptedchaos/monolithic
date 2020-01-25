# Monolithic Game Download Cache Docker Container

## Introduction

This docker container provides a caching proxy server for game download content. For any network with more than one PC gamer in connected this will drastically reduce internet bandwidth consumption. 

The primary use case is gaming events, such as LAN parties, which need to be able to cope with hundreds or thousands of computers receiving an unannounced patch - without spending a fortune on internet connectivity. Other uses include smaller networks, such as Internet Cafes and home networks, where the new games are regularly installed on multiple computers; or multiple independent operating systems on the same computer.

This container is designed to support any game that uses HTTP and also supports HTTP range requests (used by Origin). This should make it suitable for:

 - Steam (Valve)
 - Origin (EA Games)
 - Riot Games (League of Legends)
 - Battle.net (Hearthstone, Starcraft 2, Overwatch)
 - Frontier Launchpad (Elite Dangerous, Planet Coaster)
 - Uplay (Ubisoft)
 - Windows Updates

This is the best container to use for all game caching and should be used for Steam in preference to the lancachenet/steamcache and lancachenet/generic containers.

## Usage

You need to be able to redirect HTTP traffic to this container. The easiest way to do this is to replace the DNS entries for the various game services with your cache server.

You can use the [lancache-dns](https://hub.docker.com/r/lancachenet/lancache-dns/) docker image to do this or you can use a DNS service already on your network see the [lancache-dns github page](https://github.com/lancachenet/lancache-dns) for more information.

For the cache files to persist you will need to mount a directory on the host machine into the container. You can do this using `-v <path on host>:/data/cache`. You can do the same with a logs directory as well if you want logs to be persistent as well.

Run the container using the following to allow TCP port 80 (HTTP) and to mount `/cache/data` directory into the container.

```
docker run \
  --restart unless-stopped \
  --name lancache \
  --network host \
  -v /cache/data:/data/cache \
  -v /cache/logs:/data/logs \
  lancachenet/monolithic:latest
```

>You will notice, that there are no ports specified in this command, that's because we have changed the container to the 'host' network.

Unlike lancachenet/generic this service will cache all cdn services defined in the [uklans cache-domains repo](https://github.com/uklans/cache-domains) so multiple instances are not required.

## Simple Full Stack startup

To initialise a full caching setup with dns and sni proxy you can use the following script as a starting point:
```
export HOST_IP=`hostname -I | cut -d' ' -f1`
docker run --restart unless-stopped --name lancache-dns --detach -p 53:53/udp -e USE_GENERIC_CACHE=true -e LANCACHE_IP=$HOST_IP lancachenet/lancache-dns:latest
docker run --restart unless-stopped --name lancache --network host --detach -v /cache/data:/data/cache -v /cache/logs:/data/logs   lancachenet/monolithic:latest
docker run --restart unless-stopped --name sniproxy --detach -p 443:443 lancachenet/sniproxy:latest
echo Please configure your router/dhcp server to serve dns as $HOST_IP
```
Please check that `hostname -I` returns the correct IP before running this snippet


## Host network configuration  (Workaround) 

In my findings it is better to use host network for the lancache server rather than port mapping because by default docker bridge does not NAT internet traffic to docker hosts,  one could argue that we could simply modify the docker bridge network to NAT traffic,  However, why re-invent the wheel?  We can get direct network access for this continer by using host network

``docker exec -it lancache /bin/bash`` 

You will drop to a prompt

root@lancache:/scripts# 

From here the setup is realitivly stragith forward.  

Optionally We can list the avaliable IPs to the container with the following command

``ip addr``  **Note which IP you want to use for LANCACHE_IP**

First lets install nano for simplicity sake.

``apt -y install nano`` 

``nano /etc/nginx/sites-available/10_generic.conf``

At the top of this file you will see the listen line 

listen 80 reuseport; 

We need to change this,  Where the normal guide would tell you to define a host IP and port to "Map" to the container we now have direct access to the IPs on the host, so we can bind to it directly.  In my case i use the local IP 192.168.1.40 for my lancache server. **(Yours will be different)** So i set this: 

listen 192.168.1.40:80 reusport; 

ctrl+x to save 

``service nginx restart``  

Your lancache server will now have full network access and should not timeout anymore. 


## LanCache-DNS Config

W.I.P It has the same problem as the monolitic container that it does not have external network access and can't lookup remote addresses.  I will work on a guide to fixing it as well as a guide to an alterntive manual solution with unbound below (soon)

## Manual Unbound DNS (ALTERNITIVE TO LANCACHE-DNS) 
## You will need a VM (Docker, Xen, LXC) I choose to run a LXC container with CentOS 
```
yum install git
yum install jq
yum install unbound
```

```
git clone https://github.com/uklans/cache-domains.git
cd cache-domains/

cd scripts/

mv config.example.json config.json
```

## Change IP for all entries to your LANcache IP
```
nano config.json
```
## Now we need to make a few simple edits to the script. 
```
nano create-unbound.sh 
```
## Make your settings match the , this way when the script runs it will automatically output to the right config directory under /etc/unbound/local.d/  When unbound starts it parses all files in this directory.
```
basedir=".."
outputdir="/etc/unbound/local.d/"
path="${basedir}/cache_domains.json"
```

## Now just run the script, it will create the required zone files and put them in the correct place   
```
./create-unbound.sh
```
## Now Restart unbound  and when you query any record for lancache unbound will answer with the LANcache IP.
```
systemctl restart unbound
```

## Changing from lancachenet/steamcache and lancachenet/generic

This new container is designed to replace an array of steamcache or generic containers with a single monolithic instance. However if you currently run a steamcache or generic setup then there a few things to note.

1) Your existing cache files are NOT compatible with lancachenet/monolithic, unfortunately your cache will need repriming
2) You do not need multiple containers, a single monolithic container will cache ALL cdns without collision
3) lancachenet/monolithic should be compatible with your existing container's env vars so you can use the same run command you currently use, just change to lancachenet/monolithic


## Origin and SSL

Some publishers, including Origin, use the same hostnames we're replacing for HTTPS content as well as HTTP content. We can't cache HTTPS traffic, so if you're intercepting DNS, you will need to run an SNI Proxy container on port 443 to forward on any HTTPS traffic.

```
docker run \
  --restart unless-stopped \
  --name sniproxy \
  -p 443:443 \
  lancachenet/sniproxy:latest
```

Please read the [lancachenet/sniproxy](https://github.com/lancachenet/sniproxy) project for more information.

## DNS Entries

You can find a list of domains you will want to use for each service over on [uklans/cache-domains](https://github.com/uklans/cache-domains). The aim is for this to be a definitive list of all domains you might want to cache.

## Suggested Hardware

Regular commodity hardware (a single 2TB WD Black on an HP Microserver) can achieve peak throughputs of ~30MB/s using this setup (depending on the specific content being served).  This would be suitable for very small LANs (<10 people).  For any sort of hosting for LAN sizes above this, it is thoroughly recommended that your storage backend is entirely SSD or NVMe.

## Changing Upstream DNS

If you need to change the upstream DNS server the cache uses, these are defined by the `UPSTREAM_DNS` environment variable. The defaults are Google DNS (8.8.8.8 and 8.8.4.4).

```
 UPSTREAM_DNS 8.8.8.8 8.8.4.4
```

You can override these using the `-e` argument to docker run and specifying your upstream DNS servers. Multiple upstream dns servers are allowed,  separated by whitespace.

```
-e UPSTREAM_DNS="1.1.1.1 1.0.0.1"
```

## Tweaking Cache sizes

Two environment variables are available to manage both the memory and disk cache for a particular container, and are set to the following defaults.
```
CACHE_MEM_SIZE 500m
CACHE_DISK_SIZE 1000000m
```

In addition, there is an environment variable to control the max cache age.

```
CACHE_MAX_AGE 3560d
````

You can override these at run time by adding the following to your docker run command.  They accept the standard nginx notation for sizes (m) and durations (m/h/d).

```
-e CACHE_MEM_SIZE=4000m -e CACHE_DISK_SIZE=1000g -e CACHE_MAX_AGE=3560d
```

CACHE_MEM_SIZE relates to the memory allocated to NGINX for the cache manager process.  1 megabyte will hold around 8000 cache entries, and for Monolithic slicing in 1MB slices, this means each 1M allocated will service around 8GB on disk.  The default size of 500MB should allow you to have a cache of up to 4TB quite comfortably.  Any other available memory in your cachebox should then be used for the Filesystem cache.

## Tuning your cache
Steam in particular has some inherent limitations caused by strict adherence to the HTTP spec connection pool. As such Steam's download speed is highly dependent on the latency between your server and the Steam cdn servers. In the event you find your initial download speed with the default settings is slow this can be resolved by allocating more IP's to your cache. We suggest adding one IP at a time to see how much gain can be had (4 seems to work for a number of people).
### Step 1: Adding IP's to your docker host
Consult your OS documentation in order to add additional IP addresses onto your docker cache host machine
### Step 2: Adding IP's to your cache container
In order for this to work you need to add the port maps onto the relevant cdn container (for example steam). 
* If you are using `lancachenet/monolithic` then using `-p 80:80` should be sufficient as per the documentation. 
* If you are using `lancachenet/generic` or `lancachenet/steamcache` then add multiple `-p <IPadddress>:80:80` for each IP you have added. For example `-p 10.10.1.30:80:80 -p 10.10.1.31:80:80`

### Step 3: Informing lancache-dns of the extra IP's
Finally we need to inform lancache-dns that STEAM is now available on multiple IP addresses. This can be done on the command line using the following command `-e STEAMCACHE_IP="10.10.1.30 10.10.1.31"`. Note the quotes surrounding the multiple IP addresses.
### Step 4: Testing
Choose a game which has not been seen by the cache before (or clear your `/data/cache` folder) and start downloading it. Check to see what the maximum speed seen by your steam client is. If necessary repeat steps 1-3 with additional IPs until you see a download equivalent to your uncached steam client or no longer see an improvement vs the previous IP allocation.

## Monitoring

Access logs are written to /data/logs. If you don't particularly care about keeping them, you don't need to mount an external volume into the container.

You can tail them using:

```
docker exec -it lancache tail -f /data/logs/access.log
```

If you have mounted the volume externally then you can tail it on the host instead.

## Advice to Publishers

If you're a games publisher and you'd like LAN parties, gaming centers and other places to be able to easily cache your game updates, we reccomend the following:

 - If your content downloads are on HTTPS, you can do what Riot have done - try and resolve a specific hostname. If it resolves to a RFC1918 private address, switch your downloads to use HTTP instead.
 - Try to use hostnames specific for your HTTP download traffic.
 - Tell us the hostnames that you're using for your game traffic. We're maintaining a list at [uklans/cache-domains](https://github.com/uklans/cache-domains) and we'll accept pull requests!
 - Have your client verify the files and ensure the file they've downloaded matches the file they **should** have downloaded. This cache server acts as a man-in-the-middle so it would be good to ensure the files are correct.

 If you need any further advice, please contact us and we'll be glad to help!

## Frequently Asked Questions

If you have any questions, please check [our FAQs](faq.md). If this doesn't answer your question, please raise an issue in GitHub.

## How to help out
To build just run `docker build --tag lancachenet/monolithic:testing .`. To test you can run `./run_tests.sh`

If you want to test a new build with a forked uklans repo you can specify a combination of `-e CACHE_DOMAINS_REPO="https://github.com/vibroaxe/cache-domains.git" -e CACHE_DOMAINS_BRANCH="testing"` or even `-v <your repo here>:/data/cachedomains -e NOFETCH=true` if using a locally bind mounted git repo.

## Thanks

 - Based on original configs from [ansible-lanparty](https://github.com/ti-mo/ansible-lanparty).
 - Everyone on [/r/lanparty](https://reddit.com/r/lanparty) who has provided feedback and helped people with this.
 - UK LAN Techs for all the support.

## License

The MIT License (MIT)

Copyright (c) 2019 Jessica Smith, Robin Lewis, Brian Wojtczak, Jason Rivers, James Kinsman

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
