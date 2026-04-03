+++
date = '2026-04-02'
draft = false
title = 'Avoid DNS leaking with Network Manager and Openvpn'
+++

# What is a DNS leak?

Straight from [Wikipedia](https://en.wikipedia.org/wiki/DNS_leak): _"A DNS leak
is a security flaw that allows DNS requests to be revealed to internet service
provider (ISP) DNS servers, despite the use of a VPN service to attempt to
conceal them.[1] The vulnerability allows an ISP, as well as any on-path
eavesdroppers, to see what websites a user is visiting."_

If you have been using a VPN to try to gain some internet privacy, you might be
interested in reading this further.

# Identifying if you have a DNS leak

ProtonVPN™ has [an interesting read on the
subject](https://protonvpn.com/support/dns-leaks-privacy) and recommends to use
[DNSleaktest.com](https://www.dnsleaktest.com).

It's all very simple, got to [DNSleaktest.com](https://www.dnsleaktest.com) and
select one of the tests (I used the _Extented Test_), if the result of the test
shows anything else other than the VPN provided DNS resolver URL, you have a DNS
leak.

# I had a DNS leak and I didn't know

## How do I configure my network interfaces

I use `NetworkManager`, via its `nmcli` tool, which is CLI.

## How I used to connect to a VPN provider

I used to use `openvpn` directly, but in conjunction to `NetworkManager`, we
will have DNS leaks because in the end, it's the former who decides how to
direct DNS requests.

What I **used** to do with `openvpn` was:

1. Download the openvpn files from my account;
2. Save provided username and password on a file (`chmod 600`);
3. Use the openvpn selecting one of the `udp` ovpn files:
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

So after being connected to the Internet and then connecting to my VPN provider,
I used [DNSleaktest.com](https://www.dnsleaktest.com) and `voilá`, I had a DNS
leak and could not rest in peace knowing that my ISP provider knows about all
the debian ISOs I had download until now!

# How to fix it

After some reading, I found out about
[update-systemd-resolved](https://github.com/jonathanio/update-systemd-resolved)
and decided to install it's Debian package variant:

```sh
sudo apt install openvpn-systemd-resolved
```

The next thing that helped was this [StackOverflow answer](https://superuser.com/a/1847931/196820) which worked. In short you have to:

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

# The last problem

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
