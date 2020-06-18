# Deploying a Stand-Alone ICAT Stack in a Box with Ansible

## Obtain a Virtual Machine to be the Ansble Controller Host

The example given here is for a Scientific Linux 7 virtual machine. However, this installation has also been tested and found to be working correctly on Ubuntu Xenial.

### Configure the Ansible Software Repository

Create the repository configuration files:

```Shell
sudo -i
touch /etc/yum.repos.d/ansible-el7-x86_64.repo
touch /etc/yum.repos.d/ansible-el7-x86_64.pkgs
```

Add the following to `/etc/yum.repos.d/ansible-el7-x86_64.repo`:

```
[ansible-el7-x86_64]
name=ansible-el7-x86_64
baseurl=https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/
metadata_expire=7d
include=/etc/yum.repos.d/ansible-el7-x86_64.pkgs
enabled=1
gpgcheck=0
priority=20
skip_if_unavailable=0
```

Add the correct content to `/etc/yum.repos.d/ansible-el7-x86_64.pkgs`:

```Shell
echo '# Additional configuration for ansible-el7-x86_64' > /etc/yum.repos.d/ansible-el7-x86_64.pkgs
```

### CentOS 7 Error
If you're on CentOS 7, you may receive this error if you attempt to install ansible:

``` http://mirrors.gridpp.rl.ac.uk/snapshot/2019-05-20/centos-7x-x86_64/RPMS.updates/git-1.8.3.1-20.el7.x86_64.rpm: [Errno 14] HTTP Error 404 - Not Found ```

The date in the url is causing the problem. To fix this, edit the baseurl property in these files (in `/etc/yum.repos.d`):

```
sl-7x-x86_64-extras.repo
centos-7x-x86_64-os.repo
centos-7x-x86_64-updates.repo
epel-7-x86_64.repo
```

changing ‘2019-05-20’ in the url to ‘2020-05-01’, then running `yum makecache`.


### Install the Ansible Software

```Shell
yum install -y ansible git
exit
```

### Configure Ansible Working Environment

```Shell
sudo sed -i -e "s;^#inventory      = /etc/ansible/.*;inventory      = /home/${USER}/Development/icat-ansible/hosts;g" /etc/ansible/ansible.cfg
sudo sed -i -e "s;^#roles_path    = /etc/ansible/roles$;roles_path    = /home/${USER}/Development/icat-ansible/roles;g" /etc/ansible/ansible.cfg
sudo sed -i -e "s;^#log_path = /var/log/ansible.log$;log_path = /var/log/ansible.log;g" /etc/ansible/ansible.cfg
sudo touch /var/log/ansible.log
sudo chgrp wheel /var/log/ansible.log
sudo chmod g+w /var/log/ansible.log
```

### Configure SSH

If you normally use an ssh key that is protected by a password and wish to use an alternative, passwordless ssh, this can be configured as follows:

Create a passwordless ssh key for the ansible controller:

```Shell
ssh-keygen -N '' -f ~/.ssh/ansible-id_rsa
```

Copy the contents of the public key generated above on the controller host, `~/.ssh/ansible-id_rsa.pub`, into the list of authorised keys, `~/.ssh/authorized_keys` on the target host.

Configure the ansible controller host to use this key when connecting to the target host.

```Shell
echo "Host hostname.domain" >> ~/.ssh/config
echo "  IdentityFile ~/.ssh/ansible-id_rsa" >> ~/.ssh/config
echo "  User ${USER}" >> ~/.ssh/config
```

Check that the ansible controller can connect to the target host using the key created above, without a password, and to add the target host into the list of known hosts, `~/.ssh/known_hosts`.

```Shell
ssh hostname.domain
```

## Obtain the ICAT Ansible Code

Get the ICAT Ansible Code from GitHub

```Shell
mkdir Development
cd Development
git clone https://github.com/icatproject-contrib/icat-ansible
cd icat-ansible/
touch hosts
```

### Configure the Target Host

Add the following to the `hosts` file:

```
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

[icat-test-hosts]
hostname.domain
```

### Create an Ansible Vault

Create a secure store for usernames and passwords and open the file:

```Shell
ansible-vault create group_vars/all/vault.yml
```

The vault should contain the following code, replace the values named `SECRET#` with values of your choice:

```
---

# Payara admin user's password
vault_payara_admin_password: 'SECRET1'

# Database root user's username and password
vault_db_root_username: 'root'
vault_db_root_password: 'SECRET2'

# Icat database name, username and password
vault_icat_database: 'icatdb'
vault_db_icat_username: 'icatdbuser'
vault_db_icat_password: 'SECRET3'

# Authn-simple usernames
vault_authn_root_username: 'root'
vault_authn_ingest_username: 'ingest'
vault_authn_reader_username: 'reader'
vault_authn_icatuser_username: 'icatuser'

# Authn-simple passwords
vault_authn_root_password: 'rootpw'
vault_authn_ingest_password: 'ingestpw'
vault_authn_reader_password: 'readerpw'
vault_authn_icatuser_password: 'icatuserpw'

# Topcat database name, username and password
vault_topcat_database: 'topcatdb'
vault_db_topcat_username: 'topcatdbuser'
vault_db_topcat_password: 'SECRET4'
```

If you need to edit the contents of the vault at a later data, use the following command:

```Shell
ansible-vault edit group_vars/all/vault.yml
```

## Run the Playbook

To run the playbook:

```Shell
ansible-playbook --ask-vault-pass site.yml
```
