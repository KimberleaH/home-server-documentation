**Note**
I beg your finest pardon. This documentation is a chaotic mess.
Everything——except for building the rig——is something I've not done before. I've documented my process while learning as I go. :)

I got the server put together 3/6/2026 and have been gradually getting things up and running since.

---

## Table of Contents
1. Setup Checklist
2. Server Specs
3. Referenced Material
4. Getting Started
5. Set Up RAID1 for Redundancy
6. Adjust fan settings
7. Installing and Configuring Docker
8. Create Containers in Docker
9. Add Node_Exporter And Configure Monitoring Stack
10. Interlude
11. Configure Pi Hole
12. Configure JellyFin
13. Configure NextCloud
14. To be continued

---

## Setup Checklist
- [x] Build PC
- [x] Install Ubuntu
- [x] Install mdadm
- [x] Set up RAID1
- [x] Fix fan speeds
- [x] Install Docker + Docker Compose
- [x] Install Portainer
- [x] Install Grafana, Prometheus, Node Exporter
	- [ ] Add alerts
	- [ ] Pull Grafana up on other devices
	- [ ] Grab metrics for fan speed and HDD temps
- [x] Pi Hole
  - [ ] Whitelist certain things so Pi Hole stops eating the pictures in my emails
- [x] Jellyfin
	- [ ] Add movies
	- [ ] Add shows
	- [ ] Connect to TV app
- [ ] NextCloud
- [ ] Wireguard + Cloudflare tunneling (try self-hosting a VPN)

---

## Server Specs
CPU: Intel Core i5-12600K
Memory: G.SKILL Ripjaws V Series 16GB (2 x 8GB) DDR4 3200
Mobo: MSI PRO B760M-P LGA 1700 mATX DDR4 (no onboard wifi/bluetooth)
Boot: Samsung 990 Evo Plus 1TB
RAID1: Seagate 4TB IronWolf 5400 rpm SATA 3 NAS HDD - two of these for a total of 4TB of storage
PSU: Corsair RM750x Gold Certified
CPU Cooler: Peerless Assassin 120mm
Case: Asus AP201 Mesh mATX
Misc: Cat 6 Ethernet cable, be quiet! 120mm case fans

**Thing to come back to:** I kinda...have my HDDs sitting on top of each other because the case's 3.5" drive placements were weird and conflicted with the rest of the components. The drive bay from my Cirus ATX build didn't quite fit, so I'll be getting a smaller drive bay for tidiness.

Note: While I built this with intent to run 24/7, there is no current need to. So the server is on only when someone is home later in the day, during the weekends, and the couple days I work from home.

---

## Referenced Material
- https://linuxvox.com/blog/ubuntu-raid-1-setup/
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04
- https://mattadam.com/2025/07/02/setting-up-docker-in-a-home-lab-your-simple-guide-to-getting-started/
- https://local.arvindgaba.com/2026/01/10/how-to-install-docker-on-ubuntu-linux-production-ready-setup/
- https://blog.nanitechtips.org/set-up-your-own-home-lab-with-docker-and-portainer/
- https://github.com/pi-hole/docker-pi-hole
- https://docs.pi-hole.net/docker/tips-and-tricks/
- https://jellyfin.org/docs/general/installation/container/
- https://github.com/nextcloud/docker

---

## Getting Started

Installed Ubuntu with the USB stick. Followed the documentation for Ubuntu and the guided installation.

Ubuntu is really quite clean! Boots up hella fast. Making me want to put my windows 10 machines on Ubuntu. My my.

Ctrl + Alt + T opens the terminal

sudo = admin privileges | if you don't include this at the beginning, the terminal may fuss about lack of permissions!

## Need to set up RAID1

https://linuxvox.com/blog/ubuntu-raid-1-setup/

This pulls up all of the disks and their partitions
```
sudo fdisk -l
```

Note down the disks to be used in the RAID 1 array
(e.g. `/dev/sdb` and `/dev/sdc`)

Available disks as of March 7 2026
- `/dev/nvme0n1` - this is the boot drive
- `/dev/sda` - this is 4TB HDD 1
- `/dev/sdb` - this is 4TB HDD 2

#### Create Partitions

Need to use the `parted` command to create partitions

```
sudo parted /dev/sda
sudo parted /dev/sdb
and so on
```

