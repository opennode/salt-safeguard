# salt-safeguard

Features:
* environment shell - explicitly select and work safely within single salt environment at the time
* limit users access and their salt execution rights per each salt environment (through sudo)
* safeguard against accidental salt environment change/targeting (users can only target hosts inside the environment shell they entered)
* make change execution explicit (enforcing that by default salt states are run with test=True flag set)
* enforcing safe change process (ie multi-step process where changes get presented and user confirmation is requested before these changes are actually being applied to the system)
* short hostname expansion within current work environment scope - ie when entering into prd environment, short hostnames host01 and prd-host01 will be expanded into FQDN hostname (ie prd-host01.domain.example)
* tab autocompletion support for target hosts and states under environment shell

## Setup

```bash
# install jq dependency (needs EPEL7 repo on CentOS7)
yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
yum install -y jq

# clone salt-safeguard git repo
git clone https://github.com/opennode/salt-safeguard.git
cd salt-safeguard

# install files
mkdir -p /usr/lib/salt-safeguard
cp -p files/usr/lib/salt-safeguard/salt-safeguard.wrapper /usr/lib/salt-safeguard/
chown root:root /usr/lib/salt-safeguard/salt-safeguard.wrapper
cp -p files/usr/bin/salt-safeguard /usr/bin/
chown root:root /usr/bin/salt-safeguard
cp -p files/usr/sbin/salt-safeguard-hostlist /usr/sbin/
chown root:root /usr/sbin/salt-safeguard-hostlist

# OPTIONAL but recommended: add "test: True" into salt master config
vim /etc/salt/master.d/master-environments.conf
/etc/init.d/salt-master restart

# add system groups for salt user roles (saltmasters=>read-write access, saltusers=>read-only access)
groupadd saltmasters
groupadd saltusers

# adding users to groups
groupmems -g saltmasters -a <user>
# listing users in group
groupmems -g saltmasters -l

# create config directory for salt-safeguard environments
mkdir -p /etc/salt/environments.d

# create /etc/salt/environments.d/<ENV_NAME>.conf file
cat << EOF > /etc/salt/environments.d/prd.domain.example
# salt environment name/id
ENV_SALT_ID="prd"
# domain linked to environment
ENV_HOST_DOMAIN="domain.example"
# hosts shortname (ie hostname -s) wildcard mask - example: <env>-* or *-<env> 
ENV_HOST_SHORTNAME_MASK="prd-*"
# environment name within salt-safeguard
ENV_NAME="prd.domain.example"
# environment description within salt-safeguard
ENV_DESC="Production Environment"
# environment shell prompt color
ENV_COLOR=$( echo -e "\e[91m" )
EOF

# create /etc/salt/environments.d/<ENV_NAME>.sudo file
visudo -f /etc/salt/environments.d/prd.domain.example.sudo
--- EXAMPLE ---
Cmnd_Alias PRD_SALT_RW_CMD=/usr/bin/salt prd-*.domain.example state.sls * env=prd --state-output=* test=False
Cmnd_Alias PRD_SALT_RO_CMD=/usr/bin/salt prd-*.domain.example state.sls * env=prd --state-output=* test=True
Cmnd_Alias PRD_SALT_PI_CMD=/usr/bin/salt prd-*.domain.example test.ping
Cmnd_Alias PRD_SALT_HL_CMD=/usr/sbin/salt-safeguard-hostlist prd-*.domain.example
%saltusers ALL=(root) NOPASSWD: PRD_SALT_RO_CMD 
%saltusers ALL=(root) NOPASSWD: PRD_SALT_PI_CMD
%saltusers ALL=(root) NOPASSWD: PRD_SALT_HL_CMD 
%saltmasters ALL=(root) PASSWD: PRD_SALT_RW_CMD 
--- EXAMPLE ---

# link env.sudo file
# NB! sudoers.d does not allow dots in the filename!
ln -sf /etc/salt/environments.d/prd.domain.example.sudo /etc/sudoers.d/prd-domain-example
```

## Usage

```bash

Usage: salt-safeguard <command>

Commands:

	enter <env>	# Enter into salt environment
	list		# List available environments
	help | usage	# Show usage instructions


[admin@saltmaster ~]$ salt-safeguard enter prd.domain.example
[ prd.domain.example ]$ help

Usage:
	<cmd> <targets> [state.name]

commands:

	salt-ping <targets>			# Issue PING test on target(s)
	salt <targets> <state>			# Issue state check on target(s)
	salt-diff <targets> <state>		# Issue state check on target(s) with config diffs shown
	salt-apply <targets> <state>		# Apply state on target(s)
	salt-list states | hosts		# List available states or hosts (minions)
	help					# Show usage instructions

targets:

	* can be any host inside this environment - matching the environment hostfilter
	* wildcarding is supported - ie prd-*.domain.example is a valid match/mask for all hosts inside prd environment
	* tab autocompletion is supported for hosts lookup
	* short hostnames expansion is supported - ie both host01 and prd-host01 will be expanded to fqdn inside selected environment

state:
	* name of the state to run - that is available under current environment
	* tab autocompletion is supported for state files lookup

```
