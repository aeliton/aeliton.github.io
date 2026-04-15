+++
date = '2026-04-02'
draft = false
title = 'Avoid DNS leaking with Network Manager and OpenVPN'
+++

### What is a DNS leak?

Straight from [Wikipedia](https://en.wikipedia.org/wiki/DNS_leak): _"A DNS leak
is a security flaw that allows DNS requests to be revealed to internet service
provider (ISP) DNS servers, despite the use of a VPN service to attempt to
conceal them.[1] The vulnerability allows an ISP, as well as any on-path
eavesdroppers, to see what websites a user is visiting."_

If you are on Linux, use `NetworkManager` and connects to VPNs using `openvpn`,
you might be interested in reading this further.

### Identifying if you have a DNS leak

First, if you don't want your DNS requests to leak, you need to instruct your
browser to use your DNS from your network. ProtonVPN™ has [an interesting read
on the subject](https://protonvpn.com/support/dns-leaks-privacy) and recommends
to use [DNSleaktest.com](https://www.dnsleaktest.com) to test it.

It's all very simple:
1. Make your browser to execute DNS requests from you system;
1. Connect to your VPN;
1. Go to [DNSleaktest.com](https://www.dnsleaktest.com);
1. Select one of the tests (I used the _Extended Test_);
1. Check the results:
    - If the result of the test shows your ISP DNS, you have a DNS leak.

### I had a DNS leak and I didn't know

I used to ~~'hide my browsing traffic from the coffee shop owner'~~ connect to
my VPN provider using `openvpn` directly like this:

1. Download the `.ovpn` files from my VPN provider;
2. Save username and password on a file (`chmod 600`);
3. Use the `openvpn` selecting one of the `.ovpn` files:
    ```sh
    sudo openvpn --config /etc/openvpn/provider_udp/fr.udp.ovpn \
      --auth-user-pass /etc/openvpn/provider_udp/user-pass.txt
    ```

To avoid having to set `--auth-user-pass` via the command line all the time, I
injected the path to the credentials into the `.opvn` files using:

```sh
grep auth-user-pass * -il | xargs \
  sudo sed -i 's/auth-user-pass/& \/etc\/openvpn\/provider_udp/user-pass.txt/'
```

This allows me to simple run:

```sh
    sudo openvpn --config /etc/openvpn/provider_udp/fr.udp.ovpn
```

At this point, you have a VPN connection, your browsing data is protected, but
your DNS might leak, providing the attacker a good indication of your
whereabouts if you have the wrong web page opened.

### Understanding the attack

In our case, our 'attacker' will be
[DNSleaktest.com](https://www.dnsleaktest.com). Their page will use WebRTC [ICE
candidates](https://bloggeek.me/psa-mdns-and-local-ice-candidates-are-coming/),
a public [stun
server](https://webrtc.link/en/articles/stun-turn-servers-webrtc-nat-traversal/)
and very likely their own Authoritative DNS server. Details can be found (in
[their minified javascript](https://www.dnsleaktest.com/assets/js/app.js)).

The 'attack' goes like this:
1. The infected page is opened in our browser;
1. The page requests a non existing subdomain like
   `1cb6e9d1-6ec9-43e9-8aa6-8f26a8229c16.test.dnsleaktest.com`.
1. Such page won't be in our DNS cache, so the request will leave our device
   and head to whatever DNS provider is being used, to not be found and then
   will move on to the root DNS serve, then TLD (Top Level Domain) and, finally,
   Authoritative DNS server, where they can match the requesting DNS to our
   user.

### How to fix it

The fix is to make `NetworkManager` aware of the VPN connection so it can
redirect DNS requests through the VPN tunnel, rather than use the regular system
DNS. This can be accomplished by
[update-systemd-resolved](https://github.com/jonathanio/update-systemd-resolved).
To install its Debian variant you do:

```sh
sudo apt install openvpn-systemd-resolved
```

But this won't magically resolve your problem. You need to create the VPN
connection via `NetworkManager`:

1. Import the `.ovpn` configuration to create a `NetworkManager` connection:
    ```sh
    nmcli con import type openvpn file <provider>.ovpn
    ```
2. Indicate to `NetworkManager` it should not ask or expect credentials:
    ```sh
    nmcli con mod <connection name> +vpn.data "password-flags=0"
    ```
3. Set the credentials:
    ```sh
    nmcli con mod <connection name> vpn.user-name <your username>
    nmcli con mod <connection name> vpn.secrets "password=<your password>"
    ```
4. Connect to the VPN via `NetworkManager`:
    ```sh
    nmcli con up <connection name>
    ```

The above steps were extracted from [here](https://superuser.com/a/1847931/196820).

### Automating the fix for multiple `.ovpn` files

The automation of the creation and configuration of each country can be done
with this [shell
script](https://raw.githubusercontent.com/aeliton/dotfiles/refs/heads/main/.sh/nm-ovpn-config.sh).
It defines a function on your shell only if you source it (so that the script
wont need to be in your PATH).

```sh
source nm-ovpn.sh
nm-ovpn-config /etc/openvpn/provider
```

And check the results with:

```sh
nmcli conn # lists all newly created connections
nmcli conn up <one of the created connection>
```

With the VPN connection successfully established, you can redo the
[DNSleaktest.com](https://www.dnsleaktest.com) to verify the absence of DNS
leaks.
