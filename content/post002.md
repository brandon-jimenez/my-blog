---
title: Docker Compose (Part 1)
image: /images/docker-post1.jpg
imageMeta:
  attribution:
  attributionLink:
featured: true
authors:
  - brandon
date: Sun Mar 14 2021
tags:
  - docker
---

# Switching to Docker Compose
### (Part 1)

## Ubuntu Snaps

Originally, when I started up my Docker environment I set it up using the snap from Ubuntu. It was just one of the many things you can install when first setting up Ubuntu Server. I had thought that since I was going to install Docker anyway why not,, just get it out of the way.

Or so I thought...

As it turns out, there's issues when using the docker snap.

```
docker-runc did not terminate sucessfully container_linux.gosignaling init process caused "permission denied"
```

Did some reading about it being a possbile conflict with AppArmor as apparently installign snaps adds a lot apprmor profiels. 

```
sudo dmesg | grep apparmor
```

And oh boy, I think there's my problem.
```
[  100.969777] audit: type=1400 audit(1614973369.479:47): apparmor="STATUS" operation="profile_load" profile="unconfined" name="docker-default" pid=907 comm="apparmor_parser"[ 1882.634712] audit: type=1400 audit(1614975151.576:48): apparmor="DENIED" operation="signal" profile="docker-default" pid=7140 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 1884.708985] audit: type=1400 audit(1614975153.652:49): apparmor="DENIED" operation="signal" profile="docker-default" pid=7145 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 1892.755529] audit: type=1400 audit(1614975161.696:50): apparmor="DENIED" operation="signal" profile="docker-default" pid=7151 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 1894.829270] audit: type=1400 audit(1614975163.772:51): apparmor="DENIED" operation="signal" profile="docker-default" pid=7156 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 1902.324871] audit: type=1400 audit(1614975171.268:52): apparmor="DENIED" operation="signal" profile="docker-default" pid=7189 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 1904.400709] audit: type=1400 audit(1614975173.344:53): apparmor="DENIED" operation="signal" profile="docker-default" pid=7195 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 1928.444355] audit: type=1400 audit(1614975197.384:54): apparmor="DENIED" operation="signal" profile="docker-default" pid=7221 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 1930.521375] audit: type=1400 audit(1614975199.464:55): apparmor="DENIED" operation="signal" profile="docker-default" pid=7226 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 1950.834013] audit: type=1400 audit(1614975219.775:56): apparmor="DENIED" operation="signal" profile="docker-default" pid=7259 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 1952.908453] audit: type=1400 audit(1614975221.847:57): apparmor="DENIED" operation="signal" profile="docker-default" pid=7264 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 5174.247437] audit: type=1400 audit(1614978443.112:58): apparmor="DENIED" operation="signal" profile="docker-default" pid=11107 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 5176.320782] audit: type=1400 audit(1614978445.188:59): apparmor="DENIED" operation="signal" profile="docker-default" pid=11112 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 5197.118513] audit: type=1400 audit(1614978465.983:60): apparmor="DENIED" operation="signal" profile="docker-default" pid=11143 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
[ 5199.193945] audit: type=1400 audit(1614978468.059:61): apparmor="DENIED" operation="signal" profile="docker-default" pid=11148 comm="runc" requested_mask="receive" denied_mask="receive" signal=kill peer="snap.docker.dockerd"
[ 5212.430526] audit: type=1400 audit(1614978481.294:62): apparmor="DENIED" operation="signal" profile="docker-default" pid=11158 comm="runc" requested_mask="receive" denied_mask="receive" signal=term peer="snap.docker.dockerd"
```

So that's what is occuring everytime I make an attempt to stop ***any container*** in my stack. 

```
$ sudo docker stop ce896ebb9ebc
Error response from daemon: cannot stop container: ce896ebb9ebc: Cannot kill container ce896ebb9ebca07315b63f06f6a1c7bf489b42fa13555631d4a6edabd28e4299: unknown error after kill: runc did not terminate sucessfully: container_linux.go:392: signaling init process caused "permission denied"
: unknown
```

## Introducing Docker-Compose

What I've been doing the last couple of week is just, restart the whole server when I need to make changes. That's clearly the wrong way to deal with this and it's time to make a change. 
*There is a way to remove "unknown" from AppArmor but I still want to use this as an excuse to switch.*

After how long I've been dealing with this, this is how I feel about snaps now:

![Ubuntu Snap Be Like That Sometimes](/images/ubuntu-snap.jpg)

The problem's I've been experiencing aren't exclusive to the docker snap but I believe so snaps themselves and how they work with apparmor. But regardless I want the flexbility that apt provides.

So it's a two part problem; switch to apt and migrate my docker run files to docker compose.

I have persistent storage setup for my containers so it should be a breeze but I've still been stressing about doing this.

From this:
```
docker run --net=host --name=homebridge -e TZ=America/Chicago -e PUID=1000 -e PGID=1000 -e HOMEBRIDGE_CONFIG_UI=1 -e HOMEBRIDGE_CONFIG_UI_PORT=8080 -v /media/config:/homebridge --restart unless-stopped oznu/homebridge
```

To this:
```
version: '3.3'
services:
    homebridge:
        network_mode: host
        container_name: homebridge
        environment:
            - TZ=America/Chicago
            - PUID=1000
            - PGID=1000
            - HOMEBRIDGE_CONFIG_UI=1
            - HOMEBRIDGE_CONFIG_UI_PORT=8080
	ports:
	    - 8052:8080
        volumes:
            - '/media/config:/homebridge'
        restart: unless-stopped
        image: oznu/homebridge
```

After stopping and removing the docker snap, I followed the instructions from the official docker documentation. 

One of the dependencies for Compose is [Docker Engine](https://docs.docker.com/engine/install/ubuntu/).

Following the directions, after setting up the repository for Docker, getting Docker Engine up is as simple as: 

```
 sudo apt-get update

 sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Once that’s up we can install Docker Compose. Much like the instructions for the engine, [the instructions for compose are also easy to follow](https://docs.docker.com/compose/install/).

The long and short of it is two one-liners.
First:
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

And then applying executable permissions to the binary with the second one-liner:
```
sudo chmod +x /usr/local/bin/docker-compose
```

Afterwards you simply test it like this!:
![Validating Compose Install](/images/compose-success.png)
*Note: Obfuscating my username and name of the server as my internal naming convention is based on inside jokes in my household. Rather not embarrass myself.*

## Conclusion so far

It's fairly simple to get a hang of Compose.

With the help of [composerize](https://www.composerize.com/) and the reference compose files for the services I run; I've made some progress.

Below is a table showing the progress so far in this migration. 

| To-Do | Mirgated | Issues |
| --- | --- | --- |
| pihole | complete | WEBPASSWAORD env flag not working for a reason I couldn't get to the bottom of |
| portainer | pending | configuration not carrying over |
| homebridge-wyze | pending | - |
| plex | pending | - |
| unifi-controlller | complete | No issues |

I would to figure out where my portainer config is hiding since it’s not carrying over when setting it back up with compose. 

Plex and Homebridge - They haven’t been a priority for me to get up, but I’ll circle back to them.

The switch over was a little more involved than I expected, so this is gonna be a two part journey.

Catch you on part 2.

***Always Stay Curious***