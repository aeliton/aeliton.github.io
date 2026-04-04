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

1. Download the `.ovpn` files from my account;
2. Save provided username and password on a file (`chmod 600`);
3. Use the `openvpn` selecting one of the `udp` `.ovpn` files:
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

In our case, our 'attacker' will be
[DNSleaktest.com](https://www.dnsleaktest.com). Their page will use WebRTC [ICE
candidates](https://bloggeek.me/psa-mdns-and-local-ice-candidates-are-coming/),
a public [stun
server](https://webrtc.link/en/articles/stun-turn-servers-webrtc-nat-traversal/)
and very likely their own Authoritative DNS server. Details can be found (in
[their minified javascript](https://www.dnsleaktest.com/assets/js/app.js)).

The 'attack' goes like this:
1. The infected page requests in our device a non existing subdomain like
   `1cb6e9d1-6ec9-43e9-8aa6-8f26a8229c16.test.dnsleaktest.com`.
2. This host has never existed, so it won't be in our DNS cache.
3. The request now will leave our device and head to whatever DNS provider is
   being used, not to be found, it will move on to the root DNS serve -> TLD
   (Top Level Domain) -> Authoritative DNS server (malicious).

Now the UUID 1cb6e9d1-6ec9-43e9-8aa6-8f26a8229c16 is know by the attacker and
their DNS server just received an request to resolve it, they now know your DNS
provider.

### How to fix it

The fix is to make `NetworkManager` aware of the VPN connection so it can
redirect DNS requests to it now, rather than use the regular system DNS. This
can be accomplished by
[update-systemd-resolved](https://github.com/jonathanio/update-systemd-resolved).
To install its Debian variant you do:

```sh
sudo apt install openvpn-systemd-resolved
```

But this won't magically resolve your problem. You need to create the VPN
connection via `NetworkManager`. In short you have to:

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

The above steps were extracted from this [StackOverflow
answer](https://superuser.com/a/1847931/196820).

### The last problem

Awesome, we have a solution. But I refuse to use a single country
configuration. I want to use all provided configurations.

The automation of the creation and configuration of each country can be done
with this simple shell script:

```sh
#!/bin/sh

# The directory of the `.ovpn` files (e.g./etc/openvpn/provider)
OVPN_DIR=$1

# The full path of your VPN credentials
CREDENTIALS_PATH=$2

# The username MUST be at the first line of the file.
vpn_user=$(head -n 1 ${CREDENTIALS_PATH})

# The password MUST be at the last line of the file.
vpn_pass=$(tail -n 1 ${CREDENTIALS_PATH})

for file in ${OVPN_DIR}/*; do
  # NetworkManager uses the basefilename minus the extension as connection name.
  conn_name=$(basename "${file%.*}")

  # Create and configure all connections.
  nmcli con import type openvpn file $file
  nmcli con mod ${conn_name} vpn.user-name ${vpn_user}
  nmcli con mod ${conn_name} +vpn.data "password-flags=0"
  nmcli con mod ${conn_name} vpn.secrets "password=${vpn_pass}"
done
```

You can save this script somewhere and run:

```sh
sudo sh create-nm-vpn-connections.sh \
            /etc/openvpn/provider    \
            /etc/openvpn/provider/user-pass.txt
```

And check the results with:

```sh
nmcli conn # lists all newly created connections
nmcli conn up <one of the created connection>
```

With the VPN connection successfully established, you can redo the
[DNSleaktest.com](https://www.dnsleaktest.com) to verify the absence of DNS
leaks.
