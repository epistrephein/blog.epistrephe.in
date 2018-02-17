---
layout: post
title: "A WoW Vanilla server with MaNGOS"
description: "compile and install the World of Warcraft Vanilla server core on Linux"
date: 2016-10-08
comments: true
---

I used to play to World of Warcraft several years ago and I quit a few weeks before the release of The Burning Crusade. I came back a few times, to try out the new expansions, but nothing really was the same anymore after the death of the official Vanilla.

There are a lot a private server focused on Vanilla only, the most famous of which was the now closed Nostalrius. I tried some other servers, just for the sake of the memories, but I believe that the time to leave Azeroth forever is now come, and that it was good while it lasted.

However, just for the fun of it, I wanted to try the thrill of being in control of a server and being able to spawn, kill and cast whatever I wanted. So I dove in the WoW private server subculture and began to read about the various methods to do it.
Here's the result of that quest: a nice, kinda-easy, a little convoluted way to compile the [Continued MaNGOS](http://cmangos.net) core for **WoW Vanilla** on an **Ubuntu VPS**.

I choose a VPS because I thought it would be a nice challenge to add the extra level of exposing the realm on the internet: sometimes being just on `localhost` hides some troubles that we would've definitely faced if we were live. Moreover, there's already a perfect [Installation Guide](https://github.com/cmangos/issues/wiki/Installation-Instructions) from C-MaNGOS itself, so take this as a sort of journal of the installation process.

Since we're gonna compile from scratch, I spinned up a **quite capable VPS** at [DigitalOcean](https://www.digitalocean.com), specifically an Ubuntu 14.04.5, 8GB/4 CPUs with 80GB of disk. Launched and logged in as root, I began my journey with some basic configuration.

### Basic configuration
First of all I made sure the VPS was up to date

```bash
apt-get update; apt-get -y upgrade; apt-get -y dist-upgrade; apt-get autoremove; apt-get clean
```

and that some useful extra packages were installed

```bash
apt-get install -y htop tmux dnsutils
```

then I created a new user with sudo privileges, no password and ssh-key login.

```bash
NEWUSER=mangos
SSHKEY="ssh-rsa MY-REALLY-LONG-SSH-PUBLIC-KEY"

adduser --gecos "" --disabled-password $NEWUSER
gpasswd -a $NEWUSER sudo
echo "$NEWUSER ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
su $NEWUSER -c "cd; mkdir .ssh; chmod 700 .ssh; touch .ssh/authorized_keys; chmod 600 .ssh/authorized_keys"
echo "$SSHKEY" >> /home/$NEWUSER/.ssh/authorized_keys
```

Some basic security now: change the ssh port and disable root login and password authentication,

```bash
SSHPORT=2222

sed -i -e "/^Port/s/^.*$/Port $SSHPORT/" /etc/ssh/sshd_config
sed -i -e '/^PermitRootLogin/s/^.*$/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i -e '/^#PasswordAuthentication/s/^.*$/PasswordAuthentication no/' /etc/ssh/sshd_config
service ssh restart
```

install and enable `ufw` (allowing HTTP, HTTPS and SSH)

```bash
ALLOWEDPORTS=( 80 443 $SSHPORT )

hash ufw || apt-get install -y ufw
for p in "${ALLOWEDPORTS[@]}"; do ufw allow "$p"/tcp; done
echo y | ufw enable
```

and `fail2ban` with tripled ban time for keeping away bruteforce attacks.

```bash
hash fail2ban || apt-get install -y fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sed -i 's/bantime *= *600/bantime = 1800/g' /etc/fail2ban/jail.local
service fail2ban restart
```

I needed some quick tweaks to the timezone and time synchronization. My VPS was located in Frankfurt, so I used the Berlin timezone, but on Wikipedia you can find [the whole list of available timezones](http://en.wikipedia.org/wiki/List_of_tz_database_time_zones) (or reconfigure `tzdata` interactively)

```bash
TIMEZONE=Europe/Berlin

echo $TIMEZONE > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata
apt-get install -y ntp
```

Quite a good and safe configuration for a little toy. I rebooted here, for the kernel updates, and login back with the newly created user. The compile phase was about to start.

### Compiling C-MaNGOS
First of all I installed the big dependencies list, featuring the build tools, `mysql`, `git` and all the `boost` libs. During the installation process, I was prompted to choose a root password for MySQL.

```bash
sudo apt-get install -y build-essential gcc g++ automake git-core autoconf make patch libmysql++-dev mysql-server libtool libssl-dev grep binutils zlibc libc6 libbz2-dev cmake subversion libboost-all-dev
```

I cloned the specific git repositories for WoW Vanilla, that MaNGOS calls **Classic**.

```bash
cd ~
git clone git://github.com/cmangos/mangos-classic.git mangos
git clone git://github.com/classicdb/database.git classicdb
```

and created two new folders for the building phase.

```bash
mkdir build run
```

At this point, I was soon gonna need all the **graphical assets** from World of Warcraft. The official instructions suggest to use a Windows client to do the extraction from the client, but I wanted to be able to do everything on my VPS. So I compiled the MaNGOS core with the **extraction tools included**, while I was rsyncing the `World of Warcraft` folder from my workstation to the VPS in a folder called `wowdata`.

Keep in mind that for the MaNGOS core to work, you're gonna need specifically the `1.12.1` version. You can probably find it if you look around...

My user here (and after) is `mangos`, so if you're following make sure to use your user name.

```bash
cd ~/build
cmake /home/mangos/mangos -DCMAKE_INSTALL_PREFIX=/home/mangos/run -DBUILD_EXTRACTOR=1 -DBUILD_VMAP_EXTRACTOR=1 -DBUILD_MMAP_EXTRACTOR=1 -DPCH=0 -DDEBUG=0
```

Then, without caring for fine tuning of the make process, I just used all my threads at full power. This process is gonna take a while anyway. You should probably run this in a `tmux` window, so to be able to logout and come back after a while.

```bash
make -j$(nproc)
make install
```

Nice! At the end of the make, the `run` folder is now populated with the compiled bins (along with some other stuff).
The `conf` files for `mangos`, `realmd` and `ahbot` need to be manually moved here.

```bash
mv /home/mangos/run/etc/mangosd.conf.dist /home/mangos/run/etc/mangosd.conf
mv /home/mangos/run/etc/realmd.conf.dist /home/mangos/run/etc/realmd.conf
cp /home/mangos/mangos/src/game/AuctionHouseBot/ahbot.conf.dist.in /home/mangos/run/etc/ahbot.conf
```

Now comes the reeeeeeally long and frustrating part. The actual extraction. Once the tools are compiled and the client folder is on the VPS, I moved them to the WoW folder.

```bash
cd ~/wowdata

cp ~/build/contrib/extractor/ad .
cp ~/build/contrib/vmap_extractor/vmapextract/vmap_extractor .
cp ~/build/contrib/vmap_assembler/vmap_assembler .
cp ~/build/contrib/mmap/MoveMapGen .
cp ~/mangos/contrib/extractor_binary/MoveMapGen.sh .
cp ~/mangos/contrib/extractor_binary/offmesh.txt .
chmod +x ~/wowdata/MoveMapGen.sh
```

MaNGOS needs **4 folders** to be present in the `run/bin` directory: `dbc`, `maps`, `vmaps`, `mmaps`. The last two are not mandatory, but since I was doing this, I wanted to get to the bottom of it.

These 4 folders are generated by the tools we compiled from the `.mpq` files in the `Data` folder. The `mmaps` though, are assembled by calculating the actual tiles of every single map! With this amount of power, my VPS took 4 hours to complete this process at full blast. So be patient, launch everything in a tmux window and go eat a pizza.

```bash
./ad -f 0
./vmap_extractor -l

mkdir vmaps
./vmap_assembler Buildings vmaps

mkdir mmaps
./MoveMapGen.sh $(nproc)
```

### MySQL
The last step was to populate MySQL with the Warcraft data. All the necessary `sql` files are shipped with MaNGOS and I just needed to import them in the database.

```bash
mysql -uroot -p < ~/mangos/sql/create/db_create_mysql.sql
mysql -uroot -p mangos < ~/mangos/sql/base/mangos.sql
mysql -uroot -p characters < ~/mangos/sql/base/characters.sql
mysql -uroot -p realmd < ~/mangos/sql/base/realmd.sql
mysql -uroot -p mangos < ~/mangos/sql/scriptdev2/scriptdev2.sql
```

This will ask for the root password every time, but you can pass it directly after the `-p` flag if you wish.

There was a mandatory update to the `mangos` database that is not yet merged in the master `mangos.sql` and that I needed to do manually.

```bash
mysql -uroot -p mangos < ~/mangos/sql/updates/mangos/z2688_01_mangos_flametongue.sql
```

Once the databases are populated, I made it Vanilla-ready with the useful bash script provided.

```bash
cd ~/classicdb
./InstallFullDB.sh
```

Since the realm is not on `localhost`, if I try to connect with these settings I'll be stuck in a **realm loop** systematically trying to reach `127.0.0.1`. So I needed a few changes.

First of all, I opened the default server ports on UFW

```bash
sudo ufw allow 3724/tcp
sudo ufw allow 8085/tcp
echo y | sudo ufw enable
```

then used a MySQL query to specify the real IP of the realm, that I got with a quick `dig` query to opendns.

```bash
dig +short myip.opendns.com @resolver1.opendns.com
mysql -uroot -p
```

```sql
update `realmd`.`realmlist` set `address` = 'MYREMOTEIP' where `id` = '1';
```

I can also rename the realm

```sql
update `realmd`.`realmlist` set `name` = 'My Cool Realm Name' where `id` = '1';
```

before restarting `mysql`

```bash
sudo service mysql restart
```

### Starting MaNGOS

It's time to start MaNGOS. I opened two `tmux` windows, one for `mangosd` and the other for `realmd`, and called the bins

```bash
cd ~/run/bin
~/run/bin/mangosd -c ~/run/etc/mangosd.conf -a ~/run/etc/ahbot.conf
```

```bash
cd ~/run/bin
~/run/bin/realmd -c ~/run/etc/realmd.conf
```

Now inside the `mangosd` super-verbose window I can pass the commands to create a new account, set it to Vanilla and promote it to admin.
Just keep writing, or paste, the commands even if they get cut by the text. It will work anyway.

```bash
account create myusername mypassword
account set addon myusername 0
account set gmlevel myusername 3
```

Back in my workstation, I needed to change the `realmlist.wtf` file inside the WoW folder to the IP of my VPS.
Everything went smooth, and I was actually able to login, create a new character and enter Azeroth.

Since I had GM power, I immediately issued a few commands to get to level 60, learn all my spells and explore the whole map.

```
.levelup 59
.learn all_myspells
.explorecheat 1
```

Here is the list of [all the available GM commands](https://www.reaper-x.com/2007/09/21/wow-mangos-gm-game-master-commands/).

So, to mess around, I morphed into a very tiny Ragnaros and became super fast

```
.modify morph 11121
.modify scale 0.4
.modify speed 4
```

and went to explore my huge deserted world!

