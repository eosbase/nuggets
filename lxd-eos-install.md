# LXD EOS Install

Following page describes setting up a fresh Ubuntu Xenial container for installing EOS in.

Note that it's not secured or anything, just easy to work with, start over, etc.

Assuming LXC/LXD is already installed.

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
sudo apt install git
```

Further instructions are taken from https://github.com/EOSIO/eos/wiki/Local-Environment

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
