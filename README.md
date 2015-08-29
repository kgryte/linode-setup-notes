Linode Setup
===
> Notes on how to setup [Linode](http://linode.com).

---
## Provision

See the getting started [guide](https://www.linode.com/docs/getting-started). After a few minutes, the [Linode](http://linode.com) will be provisioned. This guide assumes you have chosen `Ubuntu 14.04 LTS` as your base image.


---
## Remote Access

To connect to your new [Linode](http://linode.com),

``` bash
$ ssh root@<your_server_ip>
```

where your server IP may be found via `Linodes --> dashboard --> Remote Access`. You will be prompted for the `root` password you created when deploying a new image to your [Linode](http://linode.com).


---
## Basic Ubuntu Setup

To set up your new server, do as follows...

#### Hostname

Set the server hostname. The hostname should be unique (similar to naming a person or pet). For example,

``` bash
$ echo "myserver" > /etc/hostname
$ hostname -F /etc/hostname
```

Verify that the hostname was set:

``` bash
$ hostname
```


#### Hosts

Set the server's full-qualified domain name (FQDN) by updating the `/etc/hosts` file.

``` bash
$ sudo nano /etc/hosts
```

Update the `/etc/hosts` file to include your FQDN:

```
127.0.0.1 	localhost.localdomain 	localhost
127.0.0.1 	ubuntu
<your_server_ip> myserver.<your_domain> 	myserver
<your_ipv6_ip> myserver.<your_domain> 	myserver
```

where `<your_server_ip>` is your server's IP address and `<your_domain>` is a domain over which you have control; e.g., `example.com`.

The assigned FQDN should have a corresponding "A" record in DNS pointing to your server's IPv4 address and an "AAAA" record in DNS pointing to your server's IPv6 address. Doing so allows you to easily resolve your server when using, e.g., SSH.

``` bash
$ ssh myserver.<your_domain>
```

For more information, see the "Adding DNS Records" section in the Linode [guide](https://www.linode.com/docs/websites/hosting-a-website/).


#### Timezone

Set the server timezone:

``` bash
$ dpkg-reconfigure tzdata
```

Verify that the date is correct:

``` bash
$ date
```


#### Update

Install any available updates and security patches:

``` bash
$ apt-get update
$ apt-get upgrade --show-upgraded
```

---
## Basic Security Setup

To setup basic security and protect the server from unauthorized access, do as follows...

#### User

You are recommended to never use `root` for day-to-day tasks. A `root` user has the ability to execute any command, including commands which could break your server. For most tasks, you can use an account having normal permissions and execute `superuser` commands using `sudo`. To add a new user,

``` bash
$ adduser <your_username>
```

where `<your_username>` if your desired user name. Add the user to the `sudo` group:

``` bash
$ usermod -a -G sudo <your_username>
```

Logout from the server and then login with the new user:

``` bash
$ logout
$ ssh <your_username>@<your_server_ip>
```


#### SSH

SSH keys allow you to authenticate using a public-private key pair and remove the need for password-based authentication.

__WARNING__: do __not__ complete the following steps from a publicly shared computer. Run the commands from your local computer.


##### Generate SSH keys

If you have already generated SSH keys before (e.g., when configuring Github to use SSH), you can skip the following step. To generate SSH keys,

``` bash
$ ssh-keygen -t rsa -C "<your_email_address>"
```

When prompted, create a strong password. If you use a Mac, the password can be added to your keychain.


##### Copy the public key

Copy the public key to the server using the secure copy command `scp` from your local computer.

``` bash
$ scp ~/.ssh/id_rsa.pub <your_username>@<your_server_ip>:
```

Remember the `:` at the end. Once copied to our Linode server, we need to create a new directory on that server to store the public key and set the appropriate permissions.

``` bash
$ mkdir .ssh
$ mv id_rsa.pub .ssh/authorized_keys
$ chown -R <your_username>:<your_username> .ssh
$ chmod 700 .ssh
$ chmod 600 .ssh/authorized_keys
```

##### Disable SSH password authentication

If you connect to your Linode from only one computer, you should disable password authentication and require that all users authenticate using key-based authentication. If you need to connect from multiple computers, you may want to keep password authentication enabled to prevent having to copy your private key to multiple computers.

``` bash
$ sudo nano /etc/ssh/ssh_config
```

Change `PasswordAuthentication` to `no`. You may need to remove the `#` symbol to uncomment the line.

```
PasswordAuthentication no
```

Change the `PermitRootLogin` to `no`. You may need to add this line.

```
PermitRootLogin no
```

Restart the SSH service.

``` bash
$ sudo service ssh restart
```

After the SSH service restarts, the SSH configuration changes will be applied.


##### Change the SSH port

Since all Ubuntu servers have a `root` user and most servers run SSH on port `22` (default), hackers often use automated attacks to guess the `root` password. We already prevented the `root` user from using SSH to login (see above). We can make things more difficult by changing our SSH port to a port other than `22`.

``` bash
$ sudo nano /etc/ssh/sshd_config
```

Set the `Port` to a value less than `1024`; e.g., `567`. You may need to remove the `#` symbol to uncomment the line.

```
Port 567
```

Save the file and restart the SSH service.

``` bash
$ sudo service ssh restart
```

Now, when connecting to our Linode server, we need to specify the SSH port.

``` bash
$ ssh <your_username>@myserver.<your_domain> -p 567
```


#### Fail2Ban

[Fail2Ban](Fail2Ban) is an application that prevents dictionary attacks on your server. [Fail2Ban](Fail2Ban) can monitor login attempts across different protocols (SSH, HTTP, SMTP) and will create temporary firewall rules to block traffic from an attacker's IP address. To install,

``` bash
$ sudo apt-get install fail2ban
```

To configure,

``` bash
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
$ sudo nano /etc/fail2ban/jail.local
```

In __both__ the `[ssh]` and `[ssh-ddos]` sections, set the `port` to the SSH port you specified above.

```
[ssh]

enabled 	= true
port 		= <your_ssh_port>
```

In the `[ssh-ddos]` section, set `enabled` to `true`.

```
[ssh-ddos]

enabled 	= true
port 		= <your_ssh_port>
```

Some additional configuration changes (optional):
*	Set the `bantime` option to specify how long a ban should last.
*	Set the `maxretry` option to specify the number of allowed retries before an IP is banned.

Save the file and restart Fail2Ban.

``` bash
$ sudo service fail2ban restart
```

Once running, Fail2Ban will block any IP address which exceeds the maximum allowed connection attempts and will log the event to `/var/log/fail2ban.log`.


#### Firewall

Create a firewall to limit and block unwanted inbound traffic. To see the current rules,

``` bash
$ sudo iptables -L
```

which, beside any rules created by Fail2Ban, should contain an empty ruleset. To create a file for specifying filewall rules,

``` bash
$ sudo nano /etc/iptables.firewall.rules
```

Paste the following into the opened file. By default, the rules allow traffic to the following services and ports: HTTP (80), HTTPS (443), SSH (`<your_ssh_port>`), and ping. Be sure to replace `<your_ssh_port>` with the port specified in `sshd_config` above.

```
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow SSH connections
#
#  The -dport number should be the same port number you set in sshd_config
#
-A INPUT -p tcp -m state --state NEW --dport <your_ssh_port> -j ACCEPT

#  Allow ping
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Drop all other inbound - default deny unless explicitly allowed policy
-A INPUT -j DROP
-A FORWARD -j DROP

COMMIT
```

Save the file and then activate the firewall rules.

``` bash
$ sudo iptables-restore < /etc/iptables.firewall.rules
```

Verify the rules:

``` bash
$ sudo iptables -L
```

To ensure that the firewall rules are activated each time your Linode restarts, open

``` bash
$ sudo nano /etc/network/if-pre-up.d/firewall 
```

and paste

```
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Save the file and set file permissions.

``` bash
$ sudo chmod +x /etc/network/if-pre-up.d/firewall
```

---
## Reverse DNS

While DNS allows a domain name to be resolved to an IP address, reverse DNS allows an IP address to be resolved to a domain name. Reverse DNS is useful when network troubleshooting (`ping`, `traceroute`). From the Linode manager, navigate to `Linodes --> Remote Access --> Reverse DNS` and enter your domain.

---
## Two-Factor Authentication

You are encouraged to setup two-factor authentication when logging in to the Linode manager. From the Linode manager, navigate to `My Profile --> Password & Authentication --> Two-Factor Authentication`.



---
## References

*	[Getting started](https://www.linode.com/docs/getting-started)
*	[Securing your server](https://www.linode.com/docs/security/securing-your-server/)
*	[Hosting a website](https://www.linode.com/docs/websites/hosting-a-website/)
*	[How to setup your linode](http://feross.org/how-to-setup-your-linode/)


## Author

Athan Reines ([@kgryte](https://twitter.com/kgryte)).
