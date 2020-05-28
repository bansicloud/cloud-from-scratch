[ Tested 2020.05.28 ] [ Updated 2020.05.28 ] 

# Cloud From Scratch
Build a self-hosted personal private cloud system from scratch.

You may be surprized to find that building and maintaining your own private cloud system, using your own equipment, hosted in your own home, is not only possible, it's relatively easy and is really fun and rewarding. The following instructions should provide everything you need to build a multi-host capable personal cloud computing platform from scratch. Using primarily off the shelf technologies and inspired by the "Arch Way" (**Simplicity, Modernity, Pragmatism, User centrality, Versatility**) this project aims to require only a small amount of technical attention and allow a wide range of expression and functionality. After completing the following steps you'll have yourself a fully functional personal private cloud system ready to be shaped into the services you wish.  I don't know about you, but I know I'm excited, so let's get started.


# Overview
When completed the system looks something like this.

```
[ EDGE NODE VPS ]                     |               [ LAN NODE ]
                                      |               
    [Wireguard] <---------------------+-------------- [Wireguard]
         ^                            |                   ^
         |                            |                   |
         v                            |                   v
    [Caddy Reverse Proxy]             |         +----------------------------------+
                                      |         | DOCKER                           |
                                      |         |				   |
				      |		+----------------------------------+
				      |		.				   .
                                      |         .    [ ------ Caddy ---------]     .
                                      |         .        |       |        |        . 
                                      |         .        v       |        v        .
                                      |         .       [APP]    |      [APP]      . 
                                      |         .                v                 . 
                                      |         .              [APP]               . 
                                      |         .                                  . 
                                                +. . . . . . . . . . . . . . . . . +
                                               
 
         ** Internet  **                               ** Home Network Cloud **

```

From this platform you'll be able to install, own, operate and access from anywhere, from any device, any cloudable software you choose to install.

Chat * Photos * Calls * File Storage * Music * Notes * Weather * News * Etc

Ok, Let's get started.

## Requirements
For this setup you'll need...
* **Cloudflare Account**
* **Domain Name**
* **Raspbery PI** (v3 or greater, or adopt the instructions to a PC or Virtual Machine)
* **VPS Account** (Linode/Digital Ocean/Vultr)

If you're not already familiar with [Wireguard](https://www.wireguard.com/) and [Docker](https://www.docker.com/) you may want to first familiarize yourself with these as they play core roles in this project. These instructions assume you are comfortable with command line based installation and configuration.  For text editing we'll use vim, feel free to replace vim with the text editor of your choice.

