---
author: Chris
title: Graylog on Proxmox with Ansible for Rsyslog Client Config
date: 2025-01-24
tags:
  - proxmox
  - graylog
---

Content Overview

- {% linkprimary "Introduction", "https://christopherbauer.xyz/blog/graylog-revised/#introduction" %}
- {% linkprimary "Creating the VM", "https://christopherbauer.xyz/blog/graylog-revised/#creating-the-vm" %}
- {% linkprimary "Installation & Configuration", "https://christopherbauer.xyz/blog/graylog-revised/#installation-and-configuration" %}
- {% linkprimary "Automating Client Rsyslog Configurations with Ansible", "https://christopherbauer.xyz/blog/graylog-revised/#automating-client-rsyslog-configurations-with-ansible" %}

### Introduction

After some thought, I realized installing Graylog is sufficiently complicated that I'm revising my original post to include installation steps.

In this post I first outline two obstacles to be aware of in creating a VM on Proxmox for Graylog. I then cover the installation steps for OpenSearch, MongoDB and Graylog. In the final part, I offer an overview of how you can use Ansible to configure clients to send Graylog logs using rsyslog.

#### Assumptions

The approach of this post assumes the reader is comfortable with the Linux command line and desires to install the Graylog Open version on a Proxmox Ubuntu 22.04 virtual machine, and wants to install OpenSearch on the self-same machine as Graylog. These instructions don't consider use of a proxy or TLS requirements.

One note of warning: be prepared for Graylog's RAM consumption. Graylog is a RAM hog. You'll see your RAM usage shoot up so it'd pay to go into this project with your eyes open.

### Creating the VM

I'm going to cover two notes on how to make an install of Graylog on a Proxmox VM go more smoothly. If I'd known about these challenges ahead of time I could have saved myself a little pain.

#### In Brief

Stick with Ubuntu 22.04 and select "Host" as the CPU for the VM creation parameters.

#### In Detail

I went with a VM because Graylog is a bit of a beast, requiring multiple different components and lots of RAM. I believe people have installed it on LXC containers, but I figured I'd run into fewer problems this way. Well, at least fewer problems than I would otherwise. Not absent problems.

The **first** issue was relatively trivial but time consuming to figure out. I figured I'd put Graylog on the latest Ubuntu LTS (24.04). It took a full install cycle using the [Ubuntu Debian-Installer](https://wiki.ubuntu.com/Installer/FAQ) and troubleshooting to realize that MongoDB wouldn't play nice with that version. MongoDB is required for Graylog's function so this wasn't something I could figure out later. Lesson learned. As of this writing, I used 22.04.