This will open the `parted` interactive shell. Follow these steps to create a partition:

1. Type `mklabel gpt` to create a new GPT partition table.
2. Type `mkpart primary ext4 0% 100%` to create a new primary partition that uses the entire disk.
3. Type `quit` to exit the `parted` shell.

Repeat these steps for the other disk

**Note:** GPT in this context stands for "GUID Partition Table". Remember how in WP CLI search-replaces you `--skip-columns=guid`? Aha!

#### Install mdadm

`mdadm` is a tool for managing RAID arrays on Linux systems.

```
sudo apt-get update
sudo apt-get install mdadm
```


#### Create RAID 1 array

```
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
```

In this command:

- `/dev/md0` is the name of the RAID 1 array that you are creating.
- `--level=1` specifies that you are creating a RAID 1 array.
- `--raid-devices=2` specifies that the RAID 1 array will consist of two disks.
- `/dev/sda1` and `/dev/sdb1` are the partitions that you created on the disks.

**btw you'll get a response like this:**
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: size set to [whatever]
Continue creating array? y

**Then you get this:**
mdadm: Defaulting to version 1.2 metadata
mdadm: array dev/md0 started.

#### Check the RAID 1 array status

You can check its status using the `mdadm --detail` command.

```
sudo mdadm --detail /dev/md0
```

This command will display detailed information about the RAID 1 array, including its status, the number of active disks, and the sync progress.

---
---
#### Issue encountered !!

I shut off the PC after the syncing finished around 10:30pm and came back the next morning to finish the raid1 setup. I did the above `sudo mdadm --detail /dev/md0` command and the terminal returned "no such file or directory".

After some looking around online, I used...

```
cat /proc/mdstat
```

...to check on the status of any RAID arrays and it pulled up the one I'd created. The array's name changed from `/dev/md0` to `/dev/md127` which is why the terminal was fussing. So I changed that `sudo mdadm` command to detail md127 and it pulled up everything as expected. :)

Also, it took the damn thing some 7 hours to sync. Gross.

---
---
#### Create a Filesystem

Once the RAID 1 array is created and synced, you need to create a filesystem on it. You can use the `mkfs` command to create a filesystem.

```
sudo mkfs.ext4 /dev/md127
```


#### Mount the RAID1 array

Finally, you need to mount the RAID 1 array to a directory on your system. You can create a new directory and mount the RAID 1 array to it.

```
sudo mkdir /mnt/raid1
sudo mount /dev/md127 /mnt/raid1
```

To ensure that the RAID 1 array is mounted automatically at boot time, you need to add an entry to the `/etc/fstab` file.

```
echo '/dev/md127 /mnt/raid1 ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

**Note:** The command on the left echoes out a statement (as seen in PHP) and the pipe | takes the output from the left and adds it to the provided file. `tee` reads output and writes it somewhere. `-a` means append or add (instead of overwriting existing content). `/etc/fstab` is the designated file.

You can check if the array was mounted successfully with this:

```
df -h
```

Mounted successfully!

---
---

I want to try renaming the directory from `/mnt/raid1` to something else.

It's apparently best practice to unmount the array, rename the directory, then remount the array.

```
sudo umount /mnt/raid1

OR

