# Deploying a Stand-Alone ICAT Stack in a Box with Ansible

## Obtain a Virtual Machine to be the Ansble Controller Host

The example given here is for a Centos 7 virtual machine. However, this installation has also been tested and found to be working correctly on Ubuntu Xenial.

## Install the Ansible Software

```Shell
yum install -y ansible git
exit
```

## Configure SSH (Not tested this in new workflow)

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

Get the ICAT Ansible Code from GitHub and create a directory for your Ansible installation playbook: `ansible-install`. We will keep all files relating to your installation in this directory to keep them separate from the ICAT Ansible code. This will allow you to keep your installation playbook and related files in version control and to update the ICAT Ansible code independently.

```Shell
mkdir development
cd development
git clone https://github.com/icatproject-contrib/icat-ansible
mkdir ansible-install
cd ansible-install
```

You may wish to keep track of your changes with git:
```Shell
# in the ansible-install directory
git init
```

Create the `ansible.cfg` file and add an entry to point it to your ICAT Ansible clone:
```INI
[defaults]
roles_path = /PATH/TO/development/icat-ansible/roles
```

## Configure the Target Host (not tested in this workflow)

Create a `hosts` file and add the following to it:

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

[name-of-your-host-group]
hostname.domain
# and / or
localhost ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_user=myuser
```

## Create an Ansible Vault

Create a secure store for usernames and passwords and open the file:

```Shell
ansible-vault create vault.yml
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

If you need to edit the contents of the vault at a later date, use the following command:

```Shell
ansible-vault edit group_vars/all/vault.yml
```

## Create A Playbook For Your Installation
Create a playbook file to hold the list of roles you wish to install and any variables you wish to customise. Here is an example:
```YAML
---
- hosts: name-of-your-host-group
  become: true
  vars_files:
    - 'vault.yml'

  roles:
    - role: 'common'

    - role: 'java'

    - role: 'payara'
      vars:
        # Payara user and admin user's password
        payara_user: "glassfish"
        payara_admin_password: "{{ vault_payara_admin_password }}"

    - role: 'authn-anon'

    - role: 'authn-db'
      vars:
        # ICAT database hostname, database name, username and password
        db_icat_hostname: "{{ vault_db_icat_hostname }}"
        db_icat_port: "{{ vault_db_icat_port }}"
        icat_database: "{{ vault_icat_database }}"
        db_icat_username: "{{ vault_db_icat_username }}"
        db_icat_password: "{{ vault_db_icat_password }}"

    - role: 'authn-ldap'
      vars:
        authn_ldap_provider_url: "ldaps://ldap.example.com"
        authn_ldap_mechanism_enabled: false

    - role: 'icat-lucene'
      vars:
        icat_lucene_data_dir: 'data/icat/lucene'
    
    - role: 'icat-server'
      debugger: on_failed
      vars:
        icat_server_rootUserNames: "{{ vault_icat_server_rootUserNames }}"
        icat_server_maxEntities: "250000"
        icat_server_authn_list: "db ldap"
        icat_server_authn_ldap_admin: "false"
        icat_server_lucene_populateBlockSize: "10000"
        authn_db_url: "{{ ansible_fqdn }}"
        authn_anon_url: "{{ ansible_fqdn }}"
        authn_ldap_url: "{{ ansible_fqdn }}"
        lucene_url: "{{ ansible_fqdn }}"
        icat_url: "{{ ansible_fqdn }}"


    - role: 'dev-common'
```

## Run the Playbook
At this point, you should have 4 files in your `ansible-install` directory: `ansible.cfg`, `hosts`, `vault.yml` and `your_playbook.yml`.

To run the playbook:

```Shell
ansible-playbook --ask-vault-pass your_playbook.yml
```
