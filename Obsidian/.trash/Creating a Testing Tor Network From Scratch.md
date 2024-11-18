---
title: "Creating a Testing Tor Network From Scratch"
source: "https://medium.com/@dax_dev/creating-a-testing-tor-network-from-scratch-e952d76a18cb"
author:
  - "[[dax]]"
published: 2024-10-05
created: 2024-11-18
description: "A complete, in-depth tutorial-like example on how to set up your own testing Tor network from scratch."
tags:
  - "clippings"
---
[

![dax](https://miro.medium.com/v2/resize:fill:88:88/1*nloKfxMWhBQStd3pnj_Yuw.jpeg)

](https://medium.com/@dax_dev?source=post_page---byline--e952d76a18cb--------------------------------)

I’ve recently taken an interest in the Tor network (The Onion Router). Specifically in how it works behind the scenes. I’ve had multiple ideas of things to try and experiment with, but it would require an isolated network that I entirely controlled. If you try to find information online on creating your own Tor network, you’ll find yourself scavenging through Stack Overflow, Tor forums, the Tor specification and the Tor *GitLab* repository. The goal of this article is to offer a complete, in-depth tutorial-like example on how to set this up, saving you the time to find all of this information yourself. Everything will be explained in detail, leaving no stones unturned, so you’ll completely understand the pieces of a Tor network. I will also skip the formalities and assume you know what the Tor network is, since you are trying to build your own.

> **TL;DR** If this contains too much detail for you, you can go [here](https://github.com/daxAKAhackerman/testing-tor-network) and have a look at the *README* for a boiled-down version of this article, as well as a CLI that will do all the heavy lifting for you.

## Requirements

- A working installation of Docker

## Creating a base image

The first thing we’ll do is create a base Tor docker image. The instructions are taken directly from the “[How to install Tor](https://community.torproject.org/onion-services/setup/install/)” page on the Tor Project website.

```
FROM debian:12.7ENV DEBIAN_FRONTEND=noninteractiveRUN apt-get update && apt-get install apt-transport-https wget gpg iproute2 iputils-ping sudo vim procps bash-completion curl tcpdump bsdmainutils telnet -yRUN echo "deb [signed-by=/usr/share/keyrings/deb.torproject.org-keyring.gpg] https://deb.torproject.org/torproject.org bookworm main" > /etc/apt/sources.list.d/tor.listRUN echo "deb-src [signed-by=/usr/share/keyrings/deb.torproject.org-keyring.gpg] https://deb.torproject.org/torproject.org bookworm main" >> /etc/apt/sources.list.d/tor.listRUN wget -qO- https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --dearmor | tee /usr/share/keyrings/deb.torproject.org-keyring.gpg >/dev/nullRUN apt-get update && apt-get install tor deb.torproject.org-keyring nyx -y# Delete the default torrc file since we are going to replace itRUN rm /etc/tor/torrc
```

I’ve added *Nyx* to the list of installed packages to help us monitor our nodes, as well as a few convenience packages. Build the image using `docker build -t testing-tor .`.

Last thing we’ll want to do is create a docker network so we can assign static IPs to our nodes.

```
docker network create testing-tor --subnet 10.5.0.0/16
```

## **Adding your first directory authority**

Directory authorities (or DA for short) are at the core of the Tor network. Their job is to keep the state of the network (known as the consensus) and to distribute it to the various nodes. The nodes can then use the information contained within to build circuits across the network. The status is called a consensus for a reason: for a valid consensus to be published, an agreement of 2/3 of the DAs on the state of the network needs to be reached. This ensures that no single compromised DA can affect the network. Let’s create our DA.

```
docker run --name tmp-da-1 --ip 10.5.1.10 --network testing-tor -it testing-tor bash
```

Every configuration for your nodes will be in the `/etc/tor/torrc` file. Here’s what we need to kick off the DA (we’ll be adjusting it later). Each line is commented to explain why it needs to be there. For more detail on each directive have a look at [the documentation](https://manpages.debian.org/testing/tor/torrc.5.en.html).

```
# This automatically sets multiple options that make it easier to setup a testing network# For example, it reduces voting intervals, allow routing of private IPs and disables distinct subnet restrictionsTestingTorNetwork 1# This will speed up the process of adding nodesAssumeReachable 1# We disable IPv6 since we don't use itAddressDisableIPv6 1# We activate the control port to use NyxControlPort 9051CookieAuthentication 1# These two lines make the node act as a DAAuthoritativeDirectory 1V3AuthoritativeDirectory 1# Advertising the directory service port and onion routing port are required for DAsDirPort 80 IPv4OnlyORPort 9001 IPv4Only# Disable the client port since we won't be using that on the DASocksPort 0# Without the following lines, if you stop the network and boot it again later, the nodes will have trouble getting their flags again# This is because flags are given based on the fraction of uptime compared to the relay lifetime# It will basically give the guard and hsdir (hidden service directory) flags to all nodes by defaultTestingDirAuthVoteGuard *TestingDirAuthVoteHSDir *# The nickname will allow us to reference our node by name instead of fingerprintNickname da1# The address seems to be needed when our nodes run on a private subnet# Without it, Tor never succeeds in finding our node's IP# Of course, adjust with your node's actual IPAddress 10.5.1.10# Having a contact info directive is just good practice and silences a warningContactInfo da1 <da1 AT localhost>
```

Next we will generate our DA keys and fingerprint.

> The `*debian-tor*` user is automatically created when you install tor on *Debian*. If you use something other than *Debian*, validate what’s the name of this user in your `*/etc/passwd*` file.

```
mkdir -p /var/lib/tor/.tor/keyschown -R debian-tor:debian-tor /var/lib/tor/.torcd /var/lib/tor/.tor/keysecho $(tr -dc A-Za-z0-9 </dev/urandom | head -c 12) | sudo -u debian-tor tor-gencert --create-identity-key -m 12 -a 10.5.1.10:80 --passphrase-fd 0cd ..sudo -u debian-tor tor --list-fingerprint --dirauthority "placeholder 127.0.0.1:80 0000000000000000000000000000000000000000"echo """DirAuthority da1 orport=9001 no-v2 v3ident=$(grep "fingerprint" keys/authority_certificate | cut -d " " -f 2) 10.5.1.10:80 $(cat fingerprint | cut -d " " -f 2)"""vi /etc/tor/torrc
```

Good news, our first DA is configured! Let’s save our configuration to an image.

```
exitdocker commit tmp-da-1 da-1docker rm tmp-da-1
```

## Adding more directory authorities (optional)

You can technically run your network with a single DA, but this kind of defeats the purpose of reaching a consensus. If you want the full experience, let’s add 2 more! Simply repeat the steps that were taken to create the first DA, making sure to adjust any references to the nickname and IP address along the way.

The last thing you will have to do to finally bring your (currently pretty slim) network to life is to make every DA aware of each other. To do this, we will simply adjust the images we created to add all `DirAuthority` directives to our `torrc` (so if you have 3 DAs, your `torrc` will contain 3 `DirAuthority` lines).

```
docker run --name tmp-da-1 --ip 10.5.1.10 --network testing-tor -it da-1 bashvi /etc/tor/torrcexitdocker commit tmp-da-1 da-1docker rm tmp-da-1
```

## Bringing your network up

Time to start your DA container(s)!

```
docker run --name da-1 --ip 10.5.1.10 --network testing-tor -d da-1 sudo -u debian-tor tordocker run --name da-2 --ip 10.5.1.11 --network testing-tor -d da-2 sudo -u debian-tor tordocker run --name da-3 --ip 10.5.1.12 --network testing-tor -d da-3 sudo -u debian-tor tor...
```

You can now monitor your nodes using `Nyx` with `docker exec -it da-1 nyx`. When all your nodes get the `Authority` flag, it means that they have been added to the consensus and are considered valid DAs. Congratulations! The most complicated part is behind us!

## **Adding middle and guard relays**

Tor circuits normally consist of three relays: a guard (entry) relay, a middle relay and an exit relay. In terms of configuration, guards and middles are the same. The guard flag is normally given to relays that have sufficient bandwidth, a good uptime and a little bit of age. To avoid waiting for all of this, we previously put the `TestingDirAuthVoteGuard *` directive in our DAs `torrc` which will give all relays the guard flag by default. Let’s create a relay container.

```
docker run --name tmp-relay-1 --ip 10.5.1.20 --network testing-tor -it testing-tor bash
```

Here’s the `torrc` configuration (don’t forget to put in the DirAuthority lines from all of your DAs).

```
TestingTorNetwork 1AssumeReachable 1AddressDisableIPv6 1ControlPort 9051CookieAuthentication 1SocksPort 0Nickname relay1Address 10.5.1.20ContactInfo relay1 <relay1 AT localhost>ORPort 9001 IPv4Only# All the DirAuthority information of the DAs in the networkDirAuthority da1 orport=9001 no-v2 v3ident=...DirAuthority da2 orport=9001 no-v2 v3ident=...DirAuthority da3 orport=9001 no-v2 v3ident=......
```

Once again, let’s save the image and start the container.

```
exitdocker commit tmp-relay-1 relay-1docker rm tmp-relay-1docker run --name relay-1 --ip 10.5.1.20 --network testing-tor -d relay-1 sudo -u debian-tor tor
```

Simply repeat these steps for the number of relays you want. Don’t forget to monitor your relays using *Nyx* to see when they get their flags and are ready to be used to build circuits.

## **Adding exit relays**

Exit relays are very similar to guard/middle relays, except they contain an `ExitPolicy` that allows traffic to exit the network through them. You know the drill by now.

```
docker run --name tmp-exit-1 --ip 10.5.1.30 --network testing-tor -it testing-tor bash
```
```
TestingTorNetwork 1AssumeReachable 1AddressDisableIPv6 1ControlPort 9051CookieAuthentication 1SocksPort 0Nickname exit1Address 10.5.1.30ContactInfo exit1 <exit1 AT localhost>ORPort 9001 IPv4Only# This will allow exit with the default policy# See torrc documentation for the exact list of allowed ports if you're curiousExitRelay 1DirAuthority da1 orport=9001 no-v2 v3ident=...DirAuthority da2 orport=9001 no-v2 v3ident=...DirAuthority da3 orport=9001 no-v2 v3ident=......
```
```
exitdocker commit tmp-exit-1 exit-1docker rm tmp-exit-1docker run --name exit-1 --ip 10.5.1.30 --network testing-tor -d exit-1 sudo -u debian-tor tor
```

Once again, repeat these steps for the number of exit relays you want, and monitor the flags using *Nyx*.

## **Adding a client**

At this point, we have a fully functional Tor network and are able to route traffic through it. All we need is a client. We’ll go over three ways to send your traffic through.

The first method will be to add a client container and to expose the Tor Socksv5 port to the host, so we can proxy CURL through it. The procedure is pretty much the same has what we did above.

```
docker run --name tmp-client-1 --network testing-tor -it testing-tor bash
```
```
TestingTorNetwork 1# We specifically listen on 0.0.0.0 to allow the host to connect through an exposed Docker portSocksPort 0.0.0.0:9050DirAuthority da1 orport=9001 no-v2 v3ident=...DirAuthority da2 orport=9001 no-v2 v3ident=...DirAuthority da3 orport=9001 no-v2 v3ident=......
```
```
exitdocker commit tmp-client-1 client-1docker rm tmp-client-1docker run --name client-1 --network testing-tor -p 9050:9050 -d client-1 sudo -u debian-tor tor
```

We can now use CURL to send requests!

```
curl -x socks5h://localhost:9050 https://hackerman.ca
```

The second way is really just a twist on the first one. Most modern web browsers support the Socksv5 protocol and can be configured to use them to send requests. If you want something a little more convenient than changing your proxy configuration every time you want to test your network, you can download a browser extension such as *FoxyProxy* to easily switch between configurations.

The third (and coolest) way is to use the official Tor browser. However, this depends a little bit on your machine setup and might require a bit of routing configuration (for example if you are using Windows + WSL2 like me). Just be sure that your machine is able to ping your nodes and you should be fine. [Download and install the Tor browser](https://www.torproject.org/download/), navigate to your installation directory, and go to `Browser\TorBrowser\Data\Tor`. In this folder, you will find the now familiar `torrc` file that you can simply replace with the one above (you can omit the `SocksPort` directive since we don’t have to expose the port). If it’s not the first time you use this browser, you will have to get rid of everything in this folder first (except the `geoip`, `geoip6` and `torrc-defaults` files). Start up the browser, and you’re good to go!

## **Adding a hidden service**

The final things to add to our network are hidden services. These are the services with the `.onion` addresses that you see all over the Tor network. Like when you set up the client, there are multiple ways to do this, but we’ll go with what is, in my opinion, the simplest and most versatile way to do it. First, let’s create a container with our service. We’ll setup a simple *NGINX* default page.

```
docker run --network testing-tor --ip 10.5.1.50 -d nginx
```

Next, let’s create a HS container.

```
docker run --name tmp-hs-1 --ip 10.5.1.60 --network testing-tor -it testing-tor bash
```
```
TestingTorNetwork 1AssumeReachable 1AddressDisableIPv6 1ControlPort 9051CookieAuthentication 1SocksPort 0# Set the location of the HS filesHiddenServiceDir /var/lib/tor/.tor/hs/# Tell Tor where to forward requests# Here we tell it to listen on port 80 and to forward all requests to 10.5.1.50:80HiddenServicePort 80 10.5.1.50:80DirAuthority da1 orport=9001 no-v2 v3ident=...DirAuthority da2 orport=9001 no-v2 v3ident=...DirAuthority da3 orport=9001 no-v2 v3ident=......
```
```
exitdocker commit tmp-hs-1 hs-1docker rm tmp-hs-1docker run --name hs-1 --ip 10.5.1.60 --network testing-tor -d hs-1 sudo -u debian-tor tor
```

You can get the onion URL from the HS container using `docker exec hs-1 cat /var/lib/tor/.tor/hs/hostname`, and use any of the methods listed above in the client section to access your new hidden service!

## **Now do it faster**

Well, that was convoluted. This article does its job in teaching you everything it takes to spin up a Tor network from scratch. However, I think we can all agree that while this works, it’s a bit too involved to easily spin up a network to test something real quick, and then take down to save the resources. The good news is, I’ve taken all of what we did above and packed it neatly into this Github repository: [https://github.com/daxAKAhackerman/testing-tor-network](https://github.com/daxAKAhackerman/testing-tor-network). The repository is made to be lightweight; there is very little code in there. It consists of a simple Python CLI that wraps the different docker commands discussed above, as well as a few scripts, `torrc` files and `Dockerfiles` to make the whole thing painless. The same results that we got above can be achieved with the following commands in a few seconds (note that Python3.12 as well as pipenv are required).

```
git clone https://github.com/daxAKAhackerman/testing-tor-networkcd testing-tor-networkmakepipenv shellpython cli/main.py container add-da --count 3python cli/main.py container add-relay --count 5python cli/main.py container add-exit --count 3python cli/main.py container add-client --port 9050docker run --network testing-tor --ip 10.5.0.10 -d nginxpython cli/main.py container add-hs --service-ip 10.5.0.10
```

Have a look at the *README* for more detail!

## **Conclusion**

In this article, we successfully set up a testing Tor network consisting of directory authorities, guard, middle and exit relays. We’ve shown three methods of accessing this network, and we’ve even added a hidden service. Finally, we saw that the whole process could be automated in a few seconds using the testing-tor-network CLI. So what now? Well, now it’s time to explore, break, tweak, and so on! You can try the various directive that can be put in the `torrc` files of your nodes and see the effect, test attack scenarios such as deanonymization attacks, try to break things and do what you’re not supposed to do!

For my part, that’s exactly what I’m going to do! Keep an eye out for my next articles, where I’ll dive deeper into the inner workings of the Tor network.

That’s all folks, stay safe out there, and hack the planet!