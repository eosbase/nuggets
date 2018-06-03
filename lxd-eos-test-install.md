# LXD EOS Install

Following page describes setting up a fresh Ubuntu Xenial container and installing a test EOS in.

Note that it's not secured or anything, just easy to work with, start over, etc.

Assuming LXC/LXD is already installed.

External resources:
* https://github.com/EOSIO/eos/wiki/Local-Environment
* https://developers.eos.io/
* https://www.youtube.com/watch?v=glB6UPHo1rA

## Prepare Ubuntu 16.04 LXD container:

Create the container
```sh
sudo lxc launch ubuntu:bionic eos
```

Enter a shell in the container
```sh
sudo lxc exec eos bash
```

Delete the ubuntu user and create an eos user
```sh
useradd -m -s /bin/bash eos
```

Add user eos to group sudo and allow it to sudo without password
```sh
usermod -a -G sudo eos
perl -pi -e 's/^%sudo.*$/%sudo ALL=(ALL:ALL) NOPASSWD:ALL/' /etc/sudoers
```

Add a public key for user eos
```sh
mkdir -p ~eos/.ssh
chmod 0700 ~eos/.ssh
cat >~eos/.ssh/authorized_keys2 <<EOF
ssh-rsa <your-public-key-here> eos
EOF
chmod 0600 ~eos/.ssh/authorized_keys2
chown -R eos:eos ~eos/.ssh
```

Fix networking
```sh
cat > /etc/netplan/50-cloud-init.yaml <<EOF
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: no
            addresses: [10.0.0.10/24]
            gateway4: 10.0.0.1
            nameservers:
                    addresses: [10.0.0.1]
EOF

sudo netplan apply

cat >/etc/hosts <<EOF
127.0.0.1 localhost
10.0.0.10 eos

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF
```

Reboot the container
```sh
reboot
```

## Install and start EOS.IO software and some tooling in the container

You can optionally remove the hostkey on your client with
```sh
ssh-keygen -R 10.0.0.10
```

Now shell to the container
```sh
ssh eos@10.0.0.10
```

Install prerequisites:
```sh
sudo apt install git ngrep jq
```

Clone the EOS software
```sh
git clone https://github.com/EOSIO/eos --recursive
```

Build it
```sh
cd eos
./eosio_build.sh
```
(enter yes when asked to install dependency packages)

Install it
```sh
cd build
sudo make install
```

Install some scripts for starting cleosd and keosd:
```sh
destdir=/usr/local/bin
for file in start-nodeos start-keosd eosgrep; do
  sudo wget -c -P $destdir https://raw.githubusercontent.com/eosbase/various/master/$file
  sudo chmod +x $destdir/$file
done
```

You can now start a keosd and nodeos instance. Above scripts will start them in a named screen which you can detach from with "Ctrl-A" followed by "d".
```sh
start-keosd
Ctrl-A d

start-nodeos
Ctrl-A d
```

You can list all active screens with:
```sh
screen -list
```

And re-attach to one of them with:
```sh
screen -r keosd
screen -r nodeos
```

Again, detach in the same manner.

## Create some accounts and setup basic contracts

If you open a 2nd shell to the container, then you can trace all traffic between cleos client and nodeos and keosd as follows
```sh
sudo eosgrep
```

While that is running perform some basic checks from the other shell
```sh
cleos get info
```

Get the init account
```sh
cleos get account eosio
```

Following couple of commands will create a ~/EOSCREDENTIALS file

Create a new default wallet
```sh
cleos wallet create | tee ~/EOSCREDENTIALS
```

Open and unlock the default wallet (below just reads the password without quotes from the 4th line of the just created file)
```sh
cleos wallet open
cleos wallet unlock --password $(sed -n '4 s/\"//gp' ~/EOSCREDENTIALS)
```

Create some accounts and store their keys in ~/EOSCREDENTIALS
```sh
accounts=bob alice
printf "\n%-12s %-51s %s\n\n" Account "Private key" "Public key" | tee -a ~/EOSCREDENTIALS
for account in bob alice; do
  keypair=$(cleos create key | tr '\n' ' ')
  private_key=$(echo $keypair | awk '{print $3}')
  public_key=$(echo $keypair | awk '{print $6}')
  printf "%-12s $private_key $public_key\n" $account | tee -a ~/EOSCREDENTIALS
  cleos create account eosio $account $pubkey $pubkey
done
```
