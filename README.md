
# Setup an Eth2 Mainnet Validator System on Debian

This document contains instructions for setting up an Eth2 mainnet staking system. Medalla testnet instructions are available [here](prysm-medalla.md).

These instructions have been developed to configure an Eth2 mainnet staking system using Debian 10.9 "buster" LTS on an HP Proliant ML350 G9 with eight 1 TB SSD in RAID 10 and 64GB RAM. These instructions are primarily for my own purposes, so that I can recreate my environment if I need to. They are not intended to represent best practices and may not be applicable to your hardware, software, or network configuration. There are many other good sources for instructions on setting up these services, and those may be more generally written and applicable.

Setup includes installation and configuration of the following services, including setting up systemd to automatically run services, where applicable:

- Prysm Beacon Chain
- Prysm Validator
- geth 
- Prometheus
- Grafana
- node_exporter
- blackbox_exporter
- json_exporter

Steps to install and configure all software have been copied from or inspired by a number of sources, which are cited at the end of this file. Discord discussions may have provided additional details or ideas. In addition, though I have never been a professional Linux administrator, I have many years experience running Linux servers for a variety of public and private hobby projects, which may have informed some of my decisions, for better or worse.

This process assumes starting from first login on a clean Debian 10.9 "buster" installation, and were last tested on May 21, 2021.

## Prerequisities

### BIOS Update
If you have not updated the BIOS on your system, find and follow the manufacturer instructions for updating the BIOS. An updated BIOS may improve system performance or repair issues with your system. Instructions will vary dependent on the hardware you are using, but the following links should direct HP Proliant ML350 G9 users to appropriate instructions.

https://support.hpe.com/hpesc/public/km/product/1009483731/Product#t=DriversandSoftware&sort=relevancy&layout=table&numberOfResults=25

### Configure Behavior After Power Failure
After a power failure, you may want your staking system to automatically restart and resume staking. Unfortunately, this is not the default behavior of many systems. Please check your system documentation to determine how to change this behavior in the system UEFI. For an HP Proliant ML350 G9, please check the following instructions.

- [HP Proliant M350 G9 User Manual](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=emr_na-c04431235)

### Software Update
After an initial install, it is a good idea to update everything to the latest versions.
```console
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
sudo apt autoremove
sudo systemctl reboot
```

### Set Time Zone
Run the following command to see the list of time zones, then copy the appropriate time zone to your clipboard.
```console
timedatectl list-timezones
```

Run the following command, replacing `<SELECTED_TIMEZONE>` with the time zone you have copied onto your clipboard.
```console
sudo timedatectl set-timezone <SELECTED_TIMEZONE>
```

### Create a non-root user with sudo privileges

Make a habit of logging to your server using a non-root account. This will prevent the accidental deletion of files if you make a mistake. For instance, the command rm can wipe your entire server if run incorrectly using by a root user.
Tip: Do NOT routinely use the root account. Use su or sudo,  always.
SSH to your server with your SSH client
```console
ssh username@server.public.ip.address
# example
# ssh myUsername@77.22.161.10
```
### Create a new user called 'ethereum'

```console
sudo useradd -m -s /bin/bash ethereum
```
Set the password for ethereum user

