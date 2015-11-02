Linode Setup
===
> How to setup [Linode](http://linode.com).

The following are notes on how to setup [Linode](http://linode.com). The initial setup steps have been mainly compiled from the [Linode](http://linode.com) getting started guides, which collectively provide a good starting point for basic configuration. 

1. [Provision](#provision)
1. [Remote Access](#remote-access)
1. [Basic Ubuntu Setup](#basic-ubuntu-setup)
1. [Basic Security Setup](#basic-ubuntu-setup)
1. [Reverse DNS](#reverse-dns)
1. [Software](#software)
1. [References](#references)

---
## Provision

To provision a new [Linode](http://linode.com) server, see the getting started [guide](https://www.linode.com/docs/getting-started). After a few minutes, a [Linode](http://linode.com) server will be provisioned. This guide assumes you have chosen `Ubuntu 14.04 LTS` as your base image.


---
## Remote Access

To connect to a [Linode](http://linode.com) server,

``` bash
$ ssh root@<your_server_ip>
```

where your server IP may be found via 

```
Linodes --> dashboard --> Remote Access
```

You will be prompted for the `root` password you created when deploying a new image to your [Linode](http://linode.com).


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

#### Two-Factor Authentication

You are encouraged to setup two-factor authentication when logging in to the Linode manager. From the Linode manager, navigate to

```
My Profile --> Password & Authentication --> Two-Factor Authentication
```


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
$ sudo nano /etc/ssh/sshd_config
```

Change `PasswordAuthentication` to `no`. You may need to remove the `#` symbol to uncomment the line.

```
PasswordAuthentication no
```

Change the `PermitRootLogin` option to `no`.

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

Set the `Port` to a value less than `1024`; e.g., `567`.

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

While DNS allows a domain name to be resolved to an IP address, reverse DNS allows an IP address to be resolved to a domain name. Reverse DNS is useful when network troubleshooting (`ping`, `traceroute`). From the Linode manager, navigate to

```
Linodes --> Remote Access --> Reverse DNS
```

and enter your domain.


---
## Software


#### Compiler 

A compiler is often required to install packages and software. To install one,

``` bash
$ sudo apt-get install build-essential
```



#### Git

Install [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git):

``` bash
$ sudo apt-get install git
```

Once installed, configure for use with [Github](https://help.github.com/articles/set-up-git/).

``` bash
$ git config --global user.name "<your_github_username>"
$ git config --global user.email "<your_github_email>"
```

To interact with Github over SSH, first check for any existing SSH keys on your server.

``` bash
$ ls -al ~/.ssh 
```

If you see an existing private-public key pair which you can use with Github, skip the next step. To generate a new SSH key,

``` bash
$ ssh-keygen -t rsa -b 4096 -C "<your_github_email>"
```

Accept the default file name. When prompted, enter a __strong__ passphrase. Once a key is generated, you will see a key fingerprint in your terminal.

To configure the [ssh-agent](https://en.wikipedia.org/wiki/Ssh-agent) program to use your SSH key, start the agent in the background

``` bash
$ eval "$(ssh-agent -s)"
```

and add your SSH key

``` bash
$ ssh-add ~/.ssh/id_rsa
```

where `<id_rsa>` corresponds to the name of an existing SSH key, if not generated above. Next, copy the public key,

``` bash
$ cat ~/.ssh/id_rsa.pub
```

and add to your list of SSH keys on [Github](https://help.github.com/articles/generating-ssh-keys/#platform-linux). To verify that everything is working,

``` bash
$ ssh -T git@github.com
```

Verify the fingerprint and type `yes`. If the user name matches yours, everything works as expected.



#### Python

Install Python environment:

``` bash
$ sudo apt-get install python-pip python-dev
$ sudo pip install virtualenv
```

The above creates a global `pip` command to install Python packages. You are encouraged __not__ to use it, as packages will be installed globally. Instead, use `virtualenv`.

To create a new virtual Python environment,

``` bash
$ virtualenv --distribute <environment_name>
```

where `<environment_name>` is the name of your choosing; e.g., `python/dev`. To switch to the new environment,

``` bash
$ cd <environment_name>
$ source bin/activate
```

`pip` can then be used inside of `virtualenv`.

``` bash
$ pip search <package_name>
$ pip install <package_name>
```

To exit a virtual environment,

``` bash
$ deactivate
```


#### Golang

To install [Golang](https://github.com/golang/go/wiki/Ubuntu), first download a Golang binary

``` bash
$ sudo wget https://storage.googleapis.com/golang/go$VERSION.$OS-$ARCH.tar.gz
```

where `$VERSION` is your desired version (e.g., `1.5`), `$OS` is your operating system (e.g., `linux`), and `$ARCH` is your architecture (e.g., `amd64`). See the Golang [downloads](https://golang.org/dl/) for a list of possible downloads. Next, extract the archive into `/usr/local` 

``` bash
$ sudo tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
```

For example,

``` bash
$ sudo wget https://storage.googleapis.com/golang/go1.5.linux-amd64.tar.gz -P ./downloads
$ sudo tar -C /usr/local -xzf ./downloads/go1.5.linux-amd64.tar.gz
```

Add `/usr/local/go/bin` to the `PATH` environment variable. Open your `~/.profile`

``` bash
$ sudo nano ~/.profile
```

and add the following line:

```
export PATH=$PATH:/usr/local/go/bin
```

To apply the changes to the current shell,

``` bash
$ source ~/.profile
```

To verify that Golang is working,

``` bash
$ golang version
```

Next, create a Golang workspace

``` bash
$ mkdir -p code/go
```

and update your `~/.profile`

``` bash
$ sudo nano ~./profile
```

to include

``` 
export GOPATH=$HOME/code/go
export PATH=$PATH:$GOPATH/bin
```

Reload your `.profile`:

``` bash
$ source ~/.profile
```

Verify that everything is working:

``` bash
$ echo $PATH
$ echo $GOPATH
```

Next, let's create a test package.

``` bash
$ mkdir -p $GOPATH/src/github.com/<your_github_username>/hello
$ cd $GOPATH/src/github.com/<your_github_username>/hello
$ touch hello.go
```

Open the file `./hello.go`

``` bash
$ nano ./hello.go
```

and paste

``` 
package main

import "fmt"

func main() {
	fmt.Printf("Hello, world.\n")
}
```

Save the file and install the program:

``` bash
$ go install
```

To run the program,

``` bash
$ hello
# => Hello, world.
```


#### Node.js

Install the [Node version manager](https://github.com/creationix/nvm) (NVM):

``` bash
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.26.1/install.sh | bash
```

Reload your shell

``` bash
$ source ~/.profile
```

and verify that `nvm` is working

``` bash
$ nvm
```

To install the latest version of Node,

``` bash
$ nvm install v0.12
```

Check the installed Node version:

``` bash
$ node --version
```


#### Nginx

Install [Nginx](https://www.nginx.com/), a high performance [web server](https://en.wikipedia.org/wiki/Nginx) with a strong focus on concurrency:

``` bash
$ sudo apt-get install nginx
```

To start [Nginx](https://www.nginx.com/),

``` bash
$ sudo service nginx start
```







---
## References

*	[Getting started](https://www.linode.com/docs/getting-started)
*	[Securing your server](https://www.linode.com/docs/security/securing-your-server/)
*	[Hosting a website](https://www.linode.com/docs/websites/hosting-a-website/)
*	[How to setup your linode](http://feross.org/how-to-setup-your-linode/)

