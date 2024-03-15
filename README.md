# *_This repo is severely out of date and is only here for archival purposes_*
---

_The purpose of this repo is to rapidly deploy [Icinga](https://www.icinga.org/) packages for Centos 7 in [DigitalOcean](https://www.digitalocean.com/) using [Ansible](https://www.ansible.com/)._
_This setup and install process was tested on a CentOS 7.2 x64 2GB DigtalOcean Droplet._

# Setting Up
---
### 1. Create an ansible user with `sudo` priveleges

```
[root@localhost ~]# useradd ansible
[root@localhost ~]# usermod -aG wheel ansible
[root@localhost ~]# passwd ansible
Changing password for user ansible.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[root@localhost ~]# su ansible -l
[ansible@localhost ~]$
```

---
### 2. Install `epel-release` and update yum

```
[ansible@localhost ~]$ sudo yum -y install epel-release
[ansible@localhost ~]$ sudo yum makecache
[ansible@localhost ~]$ sudo yum -y update
```
---
### 3. Install Ansible, git and python-pip

[_Install Ansible - Docs_](http://docs.ansible.com/ansible/intro_installation.html)

```
[ansible@localhost ~]$ sudo yum -y install ansible git python-pip
```
---
### 4. Install the `dopy` package required for the DigitalOcean Dynamic Inventory Script
[_Requirement shown here, on DigitalOcean's community tutorials_](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2-with-ansible-2-0-on-ubuntu-14-04)

```
[ansible@localhost ~]$ sudo pip install 'dopy>=0.3.5,<=0.3.5'
```
---
### 5. Generate an SSH key for your ansible user

```
[ansible@localhost ~]$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ansible/.ssh/id_rsa):
Created directory '/home/ansible/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/ansible/.ssh/id_rsa.
Your public key has been saved in /home/ansible/.ssh/id_rsa.pub.
---OUTPUT OMITTED----
[ansible@localhost ~]$
```
---
### 6. Clone this repository to your local machine and move into that new directory
```
[ansible@localhost ~]$ git clone (the URL repo)
[ansible@localhost ~]$ cd ansible-icinga
[ansible@localhost ansible-icinga]$
```
---
### 7. [Generate a read and write API Token for your DigitalOcean Account](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2#how-to-generate-a-personal-access-token)

---
### 8. Copy this key into your ansible environment
```
[ansible@localhost ansible-icinga]$ vi inventory/digital_ocean.ini
----beginning of inventory/digitalocean.ini----
# Ansible DigitalOcean external inventory script settings
#

[digital_ocean]

# The module needs your DigitalOcean API Token.
# It may also be specified on the command line via --api-token
# or via the environment variables DO_API_TOKEN or DO_API_KEY
#
api_token = YOUR_API_TOKEN_HERE
----rest of file omitted----
```
and

```
[ansible@localhost ansible-icinga]$ vi vars/droplet-details.yml
----beginning of vars/droplet-details.yml----
---
do_token: YOUR-API-TOKEN-HERE
----rest of file omitted----
```

### 9. Copy your SSH key into `files/public-keys/ansible-do.pub`
```
[ansible@localhost ansible-icinga]$ cp ~/.ssh/id_rsa.pub files/public-keys/ansible-do.pub
[ansible@localhost ansible-icinga]$
```
---

## Getting Started and Special Notes
These playbooks are meant to be executed from within the `ansible-icinga` directory, as a user called `ansible`.
If the directons from the `Setup` section above were followed, you should be ready to get started right away.

Here are the important files we should be aware of:

```
├── files
│   └── public-keys
│       └── ansible-do.pub           # Copied from your ansible user's ~/.ssh/id_rsa.pub
├── inventory
│   ├── digital_ocean.ini            # Your API key for DigitalOcean will go in here
│   ├── hosts                        # Provided by Ansible, this is an executable dynamic inventory script
│   └── inventory
├── playbooks
│   ├── do-create.yml                # Uses vars/droplet-details.yml to spin up droplets
│   ├── do-remove.yml                # Also uses the droplet-details.yml file, but to tear down droplets
│   └── icinga-all.yml               # Uses the roles shown below to dole out packages and tasks
├── roles
│   ├── core
│   │   └── tasks
│   │       └── main.yml
│   ├── icinga-masters
│   │   ├── files
│   │   │   └── icinga-syntax
│   │   │       ├── ftdetect
│   │   │       │   └── icinga2.vim
│   │   │       └── syntax
│   │   │           └── icinga2.vim
│   │   └── tasks
│   │       └── main.yml
│   └── icinga-nodes
└── vars
    └── droplet-details.yml          # The other location to paste your DigitalOcean API Key. Also defines common name, size, and region for droplets.
```

After pasting your DigitalOcean API key into `vars/droplet-details.yml` and `inventory/digital_ocean.ini`, as well as defining your droplet parameters in `vars/droplet-details.yml`,
run this command from inside the `ansible-icinga` directory:
```
[ansible@localhost ansible-icinga]$ ansible-playbook playbooks/do-create.yml
```

Because these plays are focused on deploying Icinga packages,
I've chosen `icinga-master` and `icinga-node` as the basis for the droplet naming structure, and subsequent playbook execution.
When the role icinga-masters is used, it is applied to the hosts `icinga-master*` to catch all servers that match,
same thing for `icinga-nodes` role, using `icinga-node*` for it's hosts.

If you use a different naming structure in `vars/droplet-details`, be sure to change the `hosts:` lines in `playbooks/icinga-all.yml`.

---

After creating your droplets, you'll need to wait for them to become available before running the next plays.
In order to check this, I'll typically run `ansible -m ping icinga*` a few times, ad-hoc, to see if they're ready to start accepting commands.

At a severe cost to playbook runtime, you can edit `tasks/digitalocean/create-droplet.yml` and change the `wait=no` line to `wait=yes`

`wait=yes` forces you to wait for each individual droplet to be created before sending the next API command for the following droplet.
By using `wait=no`, we're just telling DigitalOcean to create everything simultaneously. Much faster that way.

Both `tasks/digitalocean/create-droplet.yml` and `tasks/digitalocean/destroy-droplet.yml` use `vars/droplet-details.yml` to get instructions on what to create.

---
## Provisioning the Droplets
Now run `ansible-playbook playbooks/icinga-all.yml` to start getting the rest of the packages and configs deployed.

---
### Default Passwords

For right now, I'm doing passwords the *wrong* way. Settng them statically rather than using ansible vault with protected passwords is definitely the quick and dirty way of doing things.

In `tasks/configs/setup-mariadb.yml` you'll find that during the `mysql_secure_installation` runthrough it changes the default root password for mariadb to `icingaweb2`.
You'll also find in `tasks/icinga2/db/icingaweb2-db.yml` that during the creation of the icingaweb2 database, the `icinga` user has the password `icinga`
Super basic, super insecure. In future revisions of this project we'll be using more appropriate methods for configuring these.


### IcingaWeb on the Master Node
For the master node, there will be a 15 second pause for you to grab your setup token. Once Ansible provides you with that setup token, you can navigate to the icingaweb2 setup page here:

[master-nodes-ip-addr]/icingaweb2/setup

To get your master node's IP address (assuiming that you've followed the default naming structure outlined above) run:
`ansible -m ping icinga-master*`