[Don't re-use passwords!](https://xkcd.com/792/)


[Choose passwords easy for humans to remember, but hard for computers to crack](https://xkcd.com/936/)

```console
sudo passwd ethereum
```

Add ethereum to the sudo group
```console
sudo usermod -aG sudo ethereum
```
### Disable SSH password Authentication and Use SSH Keys only

The basic rules of hardening SSH are:
 - No password for SSH access (use private key)
 - Don't allow root to SSH (the appropriate users should SSH in, then su or sudo)
 - Use sudo for users so commands are logged
 - Log unauthorized login attempts (and consider software to block/ban users who try to access your server too many times, like fail2ban)
 - Lock down SSH to only the ip range your require (if you feel like it)


Create a new SSH key pair on your local machine. Run this on your local machine (NOT the server!). You will be asked to type a file name in which to save the key. This will be your keyname.

Generate a public-private key pair using the ED25519 key algorithm.
```console
ssh-keygen -t ed25519
```

Transfer the public key only (Never transfer the private key to any other device than the one it was generated on!) to your remote node. Update keyname.pub appropriately.
```console
ssh-copy-id -i $HOME/.ssh/keyname.pub ethereum@server.public.ip.address
```
Login with your new ethereum user
```console
ssh ethereum@server.public.ip.address
```
Disable root login and password based login. Edit the /etc/ssh/sshd_config file
```console
sudo nano /etc/ssh/sshd_config
```
Locate ChallengeResponseAuthentication and update to no
```
ChallengeResponseAuthentication no
```
Locate PasswordAuthentication update to no
```
PasswordAuthentication no
```
Locate PermitRootLogin and update to no
```
PermitRootLogin no
```
Locate PermitEmptyPasswords and update to no
```
PermitEmptyPasswords no
```

Optional: Locate Port and customize it your random port.
> Despite what you may have heard, changing the port doesn't provide any serious defense against a targetted attack. If your server is being targetted then they will port scan you and find out where your doors are. However, moving SSH off the default port of 22 will deter some of the non-targetted and amateur script kiddie type attacks. These are relatively unsophisticated users who are using scripts to port scan large blocks of IP addresses at a time specifically to see if port 22 is open and when they find one, they will launch some sort of attack on it (brute force, dictionary attack, etc). If your machine is in that block of IPs being scanned and it is not running SSH on port 22 then it will not respond and therefore will not show up in the list of machines for this script kiddie to attack. Ergo, there is some low-level "security through obscurity" provided but only for this type of opportunistic attack. By way of example, if you have the time - log dive on your server (assuming SSH is on port 22) and pull out all the unique failed SSH attempts that you can. Then move SSH off that port, wait some time, and go log diving again. You will undoubtedly find less attacks. I use Fail2Ban on my server anyway, so these types of script-kiddie attacks are really, really obvious, but easy enough to protect against.

Use a random port # from 1024 thru 49141. Check for possible conflicts. 
```
Port <port number>
```
save and close the /etc/ssh/sshd_config file.

Validate the syntax of your new SSH configuration.
```console
sudo sshd -t
```
If no errors with the syntax validation, reload the SSH process
```console
sudo service sshd reload
```
Verify the login still works
```console
ssh -i <path to your SSH_key_name.pub> ethereum@server.public.ip.address
```

Alternatively, you might need to add the -p <port#> flag if you used a custom SSH port.
```console
ssh -i <path to your SSH_key_name.pub> ethereum@server.public.ip.address -p <custom port number>
```

It's critically important to keep your system up-to-date with the latest patches to prevent intruders from accessing your system.

### Disable root account
System admins should not frequently log in as root in order to maintain server security. Instead, you can use sudo execute that require low-level privileges.

 To disable the root account, simply use the -l option. 

 ```console
sudo passwd -l root
```
 
If for some valid reason you need to re-enable the account, simply use the -u option.
```
# sudo passwd -u root
```

# Setup Two Factor Authentication for SSH
SSH, the secure shell, is often used to access remote Linux systems. Because we often use it to connect with computers containing important data, it’s recommended to add another security layer. Here comes the two factor authentication (2FA).
 ```console
sudo apt install libpam-google-authenticator -y
 ```
To make SSH use the Google Authenticator PAM module, edit the /etc/pam.d/sshd file:
 ```console
sudo nano /etc/pam.d/sshd
 ```
Add the following line:
 ```
auth required pam_google_authenticator.so
 ```
 save and close
 
Now you need to restart the sshd daemon using:
 ```console
sudo systemctl restart sshd.service
 ```
Modify /etc/ssh/sshd_config
 ```console
sudo nano /etc/ssh/sshd_config
 ```
Locate ChallengeResponseAuthentication and update to yes
 ```
ChallengeResponseAuthentication yes
 ```
Locate UsePAM and update to yes
 ```
UsePAM yes
 ```
Save the file and exit.

Run the google-authenticator command.
 ```console
google-authenticator
 ```
It will ask you a series of questions, here is a recommended configuration:
- Make tokens “time-base”": yes
- Update the .google_authenticator file: yes
- Disallow multiple uses: yes
- Increase the original generation time limit: no
- Enable rate-limiting: yes

You may have noticed the giant QR code that appeared during the process, underneath are your emergency scratch codes to be used if you don’t have access to your phone: write them down on paper and keep them in a safe place.

Now, open Google Authenticator on your phone and add your secret key to make two factor authentication work.

Because we’ll be making SSH changes over SSH, it’s important never to close your initial SSH connection. Instead, open a second SSH session to do testing. This is to avoid locking yourself out of your server if there was a mistake in your SSH configuration. Once everything works, then you can safely close any sessions. Another safety precaution is to create a backup of the system files you will be editing, so if something goes wrong, you can\srevert to the original file and start over again with a clean configuration.

To begin, make a backup of the sshd configuration file:
 ```console
sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
  ```
Now open the file using nano or your favorite text editor:
 ```console
sudo nano /etc/pam.d/sshd
  ```
Add the following line to the bottom of the file:
 ```
@include common-password
auth required pam_google_authenticator.so 
 ```
 
Save and close the file.

### Configure SSH to support this kind of authentication.

First make a backup of the file:
 ```console
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
 ```
 
Now open the SSH configuration file for editing:
 ```console
sudo nano /etc/ssh/sshd_config
  ```
Look for ChallengeResponseAuthentication and set its value to yes:

 ```
ChallengeResponseAuthentication yes
 ```
  
Save and close the file, then restart SSH to reload the configuration files. Restarting the sshd service won’t close our current open connections, meaning you won’t risk locking yourself out with this command:
 ```console
sudo systemctl restart sshd.service
  ```
To test that everything’s working so far, open ANOTHER terminal and try logging in over SSH. It is very important that you keep your current SSH session open and test with an additional session, or you will lock yourself out at some point and will need to use the web console to get yourself back in.

Note: If you’ve previously created an SSH key and are using it, you’ll notice you didn’t have to type in your user’s password or the MFA verification code. This is because an SSH key overrides all other authentication options by default. Otherwise, you should have gotten a password and verification code prompt.

Next, to enable an SSH key as one factor and the verification code as a second, we need to tell SSH which factors to use and prevent the SSH key from overriding all other types.

### Making SSH Aware of MFA
MFA is still not working if you are using and SSH key. To make SSH aware of MFA, reopen the sshd configuration file:
  ```console
sudo nano /etc/ssh/sshd_config
   ```
Add the following line at the bottom of the file. This tells SSH which authentication methods are required. We tell SSH that users need an SSH key and either a password or a verification code (or all three):

  ```
AuthenticationMethods publickey,password publickey,keyboard-interactive
   ```
Save and close the file.

Next, open the PAM sshd configuration file again:
  ```console
sudo nano /etc/pam.d/sshd
   ```
Find the line @include common-auth and comment it out by adding a # character as the first character on the line. This tells PAM not to prompt for a password:
  ```
#@include common-auth
  ```
Save and close the file, then restart SSH:
  ```console
sudo systemctl restart sshd.service
   ```
Now try logging into the server again with a different terminal session/window. Unlike last time, SSH should ask for your verification code. Enter it, and you will complete the login. Even though there is no indication that your SSH key was used, your login attempt used two factors. If you want to verify this, you can add -v (for verbose) after the SSH command.

The -v switch will produce an output like this:

Example SSH output\
>. . . debug1: Authentications that can continue: publickey
>debug1: Next authentication method: publickey
>debug1: Offering RSA public key: /Users/sammy/.ssh/id_rsa
>debug1: Server accepts key: pkalg rsa-sha2-512 blen 279
>Authenticated with partial success.
>debug1: Authentications that can continue: password,keyboard-interactive
>debug1: Next authentication method: keyboard-interactive
>Verification code:

Towards the end of the output, you’ll see where SSH uses your SSH key and then asks for the verification code. You can now log in over SSH with an SSH key and a one-time password. If you want to enforce all three authentication types, you can follow the next step.

Congratulations, you’ve successfully added a second factor when logging in remotely to your server over SSH. If this is what you wanted — to use your SSH key and a TOTP token to enable MFA for SSH (for most people, this is the optimal configuration) — then you’re done.

### Secure Shared Memory
One of the first things you should do is secure the shared memory used on the system. If you're unaware, shared memory can be used in an attack against a running service. Because of this, secure that portion of system memory. 

To learn more about secure shared memory, read this [techrepublic.com](https://www.techrepublic.com/article/how-to-enable-secure-shared-memory-on-ubuntu-server/) article.

### One exceptional case
There may be a reason for you needing to have that memory space mounted in read/write mode (such as a specific server application like DappNode that requires such access to the shared memory or standard applications like Google Chrome). In this case, use the following line for the fstab file with instructions below.
  
### Use with caution
With some trial and error, you may discover some applications(like DappNode) do not work with shared memory in read-only mode. For the highest security and if compatible with your applications, it is a worthwhile endeavor to implement this secure shared memory setting.


Edit /etc/fstab
   ```console
sudo nano /etc/fstab
   ```
Insert the following line to the bottom of the file and save/close. This sets shared memory into read-only mode.
   ```
tmpfs    /run/shm    tmpfs    ro,noexec,nosuid    0 0
   ```
Reboot the node in order for changes to take effect.
   ```console
sudo systemctl reboot
   ```

### Install Fail2ban

Fail2ban is an intrusion-prevention system that monitors log files and searches for particular patterns that correspond to a failed login attempt. If a certain number of failed logins are detected from a specific IP address (within a specified amount of time), fail2ban blocks access from that IP address.

   ```console
sudo apt install -y fail2ban
   ```
Edit a config file that monitors SSH logins.
   ```console
sudo nano /etc/fail2ban/jail.local
   ```
Add the following lines to the bottom of the file.
   ```
# Example
ignoreip = 192.168.1.0/24 127.0.0.1/8 
[sshd]
enabled = true
port = <22 or your random port number>
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
# whitelisted IP addresses
ignoreip = <list of whitelisted IP address, your local daily laptop/pc>
   ```
Save/close file.

Restart fail2ban for settings to take effect.
   ```console
sudo systemctl restart fail2ban
   ```
   
### Configure your Firewall
The standard UFW firewall can be used to control network access to your node.

Install UFW
```console
sudo apt install -y ufw
```

With any new installation, ufw is disabled by default. Enable it with the following settings.

- Port 22 (or your random port #) TCP for SSH connection

Ports for p2p traffic:

- Lighthouse uses port 9000 tcp/udp
- Teku uses port 9000 tcp/udp
- Prysm uses port 13000 tcp and port 12000 udp
- Nimbus uses port 9000 tcp/udp
- Lodestar uses port 30607 tcp and port 9000 udp
- Port 30303 tcp/udp eth1 node


By default, deny all incoming and outgoing traffic
```console
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Allow ssh access
```console
sudo ufw allow ssh #<port 22 or your random ssh port number>/tcp
```
Allow p2p ports
```console
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp
```
Allow eth1 port
```console
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
```
Enable firewall
```console
sudo ufw enable
```
Verify status
```console
sudo ufw status numbered
```

Do not expose Grafana (port 3000) and Prometheus endpoint (port 9090) to the public internet as this invites a new attack surface! A secure solution would be to access Grafana through a ssh tunnel with Wireguard.

Only open the following ports on local home staking setups behind a home router firewall or other network firewall.
It is dangerous to open these ports on a VPS/cloud node.

Allow grafana web server port
```console
sudo ufw allow 3000/tcp
```
Enable prometheus endpoint port
```console
sudo ufw allow 9090/tcp
```
Confirm the settings are in effect.

Verify status
```console
sudo ufw status numbered
```

sudo ufw allow from <your local daily laptop/pc>
Example
```console
sudo ufw allow from 192.168.50.22
```
 
 Port Forwarding Tip: You'll need to forward and open ports to your validator. Verify it's working with https://www.yougetsignal.com/tools/open-ports/ or https://canyouseeme.org/ .


Verify Listening Ports
If you want to maintain a secure server, you should validate the listening network ports every once in a while. This will provide you essential information about your network.
```console
sudo ss -tulpn
```
```console
sudo netstat -tulpn
```
Assign static internal IPs to both your validator node and daily laptop/PC. This is useful in conjunction with ufw and Fail2ban's whitelisting feature. Typically, this can be configured in your router's settings. Consult your router's manual for instructions.

### Power Outage

In case of power outage, you want your validator machine to restart as soon as power is available. In the BIOS settings, change the Restore on AC / Power Loss or After Power Loss setting to always on. Better yet, install an Uninterruptable Power Supply (UPS).

### Clear the bash history

When pressing the up-arrow key, you can see prior commands which may contain sensitive data. To clear this, run the following:
```console
shred -u ~/.bash_history && touch ~/.bash_history
```


### net-tools
Installing net-tools in order to determine network device via ifconfig.
```console
sudo apt install -y net-tools
```

### make
```console
sudo apt install -y make
```

### curl
Debian 10.9 "buster" users need to install curl to continue.
```console
sudo apt install -y curl
```

## geth
A geth full node is required to provide access to deposits made to the deposit contract. It could take many days for geth to sync, so start this process immediately.

### Install geth

On Debian, the process is essentially the same as Ubuntu, it's just not as automatic. As of the writing of this documentation, the ubuntu repository hosting the ethereum software is compatible with Debian 10.9 "buster". Start by creating a file at /etc/apt/sources.list.d/ethereum.list. 
```
sudo nano /etc/apt/sources.list.d/ethereum.list
```

In that file, place the following two lines.

```
deb http://ppa.launchpad.net/ethereum/ethereum/ubuntu bionic main 
deb-src http://ppa.launchpad.net/ethereum/ethereum/ubuntu bionic main
```
Save and exit.

Next, you'll have to import the GPG key for the PPA.
```
sudo apt-key adv --keyserver keyserver.ubuntu.com  --recv-keys 2A518C819BE37D2C2031944D1C52189C923F6CA9
```
After Apt's done importing the key, update your system, and install the Ethereum package.
```
sudo apt update
sudo apt install -y ethereum
```

Confirm that ethereum is installed:
```
dpkg -l | grep ethereum
ii  ethereum     1.9.25+build24398+bionic     amd64   Meta-package to install geth and other tools
```

### Create User Account

```console
sudo adduser --home /home/geth --disabled-password --gecos 'Go Ethereum Client' geth
```

### Set Up systemd Service File
This sets up geth to automatically run on start.

```console
sudo nano /etc/systemd/system/geth.service
```

Copy and paste the following text into the geth.service file.

```
[Unit]
Description=Ethereum 1 Go Client
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=geth
WorkingDirectory=/home/geth
ExecStart=/usr/bin/geth --http --http.addr 0.0.0.0

[Install]
WantedBy=multi-user.target
```

### Start geth

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start geth
sudo systemctl enable geth
```

Confirm that geth is running:
```
sudo systemctl status geth
```

You can check the status of the blockchain sync with the following commands
```
sudo geth attach ipc://home/geth/.ethereum/geth.ipc
eth.syncing
exit
```

## Prysm

### Create User Accounts
```console
sudo adduser --home /home/beacon --disabled-password --gecos 'Ethereum 2 Beacon Chain' beacon
sudo adduser --home /home/validator --disabled-password --gecos 'Ethereum 2 Validator' validator
sudo -u beacon mkdir /home/beacon/bin
sudo -u validator mkdir /home/validator/bin
```

### Install prysm.sh

```console
cd /home/validator/bin
sudo -u validator curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && sudo -u validator chmod +x prysm.sh
cd /home/beacon/bin
sudo -u beacon curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && sudo -u beacon chmod +x prysm.sh
```

### Set Up systemd Service File
This sets up prysm.sh to automatically run on start. This file is slightly different than the version under the Building Prysm section.

#### Beacon Chain
```console
sudo nano /etc/systemd/system/beacon-chain.service
```

Copy and paste the following text into the beacon-chain.service file.

```
[Unit]
Description=Ethereum 2 Beacon Chain
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=beacon
ExecStart=/home/beacon/bin/prysm.sh beacon-chain --config-file /home/beacon/prysm-beacon.yaml

[Install]
WantedBy=multi-user.target
Alias=beacon
```

#### Validator

```console
sudo nano /etc/systemd/system/validator.service
```

Copy and paste the following text into the validator.service file.

```
[Unit]
Description=Ethereum 2 Validator
Wants=beacon-chain.service
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=validator
ExecStart=/home/validator/bin/prysm.sh validator --config-file /home/validator/prysm-validator.yaml

[Install]
WantedBy=multi-user.target
```

### Create Prysm Configuration Files

#### prysm-beacon.yaml

```console
sudo -u beacon nano /home/beacon/prysm-beacon.yaml
```

Copy and paste the following text into the prysm-beacon.yaml configuration file.


```
datadir: "/home/beacon/prysm"
p2p-host-ip: "XXX.XXX.XXX.XXX"
http-web3provider: "http://YYY.YYY.YYY.YYY:8545"
monitoring-host: "0.0.0.0"
p2p-tcp-port: 13000
p2p-udp-port: 12000
accept-terms-of-use: true
```

 - If you have a dynamic IP address, remove the `p2p-host-ip` line.
   Otherwise, update `XXX.XXX.XXX.XXX` to your external IP address.
 - Update `YYY.YYY.YYY.YYY` to the IP address of your Eth1 node.
 - The `p2p-tcp-port` and `p2p-udp-port` lines are optional if you use the
   default values of 13000 and 12000, respectively.

Change permissions of the file.

```console
sudo -u beacon chmod 600 /home/beacon/prysm-beacon.yaml
```

#### prysm-validator.yaml

```console
sudo -u validator nano /home/validator/prysm-validator.yaml
```

Copy and paste the following text into the prysm-validator.yaml configuration file.

```
monitoring-host: "0.0.0.0"
graffiti: "YOUR_GRAFFITI_HERE"
beacon-rpc-provider: "127.0.0.1:4000"
wallet-password-file: "/home/validator/.eth2validators/wallet-password.txt"
accept-terms-of-use: true
```

- `graffiti` can be changed to whatever text you would prefer.

Change permissions of the file.

```console
sudo -u validator chmod 600 /home/validator/prysm-validator.yaml
```

### Make Validator Deposits and Install Keys

Follow the latest instructions at [launchpad.ethereum.org](https://launchpad.ethereum.org) or the correct launch pad for the network to which you will be connecting.

Look for the latest eth2.0-deposit-cli [here](https://github.com/ethereum/eth2.0-deposit-cli/releases/).

```console
cd
wget https://github.com/ethereum/eth2.0-deposit-cli/releases/download/v1.2.0/eth2deposit-cli-256ea21-linux-amd64.tar.gz
tar xzvf eth2deposit-cli-256ea21-linux-amd64.tar.gz
mv eth2deposit-cli-9310de0-linux-amd64 eth2deposit-cli
cd eth2deposit-cli
./deposit new-mnemonic --num_validators NUMBER_OF_VALIDATORS --chain mainnet
```

Change the `NUMBER_OF_VALIDATORS` to the number of validators you want to create. Follow the prompts and instructions.

**BACKUP YOUR MNEMONIC AND PASSWORD!**

The next step is to upload your deposit data file to the launchpad site. If you are using Debian 10.9 "buster", you can either open up the deposit data file and copy it to a file on your desktop computer with the same name, or you can use scp or an equivalent tool to copy the deposit data to your desktop computer.

Follow the instructions by dragging and dropping the deposit file into the launchpad site. Then continue to follow the instructions until your deposit transaction is successful.

```console
sudo -u validator /home/validator/bin/prysm.sh validator accounts import --keys-dir=$HOME/eth2deposit-cli/validator_keys
```

Follow the prompts. The default wallet directory should be `/home/validator/.eth2validators/prysm-wallet-v2`. Use the same password used when you were prompted for a password while running `./deposit new-mnemonic --num_validators NUMBER_OF_VALIDATORS --chain mainnet`.

Create a password file and make it readbable only to the validator account.

```console
sudo -u validator touch /home/validator/.eth2validators/wallet-password.txt && sudo chmod 600 /home/validator/.eth2validators/wallet-password.txt
```

Edit the file and put the password you entered into the `deposit` tool into the `wallet-password.txt` file.

```console
sudo nano /home/validator/.eth2validators/wallet-password.txt
```

Enter the password into the first line and save the file.


### Start Beacon Chain and Validator

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start beacon-chain validator
sudo systemctl enable beacon-chain validator
```

## Monitoring
The following will set up prometheus for collecting data, grafana for displaying dashboards, node_exporter for providing system data to prometheus, and blackbox_exporter for providing ping data to prometheus.

node_exporter and blackbox_exporter are optional, though some charts on the dashboard provided may need to be removed if those tools are not used. The prometheus configuration file may also need to be updated.

### Prometheus
#### Create User Account
```console
sudo adduser --system prometheus --group --no-create-home
```

#### Install Prometheus

Find the URL to the latest amd64 version of Prometheus at https://prometheus.io/download/. In the commands below, replace any references to the version 2.23.0 to the latest version available.

```console
cd
wget https://github.com/prometheus/prometheus/releases/download/v2.23.0/prometheus-2.23.0.linux-amd64.tar.gz
tar xzvf prometheus-2.23.0.linux-amd64.tar.gz
cd prometheus-2.23.0.linux-amd64
sudo cp promtool /usr/local/bin/
sudo cp prometheus /usr/local/bin/
sudo chown root:root /usr/local/bin/promtool /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/promtool /usr/local/bin/prometheus
cd
rm prometheus-2.23.0.linux-amd64.tar.gz
```

#### Configure Prometheus
```console
sudo mkdir -p /etc/prometheus/console_libraries /etc/prometheus/consoles /etc/prometheus/files_sd /etc/prometheus/rules /etc/prometheus/rules.d
```

Copy and paste the following text into the prometheus.yml configuration file:

```console
sudo nano /etc/prometheus/prometheus.yml
```

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:9090']
  - job_name: 'beacon node'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:8080']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:9100']
  - job_name: 'validator'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:8081']
  - job_name: 'ping_google'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 8.8.8.8
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
  - job_name: 'ping_cloudflare'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 1.1.1.1
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
  - job_name: json_exporter
    static_configs:
    - targets:
      - 127.0.0.1:7979
  - job_name: json
    metrics_path: /probe
    static_configs:
    - targets:
      - https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:7979
```

Change the ownership of the prometheus directory.

```console
sudo chown -R prometheus:prometheus /etc/prometheus
```

#### Data Directory
```console
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chmod 755 /var/lib/prometheus
```

#### Set Up systemd Service
```console
sudo nano /etc/systemd/system/prometheus.service
```

Copy and paste the following text into the prometheus.service file.
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start prometheus.service
sudo systemctl enable prometheus.service
```

### Grafana
```console
cd
sudo apt install -y apt-transport-https
sudo apt install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt install -y grafana-enterprise
```

#### Setup systemd

**Optional:** Edit the `grafana-server.service` file to add "grafana" as an alias to grafana server. I generally forget that the default name for this service is `grafana-server`.

```
sudo nano /lib/systemd/system/grafana-server.service
```

At the end of this file, in the `[Install]` section, add the following line:

```
Alias=grafana.service
```

Start the service.

```console
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Login to grafana at http://XXX.XXX.XXX.XXX:3000/, replacing `XXX.XXX.XXX.XXX` with the IP address of your server. If you do not know the IP address, run `ifconfig`.

Default username `admin`. Default password `admin`. Grafana will ask you to set a new password.

#### Setup Prometheus Data Source
1. On the left-hand menu, hover over the gear menu and click on Data Sources.
2. Then click on the Add Data Source button.
3. Hover over the Prometheus card on screen, then click on the Select button.
4. Enter `http://127.0.0.1:9090/` into the URL field, then click Save & Test.

#### Install Grafana Dashboard
1. Hover over the plus symbol icon in the left-hand menu, then click on Import.
2. Copy and paste the dashboard at [https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json](https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json) into the "Import via panel json" text box on the screen. If you used an older version of these instructions, where the Prometheus configuration file uses the beacon node job name of "beacon" instead of "beacon node", please (use this dashboard)[https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source.json] instead for backwards compatibility.
3. Then click the Load button.
4. Then click the Import button.

Note: At this point in the process, any widgets showing details from the validator will show "N/A", because the validator still has no keys configured. As soon as keys are configured for the validator, the validator details should begin to show up.

#### Final Grafana Dashboard Configuration
A few of the queries driving the Grafana dashboard may need different settings, depending on your hardware.

##### Network Traffic Configuration
To ensure that network traffic is correctly reflected on your Grafana dashboard, update the network interface in the Network Traffic widget. Run the following command to find your Linux network device.

```console
ifconfig
```

Output of the command should look like the following:
```
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.10  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::1e69:7aff:fe63:14b0  prefixlen 64  scopeid 0x20<link>
        ether 1c:69:7a:63:14:b0  txqueuelen 1000  (Ethernet)
        RX packets 238936  bytes 78487335 (78.4 MB)
        RX errors 0  dropped 1819  overruns 0  frame 0
        TX packets 257824  bytes 112513038 (112.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 16  memory 0x96300000-96320000

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 39805  bytes 29126770 (29.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39805  bytes 29126770 (29.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Of the two entries shows above, the first lists my IP address on the second line, network interface `eno1`. Find the entry that represents the network connection you want to monitor and copy the device name, which is the part before the colon on the first line of each entry. In my case the value is `eno1`.

1. Go to the Grafana dashboard previously installed
2. Find the Network Traffic widget, and open the drop down that can be found by the Network Traffic title.
3. Click Edit.
4. There will be four references to `eno1` in the queries that appear. Replace all four with the name of the network interface you found in the `ifconfig` command.

### node_exporter
#### Create User Account
```console
sudo adduser --system node_exporter --group --no-create-home
```

#### Install node_exporter
```console
cd
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xzvf node_exporter-1.0.1.linux-amd64.tar.gz
sudo cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm node_exporter-1.0.1.linux-amd64.tar.gz
```

#### Set Up System Service
```console
sudo nano /etc/systemd/system/node_exporter.service
```

Copy and paste the following text into the node_exporter.service file.

```
[Unit]
Description=Node Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start node_exporter.service
sudo systemctl enable node_exporter.service
```

### json_exporter

#### Install go
Install go, if you haven't already.

```console
sudo apt install -y golang-1.14-go

# Create a symlink from /usr/bin/go to the new go installation
sudo ln -s /usr/lib/go-1.14/bin/go /usr/bin/go
```

#### Create User Account
```console
sudo adduser --system json_exporter --group --no-create-home
```

#### Install json_exporter
```console
cd
git clone https://github.com/prometheus-community/json_exporter.git
cd json_exporter
make build
sudo cp json_exporter /usr/local/bin/
sudo chown json_exporter:json_exporter /usr/local/bin/json_exporter
```

#### Configure json_exporter

```console
sudo mkdir /etc/json_exporter
sudo chown json_exporter:json_exporter /etc/json_exporter
```

```console
sudo nano /etc/json_exporter/json_exporter.yml
```

Copy and paste the following text into the json_exporter.yml file. 

```
metrics:
- name: ethusd
  path: "{.ethereum.usd}"
  help: Ethereum (ETH) price in USD
```

Change ownership of the configuration file to the json_exporter account.

```console
sudo chown json_exporter:json_exporter /etc/json_exporter/json_exporter.yml
```

#### Set Up System Service
```console
sudo nano /etc/systemd/system/json_exporter.service
```

Copy and paste the following text into the node_exporter.service file.

```
[Unit]
Description=JSON Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=json_exporter
ExecStart=/usr/local/bin/json_exporter --config.file /etc/json_exporter/json_exporter.yml

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start json_exporter.service
sudo systemctl enable json_exporter.service
```


## Optional

### Install ntpd
For now, I prefer to use ntpd over the default systemd-timesyncd for syncing my system clock to an official time source.

From [this tutorial on setting up time syncing on debian.](https://vitux.com/how-to-setup-ntp-server-and-client-on-debian-10/)

> Though timesyncd is fine for most purposes, some applications that
> are very sensitive to even the slightest perturbations in time may be
> better served by ntpd, as it uses more sophisticated techniques to
> constantly and gradually keep the system time on track.

```console
sudo apt install -y ntp
```
Update the NTP pool time server configuration to those that are geographically close to you. See [http://support.ntp.org/bin/view/Servers/NTPPoolServers](http://support.ntp.org/bin/view/Servers/NTPPoolServers) to find servers near you.

```console
sudo nano /etc/ntp.conf
```
Look for lines that begin with `server` and replace the current values with the values you identified from ntp.org.

Restart ntp. This will automatically shut down systemd-timesyncd, the default debian time syncing solution.

```console
sudo systemctl restart ntp
```

enable ntp. 

```console
sudo systemctl enable ntp
```


### blackbox_exporter
I have used blackbox_exporter to provide [ping](https://en.wikipedia.org/wiki/Ping_(networking_utility)) time data between my staking system and two DNS providers. Data is sent to Prometheus and on to Grafana. I have not found a practical use for this yet, though I have seen some interesting short-term shifts in ping times to Google. Therefore, blackbox_exporter is optional. 

The Grafana dashboard in these instructions includes a panel with a ping time graph. If you choose not to install blackbox_exporter, simply remove that panel from your Grafana dashboard. It will not show data.

#### Create User Account
```console
sudo adduser --system blackbox_exporter --group --no-create-home
```

#### Install blackbox_exporter
```console
cd
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.18.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.18.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
```

Allow blackbox_exporter to ping servers.
```console
sudo setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

```console
rm blackbox_exporter-0.18.0.linux-amd64.tar.gz
```

#### Configure blackbox_exporter

```console
sudo mkdir /etc/blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
```

```console
sudo nano /etc/blackbox_exporter/blackbox.yml
```

Copy and paste the following text into the blackbox.yml file. 

```
modules:
        icmp:
                prober: icmp
                timeout: 10s
                icmp:
                        preferred_ip_protocol: ipv4
```

Change ownership of the configuration file to the blackbox_exporter account.

```console
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
```

#### Set Up System Service
`sudo nano /etc/systemd/system/blackbox_exporter.service`

Copy and paste the following text into the blackbox_exporter.service file.

```
[Unit]
Description=Blackbox Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=blackbox_exporter
ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter.service
sudo systemctl enable blackbox_exporter.service
```

## Router Configuration
You may need to configure your router to forward the following ports to your staking system. See your router documentation for details.

Prysm Beacon Chain: 12000/udp
Prysm Beacon Chain: 13000/tcp
geth: 30303/udp
geth: 30303/tcp


### Firewall
If your staking system is behind a router with a firewall, you may not want to add another level of firewall to your network security. This section may be skipped.

The following commands set up the minimal firewall rules necessary to run the Prysm beacon-chain and geth

```console
# beacon chain
sudo ufw allow 12000/udp
sudo ufw allow 13000/tcp

# geth
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp

# grafana
sudo ufw allow 3000/tcp
```

Run the following command to set up firewalls rules for SSH. If you changed your default SSH port above, change the `22` in this command to the port you are using.

```console
# ssh
sudo ufw allow 22/tcp
```

Set up default firewall rules and enable the firewall.

```console
# Defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

The following commands open up the remaining ports that are used by the software in this set of instructions. These ports are typically used only by other software internal to the staking system, and do not need to be opened on the firewall unless you would like direct access to some of the administrative/metrics pages, or if systems external to your staking system will be services on your staking system.

```console
# beacon chain
#   - This only needs to be enabled if external validators will be accessing this beacon chain.
sudo ufw allow 4000/tcp

# node_exporter
#   - This only needs to be enabled if you want to access node_exporter stats directly.
sudo ufw allow 9100/tcp

#geth
#   - This only needs to be enabled if external beacon chains will be accessing this geth full node.
sudo ufw allow 8545/tcp

# beacon-chain metrics
#   - This only needs to be enabled if you want to access beacon-chain stats directly.
sudo ufw allow 8080/tcp

# blackbox_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 9115/tcp

# prometheus
#   - This only needs to be enabled if you want to access prometheus directly.
sudo ufw allow 9090/tcp

# json_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 7979/tcp
```
## Common Commands
The following are some common commands you may want to use while running this setup.

### Service Statuses
To see the status of system services:

```console
sudo systemctl status beacon-chain
sudo systemctl status validator
sudo systemctl status geth
sudo systemctl status prometheus
sudo systemctl status grafana-server
sudo systemctl status node_exporter
sudo systemctl status blackbox_exporter
sudo systemctl status json_exporter
```

Or, to see the status of all at once:
```console
sudo systemctl status beacon-chain validator geth prometheus grafana-server node_exporter blackbox_exporter json_exporter
```
### Service Logs
To watch the logs in real time:

```console
sudo journalctl -u beacon-chain -f
sudo journalctl -u validator -f
sudo journalctl -u geth -f
sudo journalctl -u prometheus -f
sudo journalctl -u grafana-server -f
sudo journalctl -u node_exporter -f
sudo journalctl -u blackbox_exporter -f
sudo journalctl -u json_exporter -f
```
### Restarting Services
To restart a service:

```console
sudo systemctl restart beacon-chain
sudo systemctl restart validator
sudo systemctl restart geth
sudo systemctl restart prometheus
sudo systemctl restart grafana-server
sudo systemctl restart node_exporter
sudo systemctl restart blackbox_exporter
sudo systemctl restart json_exporter
```

### Stopping Services
Stopping a service is separate from disabling a service. Stopping a service stops the current execution of the server, but does not prohibit the service from starting again after a system reboot. If you intend for the service to stop running and to not restart after a reboot, you will want to stop and disable a service.

To stop a service:

```console
sudo systemctl stop beacon-chain
sudo systemctl stop validator
sudo systemctl stop geth
sudo systemctl stop prometheus
sudo systemctl stop grafana-server
sudo systemctl stop node_exporter
sudo systemctl stop blackbox_exporter
sudo systemctl stop json_exporter
```

**Important:** If you intend to stop the beacon chain and validator in order to run these services on a different system, stop the services using the instructions in this section, and disable these services following the instructions in the next section. You will be at risk of losing funds through slashing if you accidentally validate the same keys on two different systems, and failing to disable the services may result in your beacon chain and validator running again after a system reboot.

### Disabling Services
To disable a service so that it no longer starts automatically after a reboot:

```console
sudo systemctl disable beacon-chain
sudo systemctl disable validator
sudo systemctl disable geth
sudo systemctl disable prometheus
sudo systemctl disable grafana-server
sudo systemctl disable node_exporter
sudo systemctl disable blackbox_exporter
sudo systemctl disable json_exporter
```

### Enabling Services
To re-enable a service that has been disabled:

```console
sudo systemctl enable beacon-chain
sudo systemctl enable validator
sudo systemctl enable geth
sudo systemctl enable prometheus
sudo systemctl enable grafana-server
sudo systemctl enable node_exporter
sudo systemctl enable blackbox_exporter
sudo systemctl enable json_exporter
```
### Starting Services
Re-enabling a service will not necessarily start the service as well. To start a service that is stopped:

```console
sudo systemctl start beacon-chain
sudo systemctl start validator
sudo systemctl start geth
sudo systemctl start prometheus
sudo systemctl start grafana-server
sudo systemctl start node_exporter
sudo systemctl start blackbox_exporter
sudo systemctl start json_exporter
```

### Upgrading Prysm
Upgrading the Prysm beacon chain and validator clients is as easy as restarting the service when running the prysm.sh script as we are in these instructions. To upgrade to the latest release, simple restart the services. Use the commands above to check the log files of both the beacon chain and validator. If any important command line flags have changed, a notice should appear in the logs. Even better, read the release notes in advance of an upgrade.

```console
sudo systemctl restart beacon-chain
sudo systemctl restart validator
```

### Changing systemd Service Files
If you edit any of the systemd service files in `/etc/systemd/system` or another location, run the following command prior to restarting the affected service:

```console
sudo systemctl daemon-reload
```
Then restart the affected service:
```console
sudo systemctl restart SERVICE_NAME
```

- Replace SERVICE_NAME with the name of the service for which the service file was updated. For example, `sudo systemctl restart beacon-chain`.

### Updating Prysm Options
To update the configuration options of the beacon chain or validator, edit the Prysm configuration file located in the home directories for the services.

```console
sudo nano /home/validator/prysm-validator.yaml
sudo nano /home/beacon/prysm-beacon.yaml
```

Then restart the services:

```console
sudo systemctl restart validator
sudo systemctl restart beacon-chain
```

## Future Updates

There are at least one area where I may expand on my system configuration or instructions, but I have not pursued it yet.

 - SSH Key-Based Login
   - This seems to be a good security move, but it also seems to be the perfect way to get me locked out of my own system. I have never set this up before, but may look into it.


## Sources/Inspiration
Prysm: [https://docs.prylabs.network/docs/getting-started/](https://docs.prylabs.network/docs/getting-started/)

Go: [https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html](https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html)

Timezone: [https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/)

Account creation and systemd setup: [https://github.com/attestantio/ubuntu-server](https://github.com/attestantio/ubuntu-server)

blackbox_exporter: [https://github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter)

node_exporter: [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

Prometheus: [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)

Grafana: [https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

Dashboard: [https://github.com/metanull-operator/eth2-grafana](https://github.com/metanull-operator/eth2-grafana)

systemd: [https://www.freedesktop.org/software/systemd/man/systemd.unit.html](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

geth: [https://geth.ethereum.org/docs/install-and-build/installing-geth](https://geth.ethereum.org/docs/install-and-build/installing-geth)

sshd: [https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh](https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh)

ufw: [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04)

ufw: [https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)

ntpd: [https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04)