sudo umount /dev/md127
```

Then change the name of the directory. `/mnt/` or `/media/` are best practice naming conventions although not required. So... `/mnt/raid1` could change to `/raid1/archive` for example.

```
sudo mv /mnt/raid1 /mnt/archive
```

OR

```
sudo mkdir /raid1/archive
```

Then remount the array to whatever it is you want.

To remove the old directory (assuming you didn't just rename it), do this:
*Note that it only works if the directory is empty.*

```
sudo rmdir /mnt/raid1
```

Then you need to update the `/etc/fstab` file to reflect the change.

```
sudo nano /etc/fstab
```

Change `/dev/md127 /mnt/raid1 ext4 defaults 0 0` accordingly.

Save by Ctrl+X followed by Y and Enter

The outcome? I successfully created a new `raid1/archive` directory, removed `/mnt/raid1` (this didn't delete `mnt` btw) then mounted the array to `raid1/archive` and updated the `etc/fstab` file!

I'm getting this shit figured the fuck out! :)


## Adjust fan settings

I just went into the BIOS and adjusted the settings from DC to PWM and set up the smart fan curve how I like. I will likely swap out the rear exhaust fan with a PWM fan since it is DC while the others are not. Pretty sure I have a few downstairs I can pull from my stash of parts.

 Server is much quieter now. Rear exhaust fan is still going at full speed because it's DC. That's fine for the moment.

---

## Installing and Configuring Docker

Ubuntu Codename: Noble
Architecture: amd64


Docker - https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04
Docker compose - https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04

Docker for home lab - https://mattadam.com/2025/07/02/setting-up-docker-in-a-home-lab-your-simple-guide-to-getting-started/

First update existing stuff

```
sudo apt update
sudo apt install docker.io
sudo systemctl enable --now docker
```

Test if docker is running

```
docker run hello-world
```

I got a "Hello from Docker" message so hurray!

I also made a directory on /home/kimberlea-heili/docker-lab

And in docker-lab I created the following folders:
- Wireguard
- Pi-hole
- Plex
- Portainer

Note: I ended up adding Portainer (successfully, mind you) before I properly configured Docker lol and ran into issues later down the road before figuring out I'm a dummy and didn't everything I needed to first.
### Jumped the gun
#### Install Portainer

https://blog.nanitechtips.org/set-up-your-own-home-lab-with-docker-and-portainer/

In order to tell Docker where to put things--make sure containers go into the right folder--we need to install and mount "volumes" in the places we need them to go.

Here is the original command...

```
sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

I will be using this one instead to try mounting portainer in the directory I've already created.
```
sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /home/kimberlea-heili/docker-lab/portainer:/data portainer/portainer-ce
```

---
---

#### Issue encountered !!

I accidentally put `/home/user/docker-lab/portainer` which doesn't exist on my machine. Portainer was pulled down anyway.

I tried pulling it down again with the correct `/home/kimberlea-heili/docker-lab/portainer` but it fussed because "portainer" already existed. So I attempted to `sudo docker kill portainer` but that didn't necessarily work.

So then I did `sudo docker remove [long-ass container ID]` which seemed to work. When I then did `sudo docker stop portainer` it said there was no such file or directory. Same error when I did `sudo docker remove portainer`. Confirmation I got rid of portainer successfully.

Then I tried to pull down portainer with this:

```
sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v docker-lab/portainer:/data portainer/portainer-ce
```

But it fussed that that "includes invalid character for a local volume name." So now finally trying with the proper command again...

```
sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /home/kimberlea-heili/docker-lab/portainer:/data portainer/portainer-ce
```

And that fucking worked! Jeez.

---
---

Open up Portainer with the following: http://localhost:9000/

#### Install Docker Compose v2 (plugin)

Mind the difference between docker-compose (old) and docker compose (new)

Modern version of Docker include Compose as a plugin.

Install Docker Compose plugin
```
sudo apt install docker-compose-plugin -y
```

Verify installation
```
docker compose version
```

---
---

#### Issue encountered !!

Docker Compose didn't properly install. It said it couldn't find the package called docker-compose-plugin.

I checked around and read that I need to verify that the sources.list has the docker repo. So I went to 

```
sudo nano /etc/apt/sources.list
```

And it told me that ubuntu resources have been moved to `/etc/apt/sources.list.d/ubuntu.sources`. So I looked there and I didn't see the docker repo.

I also read that I must check if `/etc/apt/sources.list.d/docker.list` is a thing and I found it but it's empty. So I think we need to remove it first and try creating a new one.

So I did this:

```
sudo rm /etc/apt/sources.list.d/docker.list
```

And it turns out it doesn't exist. No I did not "find" the file like I thought. I was about to create such a file. I will get the hang of this eventually.

### Finish properly configuring docker

https://local.arvindgaba.com/2026/01/10/how-to-install-docker-on-ubuntu-linux-production-ready-setup/

Docker relies on a few foundational packages to securely fetch and verify software.

```
sudo apt install -y ca-certificates curl gnupg lsb-release
```

These are standard utilities.

#### Add official GPG key

We’ll install Docker **from the official source**, not Ubuntu’s default repo (which often lags behind).

```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

This ensures package authenticity and integrity.

I then checked to see if something was added correctly with `sudo nano /etc/apt/keyrings/docker.gpg` and there was a bunch of weird shit. So hey, something got added!

#### Add the docker repo

```
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \ https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Then refresh the package index:

```
sudo apt update
```

#### Install docker engine