Prefer something a little more automated?  Consider one of [these](#web-gui-automated-run-your-own-cloud-system).

## Sections
* **Provision Edge Node**
  * VPS Instance
  * Wireguard
  * Docker
  * Caddy

* **Setup Domain**
  * Register Domain (or use an existing one)
  * Cloudflare Setup

* **Build a Local Node**
  * Configure OS
  * Wireguard
  * Docker
  * Caddy

* **Example Applications**
  * Ghost
  * GOGS
  * Express

* Troubleshooting
* Optional Configurations
* Discussion
* Links


## Provision Edge Node
The edge node functions as a lightweight, always online, public access gateway, mainly routes traffic, provides a layer of privacy and helps mitigate NAT issues.

Create a VPS instance at your favorate VPS service, like Digital Ocean or Vultr.  (use these affiliate links to support this project: [Digital Ocean](https://digitalocean.com) | [Vultr $100 free credit for 30 days](https://www.vultr.com/?ref=8580218-6G).)

Any tier level with at least **512MB RAM** should be enough.

Create a new instance using **Debian 10** (Buster)

For the purpose of this tutoral we'll assume your edge node ip address is `198.51.100.1`.

Log in via SSH to your new server
```
$ ssh root@your-new-server-ip
```

and update the system
```
$ apt update
$ apt upgrade
```

### Wireguard

("Users with Debian releases older than Bullseye should enable backports." This includes Buster so, we'll do backport.)


Edit sources.list, add buster-backports, install wireguard.

```
$ vim /etc/apt/sources.list.d/sources.list
```

add
```
deb https://deb.debian.org/debian buster-backports main
```

install wireguard
```
$ apt update
$ apt -t buster-backports install "wireguard"

$ reboot
```
Reconnect to server after reboot.

Verify Install
```
$ which wg
```
(should see something like `/usr/bin/wg`)

Generate Wireguard Keys
```
$ cd ~
$ mkdir wg-setup
$ cd wg-setup

$ wg genkey | tee privatekey | wg pubkey > publickey
```
(contents of privatekey or public key file look like `ayQFJiofd+fji2oDi8N/Jfi3=`)

Create wg0.conf file
```
$ vim wg0.conf
```
We're using `10.1.1.1/24` from here forward you may use this or your own preference for the wireguard network.  ListenPort may also be your choice.  Here we'll use `51820`.
```
[Interface]
Address = 10.1.1.1/24
ListenPort = 51820
PrivateKey = your-private-key-text-from-privatekey-genearation=

#[Peer]
##client 1 -- living room
#PublicKey = clients-public-key-from-a-future-step=
#AllowedIPs = 10.1.1.2/32

#[Peer]
##remote pi 1 -- kitchen
#PublicKey = clients-public-key-from-a-future-step= 
#AllowedIPs = 10.1.1.3/32
```

Copy the `~/wg-setup/wg0.conf` file to `/etc/wireguard/wg0.conf`
```
$ cp wg0.conf /etc/wireguard/wg0.conf
```

We'll come back and fill out clients-public-key in the `/etc/wireguard/wg0.conf` file in an upcomming step.

Start Wireguard
```
$ wg-quick up wg0
```

Should see something like
```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.1.1.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

Verify Looks good
```
$ wg
```
should see someting like
```
interface: wg0
  public key: sf8fjDJj9fiJaawQdfiDFi8sfjkl+fxXIxf/=
  private key: (hidden)
  listening port: 51820
```

Enable Wireguard to start automatically at boot
```
$ systemctl enable wg-quick@wg0.service
```

Reboot to verify its working.
```
$ reboot
```

Verify
```
$ systemctl status wg-quick@wg0
```

Test
```
$ ip a ... (should show wg0 interface)
$ wg   ... (should show client config)
```


### Docker
https://docs.docker.com/engine/install/debian/

Install Docker
```
$ apt install gnupg2 software-properties-common
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
$ add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
$ apt update
$ apt install docker-ce docker-ce-cli containerd.io
```

Test (this should download and run docker hello world image)
```
$ docker run hello-world  
```


### Caddy
https://hub.docker.com/_/caddy


Create Caddyfile
```
$ vim ~/Caddyfile
example.com:80 {
        #tls cert@example.com

	#reverse_proxy 10.1.1.2:80
	
	respond "It Works!!"
        }
```
(we'll replace `example.com` with your domain later in the domain step)

Create a docker network called `master`
```
$ docker network create -d bridge master
```

Create Caddy file locations
```
$ mkdir ~/certs
$ mkdir -p ~/www/example.com
```

Start Caddy
```
$ docker run -d --restart=always --name=caddy_web_server -p 80:80 -p 443:443 --network=master -v /root/Caddyfile:/etc/caddy/Caddyfile -v /root/www:/usr/share/caddy -v /root/certs:/data caddy
```

Check
```
$ docker ps -a
```
should see something like
```
eb631324c3b9        caddy               "caddy run --config …"   22 seconds ago      Up 21 seconds              0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 2019/tcp   caddy_web_server
```

## Setup Domain
### Domain Name
Use your own existing domain or register a new one.  | [namecheap](https://namecheap.com) -- support this project by using this affiliate link.
	
### Cloudflare
Use [CloudFlare](https://cloudflare.com) as your name server (set your domain name name servers to the nameserver names your cloudflare account instructs.)

**DNS**
Configure an A record to point to the IP of your VPS.  **WITH cloudflare proxy** enabled. eg `A @ 198.51.100.1`
(replace `example.com` with your domain name and `198.51.100.1` is an example address, use the ip address of your edge node whenever you see the `198.51.100.1` address.)

Configure cloudflare a CNAME record `edge.example.com` (replace `.example.com` with your domain name eg `edge.yourdomain.com`) point it to your domain name **WITHOUT cloudflare proxy (click to make a grey cloud)**. eg `CNAME edge example.com`

**SSL/TLS**
Set to **Full (strict)** Option  (otherwise you may get too many an redirects error.)
  
Modify Edge Node Caddyfile
```
$ vim ~/Caddyfile
example.com:80 {
        #tls certs@example.com

        #reverse_proxy 10.1.1.2:80
	respond "It Works!!"
                }
```
Replace `example.com`s with your domain name.  Replace 'certs@example.com' with your domain's admin email address.

In future references to `example.com` replace `example.com` with your domain name.

Restart Caddy
```
$ docker restart caddy_web_server
```

Check your domain
```
$ curl -v http://example.com
```
should see something like
```
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Thu, 28 May 2020 21:56:06 GMT
< Content-Length: 10
< Connection: keep-alive
< Set-Cookie: __cfduid=d7b74165a55e4ebe027d2a259181e53731590702966; expires=Sat, 27-Jun-20 21:56:06 GMT; path=/; domain=.example.com; HttpOnly; SameSite=Lax
< CF-Cache-Status: DYNAMIC
< cf-request-id: 02fee21e7b0000920a16007200000001
< Server: cloudflare
< CF-RAY: 59ab3943faf7920a-EWR
< 
* Connection #0 to host adammitchell.org left intact
It Works!!
```
if not, check your Caddyfile then restart caddy.
if still not... perhaps the caddy log will help...
```
$ docker logs caddy_web_server
```

Enable https
```
$ vim Caddyfile
```

Remove `:80` and uncomment your email address.
```
example.com {
        tls certs@example.com

        #reverse_proxy 10.1.1.2:80
	respond "It Works!!"
                }
```

Restart Caddy
```
$ docker restart caddy_web_server
```

Make sure Certificate obtained successfully (use ctrl+c to return to term)
```
$ docker logs -f caddy_web_server
```

Check your domain
```
$ curl -v https://example.com
```
Should see It Works!! and TLS handshakes.


## Local Node
Local nodes live within your home network.  In this system local nodes are pretty much where everything lives and happens.  Cloud systems can be built from one or more hosts, but to keep things simple we'll start out with just one node, a Raspberry Pi.

Get Pi

Pi (Amazon)

SD Card (Amazon)

Install Raspbian
Note: if you don't have a Pi, you could use a virtual machine, or laptop if so note you may need to adapt these instructions to your situation.

Download (Raspbian Buster Lite at the time of this tutorial)

Create the bootable sdcard

**Be careful with this step!  Be sure of your of=/dev/(yourdevhere) link.**
```
$ unzip 2020-02-13-raspbian-buster-lite.zip
$ sudo dd if=2020-02-13-raspbian-buster-lite.img of=/dev/(yourdevicehere) bs=4M status=progress conv=fsync
```
**or** in one step
```
$ sudo unzip -p 2020-02-13-raspbian-buster.zip | dd if=2020-02-13-raspbian-buster-lite.img of=/dev/(yourdevhere) bs=4M status=progress conv=fsync
```

Enable ssh: place a file named ssh, without any extension, onto the boot partition before booting.
 
login

**user** pi

**password** raspberry

Initial Configuration
```
$ sudo raspbi-config
- set localization / keyboard
- set wifi connection (if not using ethernet)
- enable ssh [if not already enabled from boot ssh file]
```

Update system
```
$ sudo apt update
$ sudo apt upgrade
```

Wireguard

Check if already installed
```
$ which wg
```

If not try install
```
$ sudo apt install wireguard
```

If not try...
```
$ sudo apt-get install raspberrypi-kernel-headers

$ echo "deb http://deb.debian.org/debian/ unstable main" | sudo tee --append /etc/apt/sources.list.d/unstable.list

$ wget -O - https://ftp-master.debian.org/keys/archive-key-$(lsb_release -sr).asc | sudo apt-key add -

$ printf 'Package: *\nPin: release a=unstable\nPin-Priority: 150\n' | sudo tee --append /etc/apt/preferences.d/limit-unstable

$ sudo apt update

$ sudo apt install wireguard 

$ reboot
```
Verify
```
$ which wg
```

Generate Wireguard Keys
```
$ cd ~
$ mkdir wg-setup
$ cd wg-setup
$ wg genkey | tee privatekey | wg pubkey > publickey
```

Create wg0.conf
```
$ vim wg0.conf
[Interface]
Address = 10.1.1.2/24
PrivateKey = xxxxx-your-private-key-here-xxxxx=

[Peer]
Endpoint = edge.example.com:51820
PublicKey = xxxxx-your-SERVERS-public-key-here-xxxxx=
AllowedIPs = 10.1.1.1/32, 10.1.1.2/32, 10.1.1.0/24
PersistentKeepalive = 25
```

IMPORTANT KEY PLACEMENTS
Replace `edge.exmaple.com` with your `edge.yourdomain.com` setting from the cloudflare domain setting step.

Copy the **PUBLIC** key of your **EDGE SERVER** to the [Peer] PublicKey Here

Move wg0.conf to /etc/wireguard
```
$ sudo mv wg0.conf /etc/wireguard
```

Copy your **PUBLIC** key of your **LOCAL NODE** to the publickey in your EDGE NODE wg0.conf file
```
(ON EDGE NODE)
$ vim /wireguard/etc/wg0.conf
[Peer]
#client 1 -- living room
PublicKey = this-local-node-public-key-goes-here=
AllowedIPs = 10.1.1.2/32
```
(on edge node) restart wireguard
```
$ wg-quick down wg0
$ wg-quick up wg0
```

Continue on your local node... activate wg
```
$ sudo wg-quick up wg0
```

Verify
```
$ sudo wg
```
Should see up and down traffic (if no down traffic verify Endpoint address.)

Enable Wireguard to start automatically at boot
```
$ systemctl enable wg-quick@wg0.service
$ systemctl daemon-reload
$ systemctl start wg-quick@wg0
$ reboot ... (to check that is working)
$ systemctl status wg-quick@wg0
```

### Docker
```
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```

Verify
```
$ docker version
$ sudo docker info
$ sudo docker run hello-world
```

Create Network "Master"
```
$ sudo docker network create -d bridge master
```


### Caddy 
```
$ vim ~/Caddyfile
http://example.com {
       respond "Yay!  It Works!"
       }
```
(replace example.com with your domain)

Setup Caddy Folders
```
$ mkdir ~/certs
$ mkdir ~/www
$ mkdir ~/www/example.com
```

Start Caddy
```
$ docker run -d --restart=always --name=caddy_web_server -p 80:80 -p 443:443 --network=master -v /home/pi/Caddyfile:/etc/caddy/Caddyfile -v /home/pi/www:/usr/share/caddy -v /home/pi/certs:/data caddy
```

Testing
```
$ sudo vim /etc/hosts
192.168.1.2 example.com
```
.. replace `192.168.1.2` with the ip of your local node.  and `example.com` with the domain you're using.
```
$ curl -v example.com
```
You should see.. Yay!  It Works! 

Try globally (comment out locally host mapping)
```
$ sudo vim /etc/hosts
#192.168.1.2 example.com

$ curl -v https://example.com
```

## Applications
Finally, a cloud needs to do things, you'll be able to build your cloud into whatever suits you. Here are a few examples to get you started.

* [Ghost](apps/ghost.md)
* [GOGS](apps/gogs.md)
* [Express](apps/express.md)

There's plenty [more](apps/README.md) one can install, and more becomes available all the time.

There you have it.  Your own cloud.  Let us know what you do with yours!

## Troubleshooting

## Discussion

Why run a personal cloud.
* Ad Free (Calmer/Cleaner/Faster)
* Control
* Convienence
* Persistance
* Personaliztion
* Privacy
* Options
* Speed
* "Unlimited" Storage

### Related Links

* https://www.inkandswitch.com/local-first.html
* https://www.reddit.com/r/selfhosted/
* https://neustadt.fr/essays/against-a-user-hostile-web/

### Similar Projects
* https://github.com/ahmadsayed/cloud-from-scratch
* https://github.com/funkypenguin/geek-cookbook
* https://github.com/progmaticltd/homebox
* https://github.com/sovereign/sovereign
* https://freedombox.org/


### Web GUI automated run your own cloud systems
Prefer turnkey web gui based management?  Take a look at these.
* https://caprover.com/
* https://cloudron.io/
* https://cozy.io/
* https://nextcloud.com/
* https://owncloud.org/
* https://sandstorm.io/
* https://yunohost.org/

### Useful services
* https://www.backblaze.com/
* https://ngrok.com/
* https://www.zerotier.com/


Have a suggestion, question, comment, or request?  Submit a [new issue](https://github.com/technomada/cloud-from-scratch/issues/new).