The **second** was that MongoDB[ doesn't play nice with Proxmox's defaults](https://forum.proxmox.com/threads/mongo-db-5-0-not-install.95857/post-440005). I had learned and subsequently forgotten this one from a previous encounter with MongoDB. In particular, I had to tweak an option that I otherwise never change regarding the processor, as MongoDB cannot work on the default selection of x86-64-v2-AES. In the VM creation guide, on the CPU tab, for "Type," I had to set the processor to _host_ in order for it to function correctly in the VM.
{% imagesmall '/img/2025-01-21_11-41.png', '' %}

### Installation & Configuration

Regarding the install process, we'll work through four stages:

1. Install & configure OpenSearch to store the log entries that Graylog collects
2. Install MongoDB to store Graylog's metadata
3. Install the Graylog Data Node
4. Install the Graylog server

#### Installing OpenSearch

I'm borrowing from [this Graylog repo's instructions](https://github.com/Graylog2/se-poc-docs/blob/main/src/On%20Prem%20POC/installing%20opensearch.md) on installing OpenSearch.

First, update and upgrade the VM. Restart if required.

Take note of the IP address (if you need to, run `ip a`).

Verify the required dependencies are met.

```bash
sudo apt-get update && sudo apt-get -y install vim lsb-release ca-certificates curl gnupg2
```

Download signing key.

```bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
```

Create repository file.

```bash
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```

Install OpenSearch.

```bash
sudo apt-get update
```

Set a temporary password for installation.

```bash
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=$(tr -dc A-Z-a-z-0-9_@#%^-_=+ < /dev/urandom  | head -c${1:-32}) apt-get -y install opensearch=2.15.0
```

Set default value for heap variable.

```Bash
tmpheap=1
```

#### Configuring OpenSearch

Back up the original OpenSearch configuration.

```bash
sudo cp /etc/opensearch/opensearch.yml /etc/opensearch/opensearch.yml.bak
```

Create the new configuration with parameters set to Graylog's requirements.

```bash
echo "cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
transport.host: 0.0.0.0
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
action.auto_create_index: false
plugins.security.disabled: true
indices.query.bool.max_clause_count: 32768" | sudo tee /etc/opensearch/opensearch.yml
```

#### Installing MongoDB

Once I {% linkprimary "configured the vm properly", "http://christopherbauer.xyz/blog/graylog-revised/#creating-the-vm" %}, I followed [the official docs ](https://go2docs.graylog.org/current/downloading_and_installing_graylog/ubuntu_installation.htm#aanchor21) for an Ubuntu install without problems. Those same instructions appear here.

Set a timezone.

```bash
sudo timedatectl set-timezone UTC
```

Install gnupg.

```bash
sudo apt-get install gnupg curl
```

Import the key for MongoDB.

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg \
   --dearmor
```

Create a list file for MongoDB.

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```

Reload the local package database.

```bash
sudo apt-get update
```

Install the latest stable version of MongoDB.

```bash
sudo apt-get install -y mongodb-org
```

I don't use proxies, so if you do refer to [the official docs ](https://go2docs.graylog.org/current/downloading_and_installing_graylog/ubuntu_installation.htm#aanchor21), then Enable.

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable mongod.service
```

```bash
sudo systemctl restart mongod.service
```

```bash
sudo systemctl --type=service --state=active | grep mongod
```

Freeze the MongoDB version.

```bash
sudo apt-mark hold mongodb-org
```

#### Installing the Graylog Data Node

Get the Graylog repo as a package.

```bash
wget https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.deb
```

Install it.

```bash
sudo dpkg -i graylog-6.1-repository_latest.deb
```

The repositories your package manager checks should now include Graylog's repos. Update them.

```bash
sudo apt-get update
```

Install Graylog.

```bash
sudo apt-get install graylog-datanode
```

#### Configuring the Graylog Data Node

To conform to the OpenSearch requirements check the vm.max count to ensure it is set to at least 26144.

```bash
cat /proc/sys/vm/max_map_count
```

If it isn't, you can use the following:

```bash
vm.max_map_count=262144
```

Reload the config.

```bash
sudo sysctl -p
```

Create a password secret and copy the output to a file editor. You'll need it for the next step and later as well.

```bash
< /dev/urandom tr -dc A-Z-a-z-0-9 | head -c${1:-96};echo;
```

Open the config file using vim (you can use nano if you prefer something straightforward). Paste the above into the `password_secret` line, no quotations.

```bash
sudo vim /etc/graylog/datanode/datanode.conf
```

Enable the Graylog service.

```bash
sudo systemctl enable graylog-datanode.service
```

Ensure it has started.

```bash
sudo systemctl start graylog-datanode
```

#### Install Graylog Server

Install the Graylog server.

```bash
sudo apt-get install graylog-server
```

#### Configure the Graylog Server

Generate a password and copy it before opening the config file in the next step.

```bash
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

Add your password secret generated during the Graylog Node configuration step to the server config file with the line `password_secret`. Then add the password you generated above to the line `root_password_sha2`. Select an alternative username for `root_username` if you desire.

```bash
sudo vim /etc/graylog/server/server.conf
```

Set the `http_bind_address` value in the Graylog configuration file to the public host name or a public IP address for the machine to which you can connect.

```bash
sudo sed -i 's/#http_bind_address = 127.0.0.1.*/http_bind_address = 0.0.0.0:9000/g' /etc/graylog/server/server.conf
```

Enable Graylog during the operating system’s startup.

```bash
sudo systemctl daemon-reloadsudo systemctl enable graylog-server.servicesudo systemctl start graylog-server.servicesudo systemctl --type=service --state=active | grep graylog
```

With that you should be able to proceed to IP assigned to your VM with the port 9000 and find the login page (e.g. http:// 192.168.0.1:9000).

### Automating Client Rsyslog Configurations with Ansible

Take what follows as a rough model rather than specific directions.

Next, there was the matter of configuring my machines [to send logs to Graylog](https://go2docs.graylog.org/current/getting_in_log_data/syslog_inputs.html). Graylog offers a [variety of different ways](https://go2docs.graylog.org/current/getting_in_log_data/inputs.htm) to accept logs from client machines. I went with Rsyslog.

Now, I could have ssh-ed into each individual machine and altered the `/etc/rsyslog.conf` file by hand to include:

```vim
*.*@@yourgraylog.example.org:514;RSYSLOG_SyslogProtocol23Format
```

However, from the start this task seemed ripe for automation.

I built an Ansible script to ensure rsyslog was installed and then to configure it on the clients. I'd been playing around with Ansible over the weekend, and the YAML format is great compared to writing everything out in pure python.

Be aware, while the Ansible code can be found below, there are a bunch of configuration challenges to Ansible that can sidetrack you for hours. I can think of at least two off the top of my head. First, I used the code below as an [Ansible Role](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html). Covering what that means, and how to setup Ansible, is far beyond the scope of this article. If you're new to it, plan some free hours to start toying around. Start with [the docs](https://docs.ansible.com/ansible/latest/getting_started/index.html); they're just average but they're worth it for just getting started. Second, if you use _sudo_ with a password on the target, you'll want to use Ansible-Vault. However, the whole way Ansible Vault works is super unclear to me. I hope to write up a blog post about it sometime. I'm afraid I don't have a confirmed solution for it as of this writing. Also, I can't help if you are thinking of windows hosts, I don't have windows clients.

Because this is a role _main.yml_ file, it may seem truncated. To explain, the first section determines whether rsyslog is installed. The second determines whether the Graylog code for forwarding the logs is already present in the `/etc/rsyslog.conf` file, and if not, adds it to the final line. The final section then restarts the rsyslog server.

```yml
- name: Install rsyslog
  ansible.builtin.apt:
    name: rsyslog

- name: Add instruction to dispatch logs to graylog
  ansible.builtin.lineinfile:
    path: /etc/rsyslog.conf
    line: "*.*@@YOUR_IP_HERE;RSYSLOG_SyslogProtocol23Format"
    regexp: '\*.\*@@YOUR_IP_HERE;RSYSLOG_SyslogProtocol23Format'
    state: present
    insertafter: EOF
    create: true

- name: Restart rsyslog
  ansible.builtin.shell:
    cmd: systemctl restart rsyslog
```

You'd then run that using Ansible as a role in your playbook.

### Conclusion

Admittedly, this is a post with a niche attraction, but nevertheless I hope it may help someone in the future. Should you have any questions, don't hesitate to DM me on {% linkprimary "Mastodon", "https://infosec.exchange/@anthro_packets" %}.