Now install Docker and its core components:

```
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

At this point, **Docker** is installed and running on your Ubuntu system.

#### Verify Docker (again lol)

Run the classic test container:

```
sudo docker run hello-world
```

If you see a success message, Docker is working exactly as expected.

Note: If it asks if the daemon is running, like it did for me, do this:

```
sudo systemctl enable --now docker
```

#### Run docker without sudo command (recommended)

This makes it so you don't have to put "sudo" every time for docker CLI

```
sudo usermod -aG docker $USER
```

Log out and log back in (or reboot) for the change to take effect.

Then test:

```
docker ps
```

If it runs without errors, you’re good.

If not try

```
newgrp docker
```

That worked!

Obnoxiously, whenever I get logged out, it wants me to use sudo again. I don't know if I need to do `newgrp docker` every time. I'll look into this later because using sudo doesn't particularly bother me right now.

#### Enable Docker at Boot

For servers and long-running machines, Docker should start automatically.

```
sudo systemctl enable docker
sudo systemctl enable containerd
```

This ensures your containers survive reboots—**non-negotiable for hosting scenarios**.

At this point, Docker has been successfully installed and configured!!!!! WOOO! 🎉🎉🎉

---

## Create Containers in Docker

#### Prometheus Config File

Prometheus YAML config file
Located in /home/kimberlea-heili/docker-lab/prometheus
Titled: prometheus.yml

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```


#### Set Up Docker Compose File

This will include Grafana, Portainer, and Prometheus!

```
services:
  prometheus:
    image: prom/prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
      
  portainer:
    image: portainer/portainer-ce
    restart: always
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  prometheus_data:
  grafana_data:
  portainer_data:
```

**Note:** I originally had at the very top `version: '3.8'` but the terminal said that that is obsolete and should be removed so I removed it.

Then I ran

```
docker compose up -d
```

---
---

#### Troubleshooting

and got this nasty error:

```
Image grafana/grafana - pulled
Image prom/prometheus - pulled
Network docker-lab_default - Created
Volume docker-lab_portainer_data - Created
Volume docker-lab_prometheus_data - Created
Volume docker-lab_grafana_data - Created
Container docker-lab-portainer-1 - Starting
Container docker-lab-prometheus-1 - Starting
Container docker-lab-grafana-1 - Created

Error response from daemon: failed to create task for container: failed to create shim task: OCI run time create failed: runc create failed: unable to start container process: error during container init: error mounting "/home/user/docker-lab/prometheus.yml" to rootfs at "/etc/prometheus/prometheus.yml": mount src=/home/user/docker-lab/prometheus.yml, dst=/etc/prometheus/prometheus.yml, distFd=/proc/thread-self/fd//11, flas=MS_BIND|MS)REC: not a directory: Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type
```

Turns out there was a `/docker-lab/prometheus.yml` directory. What should have been a file was actually a folder. I think it was when I tried to move a file but accidentally created one perhaps. No idea. That or my docker-compose.yml file or prometheus.yml file was established incorrectly.

No it turns out my typos made it try to add a directory because it didn't see it in the space I'd declared it to be. The declaration was incorrect of course lol. I accidentally put `./prometheus.yml` in the docker-compose.yml when I should have put `./prometheus/prometheus.yml`. 

Then it fussed that port 9000 was already allocated. That's because I created that standalone portainer container earlier. After deleting the container, pulling down the docker-compose volumes, and starting them up again, that fixed the errors.

And now localhost:9000 pulls up portainer again. localhost:3000 pulls up grafana. And for some reason 9090 won't pull up prometheus because according to the terminal, no such file or directory exists. :( This is not unlike the issue I had with the Macbook where I absolutely couldn't get that shit to show up correctly so I dumped the damn file where it was looking. But I actually have this thing looking in the right place and simply doesn't see it.

The reason it didn't see it is because of another typo. I put in the docker-compose file `--config.file=etc...` instead of `--config.file=/etc...` so Docker couldn't properly find the file! Once I fixed that typo, everything started working as expected!

---

## Add Node_Exporter And Configure Monitoring Stack

Add this to the docker-compose.yml file to declare a node_exporter container. You can put it under "portainer."

```
node-exporter:
  image: prom/node-exporter:latest
  restart: always
  ports:
    - "9100:9100"
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc)($|/)"'
```

