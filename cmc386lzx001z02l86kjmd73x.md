---
title: "Bastion Host"
seoTitle: "What Is a Bastion Host? Secure SSH Access Explained with Examples"
seoDescription: "Learn what a Bastion Host is, why it's essential for securing SSH access to private servers, and how to configure one using ProxyJump. Includes practical ex"
datePublished: Thu Jun 19 2025 10:15:42 GMT+0000 (Coordinated Universal Time)
cuid: cmc386lzx001z02l86kjmd73x
slug: bastion-host
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/UVTXR-4wUY0/upload/76f210fc861c43febd6622fd150bacaf.jpeg
tags: ssh, networking, network-security, jumpserver, bastion-hosts

---

Bastion host or server is a server mainly for jumping connection, especially ssh connection. It’s also known as jump server

The reason is mainly for security and to reduce the attack surface. Only bastion server is exposed to the outer world and all rest of the server lives in the internal network or firewall.

This configuration ensures that you only need to take care of the security of one server resulting into strengthen security across all your infra.

### Setup:

```yaml
services:
  bastion:
    build:
      dockerfile: bastion
    ports:
      - '6969:22'

  server1:
    build:
      dockerfile: client


  server2:
    build:
      dockerfile: client
```

Here, I have 3 servers, one is working as bastion host and exposed to the host and rest two are in the separate network not accessible from outside.

Let’s see how the Dockerfile for Bastion and clients look like

```dockerfile
# For Bastion
FROM ubuntu:25.04 # Using ubuntu 25 as my base image

# installing openssh and creating necessary dir
RUN apt update -y && apt install openssh-server -y && mkdir -p /root/.ssh/

# Here I'm copying my ssh config into the container, we'll look at it later
COPY ssh_config/bastion/config /etc/ssh/sshd_config

# Copying my public key, so I can access it
COPY ssh_config/bastion/authorized_keys /root/.ssh/

EXPOSE 22

# changing file permission of my key and testing sshd config 
RUN ls /root/.ssh/ && chmod 700 /root/.ssh/ && \
    chmod 600 /root/.ssh/authorized_keys && \
    sshd -t -f /etc/ssh/sshd_config

CMD ["/usr/sbin/sshd", "-D"]  # Running ssd in foreground
```

```dockerfile
# For client same as bastion, only change is in ssh config, I'll show you that in the next step

FROM ubuntu:25.04

RUN apt update -y && apt install openssh-server -y && mkdir -p /root/.ssh/

COPY ssh_config/client/config /etc/ssh/sshd_config

COPY ssh_config/client/authorized_keys /root/.ssh/

EXPOSE 22

RUN chmod 700 /root/.ssh/ && \
    chmod 600 /root/.ssh/authorized_keys && \
    sshd -t -f /etc/ssh/sshd_config

CMD ["/usr/sbin/sshd", "-D"]
```

The sshd server config for bastion and normal servers:

```bash
# minimal sshd config for bastion

Port 22  # exposing my ssh server to port 22, in prod you can set to some diff port to trick basic bots
LogLevel VERBOSE

# PermitRootLogin should be disbaled and there should be seperate user for jump,
# I kept root cuz of laziness while writing this article
PermitRootLogin yes
PubkeyAuthentication yes  # to authenticate via pub/private keys
AuthorizedKeysFile     .ssh/authorized_keys
PasswordAuthentication no  # to disable login via password

# Below, for user roor we are disabling tunneling, port forwarding, tty session
# Because this server should only act as a jump and shouldn't be allow to access for any other purpose.
# In prod, any other type of access should be done via cloud console
Match User root  
    PermitTTY no
    AllowAgentForwarding no
    AllowStreamLocalForwarding no
    X11Forwarding no
    PermitTunnel no
    GatewayPorts no
    ForceCommand /usr/sbin/nologin
```

The above `sshd` config is to disallow any type of ssh request except the Jump request

```bash
# sshd config for servers behind the bastion server
Port 22
PermitRootLogin yes  # can be kept like this
PubkeyAuthentication yes  # auth via keys are allowed
PasswordAuthentication no  # password login is  disabled
```

The above configuration is for server behind the bastion, and it should only be accessible via bastion. So, these servers should be placed in internal network or there should be proper firewall to disable access from any other source.

That’s all, now to access any server all you have to do is:

`ssh -J <jump_user>@<jump_server> <user>@<actual_server>`  

To avoid running long command you can setup ssh config file:  
Open `~/.ssh/config` and paste

```bash
Host jump-host
    HostName localhost  # jump serevr host
    Port 6969  #jump server port
    User root  # jump server user
    IdentityFile ~/temp/test  # location of the private key
    IdentitiesOnly yes

Host server1
    HostName server1  # actual server host
    User root  # actual server user
    ProxyJump jump-host  # to tell, to use jump server during ssh
    IdentityFile ~/temp/test
    IdentitiesOnly yes

Host server2
    HostName server2
    User root
    ProxyJump jump-host
    IdentityFile ~/temp/test
    IdentitiesOnly yes
```

Now, you can simply do  
`ssh server1` or `ssh server2`, and it’ll leverage the jump server for the connection.

You can check out the whole containerized version of this article [here](https://github.com/Cosmicoppai/bastion):  
[Cosmicoppai/bastion: Demonstration of Bastion Config via containers](https://github.com/Cosmicoppai/bastion)