The terminal said `--collector.filesystem.ignored-mount-points` is deprecated, use `--collector.filesystem.mount-points-exclude`. Swapped that out.

Then put this in the prometheus.yml file under `scrape_configs`:

```
- job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

Use `node-exporter` (the container name), not `localhost`, since Prometheus will reach it via Docker networking.

#### Configure Grafana

I went to Connections > Data Sources and added Prometheus

Then went to Dashboard and started creating visualizations.

I found hwmon_temp and basically started fucking around with the types of displays. I went to Prometheus at some point and found something called hwmon_chip or something and that told me what the chip names are. There was nvme (my boot drive) and coretemp (cpu).

Then I went back in to Grafana and played around with adding visualizations and filters until I had a lovely dashboard. Then went diving for more.

CPU load, added rate and state (idle) to CPU seconds total (usage), memory usage, and disk I/O.

Thereafter, I started looking into other metrics I could try. Perhaps fan speed, HDD temperatures even.

---

## Interlude

I went into portainer and deleted the old hello-world images and containers. It's nice having this visualization. I don't know yet how much I'll use it but I wanted to have it onboard. I think I like this setup more than the Docker app itself. I will need to try installing Docker on my work machine again to properly pull up grafana and prometheus cuz right now I'm stuck on node_exporter not showing up in prometheus. It'll be exciting to try it and get it to work there too.

--

Alright I managed to get prometheus, grafana, and node exporter removed from my Mac. I had originally installed them with Homebrew and uninstalled them that same way. Then I downloaded Docker and used the same docker-compose.yml file as above along with the prometheus.yml file.

Docker said the ports were in use. For some reason those three programs were still on my machine. So I did a check with these:

```
ps aux | grep grafana
ps aux | grep prometheus
ps aux | grep node_exporter
```

They pulled up those services with their IDs and then I used this to get rid of the services:

```
kill <ID>
```

And that did it! Got it set up correctly on my Mac!

---

## Configure Pi Hole

Added this to the docker compose after adjusting it a bit. I pulled it from the official Pi-Hole GitHub page.

```
pihole:
	container_name: pihole
	image: pihole/pihole:latest
	restart: always
	ports:
	  # DNS Ports
	  - "53:53/tcp"
	  - "53:53/udp"
	  # Default HTTP Port
	  - "80:80/tcp"
	  # Default HTTPs Port. FTL will generate a self-signed certificate
	  - "443:443/tcp"
	environment:
	  TZ: 'US/Chicago'
	  FTLCONF_webserver_api_password: '<password>'
	  # If using Docker's default `bridge` network setting the dns listening mode should be set to 'ALL'
	  FTLCONF_dns_listeningMode: 'ALL'
	volumes:
	  - './etc-pihole:/etc/pihole'
	  # - SYS_NICE # Optional, if Pi-hole should get some more processing time
```

Then I did the docker compose down followed by up -d and docker fussed that port 53 tcp and udp are already in use. I looked up how to check if a port is in use and I ran this:

```
sudo lsof -i :53
```

There's apparently a service called `system-resolve` that is standard for Linux distributions to manage DNS stuffs. But I want Pi Hole instead since it's a DNS-level ad-blocker. SO! I'm gonna try to disable this and see if I break shit.

For starters, lets run this to check if system-resolve is running.

```
systemctl status systemd-resolved
```

I did that and I got a bunch of stuff but it started with this:

```
Loaded: loaded (usr/lib/systemd/system/systemd-resolved.service; enabled; preset: enabled)
Active: active (running)
```

So yes, it's running.

There's something called a symlink which I can find in `/etc/resolve.conf`. Lets do

```
cat /etc/resolve.conf
```

No such file or directory so I'm going to assume that systemd-resolve simply hasn't created that file. So I'll make one for Pi Hole.

Lets do this:

```
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

127.0.0.1 is apparently the IP address for localhost.


Alright. Doing `cat /etc/resolve.conf` now returns `nameserver 127.0.0.1` :)

And now I will stop and disable systemd-resolved that and should solve the "port 53 already in use" issue.

```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

`removed "/etc/systemd/system/sysinit.target.wants/systemd-resoved.service".`
`removed "/etc/systemd/system/dbus-order.freedesktop.resolve1.service".`

Cool.

docker compose ps > docker compose down > docker compose up -d

No errors

docker compose ps

Everything is UP!

The trouble now is that Firefox won't connect to the internet. Expected since we just turned off the DNS manager. So lets get Pi Hole configured to start managing the DNS now that systemd-resolved isn't enabled to do that.

btw `ip a` lets you see your device's IP address.

We put 127.0.0.1 in the symlink and that's what we'll use to pull up Pi Hole

http://127.0.0.1/admin and there it is!

I tried using "admin" as the password and that didn't work. I was sure I set the password somewhere. Pihole said that when installing it for the first time, a password would be shown to the user but that definitely didn't happen. I tried

```
docker exec -it pihole pihole setpassword
```

And it went on to say that the environment had a variable set for the password. Went to the docker-compose.yml and boom there it is. Thought I put a password somewhere. Forgot about that! And now I'm in! Now I need to make it so the internet works again.

I went back to the symlink resolv.conf thing and appended this:

```
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
```

And 8.8.8.8 is Google DNS and is a temporary fallback for if Pi Hole is down. Which it is.

I went to the pi hole dashboard > settings > DNS

Pick the servers to use. There are a few. I recognize OpenDNS and I really like Cloudflare. I will select those two.

Then I did the echo nameserver thing sudo tee (no -a) to replace the contents of the resolve.conf file with `nameserver 127.0.0.1`.

I don't know if this is necessary but I'm going to try restarting the networking with this:

```
sudo systemctl restart networking
```

This didn't do anything but throw an error saying it can't do it. So instead I took down docker compose and brought it back up. That updated the network.

EDIT: 127.0.0.1 is a loopback IP address, not the address for localhost.

I need to change my resolve.conf to have my machine's ip address. I tried to the following:

```
ip a
ip addr show
hostname -I
```

The first two showed me a bunch of junk I don't understand. Took a while to pick out IP addresses. But the third one there showed me the IP addresses without any of the other details which is nice.

Then I updated the resolve.conf with the 192.168.x.x address.

So trying different IPs didn’t work and then I tried adding the upstream IPs to the docker-compose file which didn’t work either, I decided to start over and completely uprooted pi hole and got rid of it and reenabled systemd-resolve. Internet back to working again.

Next day I dug around in the documentation some more and looked at my google wifi router info. It’s already set to DHCP.

https://docs.pi-hole.net/docker/tips-and-tricks/

Oh I also got rid of that /etc/resolve.conf file.

Found the systemd one instead in /etc/systemd/resolve.md and updated the DNS= to 1.1.1.1 and the Fallback_DNS= to 8.8.8.8

Made sure to docker compose down and restarted systemd with this thing:

```
sudo systemctl restart systemd-resolved
```

Internet still works so this is good.

I updated the docker compose file to this:

```
pihole:
	container_name: pihole
	image: pihole/pihole:latest
	restart: always
	ports:
	  # DNS Ports
	  - "53:53/tcp"
	  - "53:53/udp"
	  # Default HTTP Port
	  - "80:80/tcp"
	  # Default HTTPs Port. FTL will generate a self-signed certificate
	  - "443:443/tcp"
	environment:
	  TZ: 'US/Central'
	  FTLCONF_webserver_api_password: '<password>'
	  FTLCONF_dns_listeningMode: 'ALL'
	  PIHOLE_DNS: 1.1.1.1;1.0.0.1
	volumes:
	  - './etc-pihole:/etc/pihole'
	dns:
	  - 127.0.0.1
```

Went back to that .conf file for systemd and updated it to this

```
DNS=1.1.1.1 1.0.0.1
Fallback_DNS=8.8.8.8 8.8.4.4
DNSStubListener=no
```

I don’t want systemd using port 53 so turning off stub listener will prevent it from doing that.

Restarted systemd again. Internet still working.

Let’s do this to see if port 53 is being used.

```
sudo ss -tulpn | grep :53
```

I actually forgot to uncomment those lines so it appeared to not have worked lol. But after uncommenting and restarting, I got the proper result. `sudo lsof -i :53` didn’t pull up anything which is good.

Pihole didn’t fuss about port 53 being used! But it fussed that dns resolution is unavailable.

Changed the dns section in the docker from 127.0.0.1 to the two cloudflare dns addresses and that fixed it.

Pihole is running!
- ufw
- blocked ChatGPT hehe

Proton vpn gets in the way so I think I’ll just use proton out of the house and turn it off at home.

Look into cloudflare tunneling + wireguard.

---

## Configure JellyFin

I was originally gonna do Plex but after going through various opinions on Reddit and experiencing Plex, thanks to a friend sharing their account, I decided to try JellyFin.

The reasoning:
- People had mentioned that Plex requires internet to play the media from one's own local library, requires Plex Pass for transcoding, and Plex likes to recommend content not in one's library. Among a few other things.
- JellyFin offers transcoding for free, is open source, and minimalistic. It doesn't require internet and is limited to your own media library. All of which is perfect.

Note: I forgot to document as I went but I still remember what happened, so here we go.

https://jellyfin.org/docs/general/installation/container/

Grabbed the official docker compose image and trimmed it to this:

```
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    restart: 'unless-stopped'
    ports:
      - 8096:8096/tcp
      - 7359:7359/udp
    volumes:
      - /home/kimberlea-heili/docker-lab/jellyfin/config:/config
      - /home/kimberlea-heili/docker-lab/jellyfin/cache:/cache
      - type: bind
        source: /raid1/archive/jellyfin
        target: /jellyfin
        read_only: true
    extra_hosts:
      - 'host.docker.internal:host-gateway'
```

First of all I misunderstood what `target` was thinking that if `source: /path/to/media` was...well...the path to my media, then the target must be the media folder itself... for whatever reason.

Couldn't figure out why it kept saying /jellyfin/jellyfin didn't exist (it doesn't) and why the heck it was looking for /jellyfin/jellyfin at all. Well it turns out that--if I'm understanding this correctly--Jellyfin uses `/media` as the name for one's media directory for its own purposes. So you may have /folder/jellyfin/movies and /folder/jellyfin/shows and your source is /folder/jellyfin. JellyFin looks at what's inside your source, wraps it in /media and you can see all your folders under /media *within* JellyFin when you're setting it up. :)

I don't for sure understand *why* it does that, but my best guess is it's a readability thing. A standard for JellyFin to parse what it's looking at.

Then I also had another issue after understanding that. It kept saying that it didn't have permission to do what it needed to within /raid1/archive/jellyfin. The reason was because that directory tree belonged to the root and once I updated the director's owner, that solved the issue. I did that with this:

```
sudo chown -R $USER /raid1/archive
```

chown = change owner
-R = recursive so all the directories and their contents change too.

Using the above command made it so /archive and everything nested inside changed ownership to my user.

Thereafter, I pulled JellyFin up in localhost:8096, added some more folders and assigned those within JellyFin. I now have Movies, Shows, and Music. And no media in there yet. Soon I hope. I gotta get a CD drive so I can rip DVDs.

Ah. I added the JellyFin app to my TV's Roku home page and I'll get that connected to my server once I get a DVD or two ripped.

---

## Configure NextCloud

https://github.com/nextcloud/docker

I pulled the docker compose image (base version - apache) from the official source thusly:

```
    image: mariadb:lts
    restart: always
    command: --transaction-isolation=READ-COMMITTED
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_PASSWORD=
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  redis:
    image: redis:alpine
    restart: always

  app:
    image: nextcloud
    restart: always
    ports:
      - 8080:80
    depends_on:
      - redis
      - db
    volumes:
      - nextcloud:/var/www/html
    environment:
      - MYSQL_PASSWORD=
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db

volumes:
  nextcloud:
  db:
```

Note: I have not changed the restart policy from "always" to something else since that's how they came.

This so far is the least understandable service I have approached. I don't know what Redis is and I have had very little exposure to any form of SQL besides poking around phpMyAdmin for WordPress sites.

I was concerned about the `app` container showing port: 8080:80 because Pi Hole is using that I think but when I checked what was running on that port, I think nothing showed up if I remember.

In the end, I was able to get NextCloud to pull up in localhost but upon login it threw a SQL error I don't understand at all. I think it's probably a configuration error. I'm pretty darn stumped so I think I'll start over with this one and go from there.

---

## To Be Continued

I have plenty more I plan to do like wireguard, cloudflare tunneling, optimizing my dashboard on Grafana, add alerts, etc. I would like to figure out how to pull up my Grafana from my server onto my phone locally *without* signing up for a Grafana account. I've seen people do this and I must know!! I also need to get my movies compiled so my HDDs have something to do.
