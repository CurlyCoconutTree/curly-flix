# Curly-Flix Setup Guide
Inspiration for this project came from YouTuber Grafick (https://www.youtube.com/@GrafickStudios) and this specific video: https://www.youtube.com/watch?v=6pRHlH2eNBw

I would be remiss if I didn't mention Joshua Riek: https://github.com/Joshua-Riek/ubuntu-rockchip

This guide will help you set up Jellyfin on a Radxa Zero 3E using a custom Rockchip Ubuntu image in order to get hardware transcoding support. The setup includes Docker installation, USB drive configuration for swap and cache, and setting up various directories and services.

PLEASE NOTE:
I'm running with 2 ethernet connections, one (via USB C to USB/Eth adapter) is assigned a static IP and plugged directly into one of the ethernet ports on my NAS and the other is the eth0/end1 device on the Radxa board, which I connect to my main network.  I'm also using USB drives for cache, logs and swap so I don't thrash the SD card.  There's plenty of USB drives, lying around my office, so I'm not very concerned if I wear out a drive or two.

## Prerequisites
- 27W or 30W USB C power supply
- 1 x Radxa Zero 3E (4 gigs of RAM)
- 1 x 256 gig high speed SD card
- https://www.amazon.com/gp/product/B0871ZL9TG/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1
- 2 x 64 gig USB 3.1 thumb drives
https://joshua-riek.github.io/ubuntu-rockchip-download/boards/radxa-zero3.html
- Grab the file: ubuntu-24.04-preinstalled-server-arm64-radxa-zero3.img.xz
   - I used BalenaEtcher to burn it to the SD card
- The Debian Distro for the Radxa Zero 3E has the CPU clocked to 1.4Ghz.  The above Ubuntu image has it clocked at 1.8Ghz, which should be fine but I'd still suggest both heat sinks and a fan


### 3D Printed Case
https://www.printables.com/model/922305-radxa-zero3e-case-by-nenter
- Note: I had to scale the model vertically to give me enough space to close the lid without pressing into the SD reader on the board
- Note 2: Print the one with the vents on the sides and hole in the bottom for SD card access and print the lid with the hold for the fan

## Initial Setup

You can SSH into the instance, default User/Pass as of 8/23/2024 is ubuntu/ubuntu 

### 1. Create Directories
```bash
sudo apt-get update
sudo mkdir -p /media/movies
sudo mkdir -p /media/music
sudo mkdir -p /media/tv
sudo mkdir -p /media/music_videos
sudo mkdir -p /cache/logs
sudo mkdir -p /home/apps/docker/jellyfin/config
sudo mkdir -p /home/apps/docker/portainer/data
sudo chown -R ubuntu:ubuntu /cache
sudo chown -R ubuntu:ubuntu /home/apps/docker/jellyfin/config
```

### 2. Configure USB Ethernet Adapter
- **Find USB Ethernet device** (not `eth0/end1`) and set it to `192.168.20.4/24` (the port on my NAS is statically assigned to 192.168.20.2/24 so they can "talk")
   - `ip address show`
   - Mine is `enx00e04c680f35`, yours may be different

#### **Edit the Network Configuration**
`sudo nano /etc/netplan/50-cloud-init.yaml`

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  version: 2
  ethernets:
    enx00e04c680f35:
      dhcp4: no
      addresses:
        - 192.168.20.4/24
      gateway4: 192.168.20.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
    zz-all-en:
        dhcp4: true
        match:
            name: en*
        optional: true
    zz-all-eth:
        dhcp4: true
        match:
            name: eth*
        optional: true
```

### 3. Set Default Route
```bash
sudo ip route add default via 192.168.50.1 dev eth0
# OR dpending on what `ip address show` displays
sudo ip route add default via 192.168.50.1 dev end1
```

### 4. Create rc.local 
`sudo nano /etc/rc.local`

#### **Add the content below**
**NOTE: Your `sudo ip route del default via 192.168.20.2 dev enx00e04c680f35` may be different, just change the IP and/or device name at the end as needed**
```bash
#!/bin/bash
sudo ip route del default via 192.168.20.2 dev enx00e04c680f35
sudo ufw allow 5353/udp
sudo ufw allow 61208/udp
sudo ufw allow 61208/tcp
sudo ufw allow 61209/udp
sudo ufw allow 61209/tcp
```
#### **Execute**
`sudo chmod +x /etc/rc.local`

### 5. Configure ZRAM
```
sudo apt-get install zram-tools -y
sudo nano /etc/default/zramswap
```

#### **Make the necessary changes to match the file below**
```ini
# Compression algorithm selection
# speed: lz4 > zstd > lzo
# compression: zstd > lzo > lz4
# This is not inclusive of all that is available in latest kernels
# See /sys/block/zram0/comp_algorithm (when zram module is loaded) to see
# what is currently set and available for your kernel[1]
# [1]  https://github.com/torvalds/linux/blob/master/Documentation/blockdev/zram.txt#L86
ALGO=lzo

# Specifies the amount of RAM that should be used for zram
# based on a percentage the total amount of available memory
# This takes precedence and overrides SIZE below
#PERCENT=50

# Specifies a static amount of RAM that should be used for
# the ZRAM devices, this is in MiB
SIZE=1536

# Specifies the priority for the swap devices, see swapon(2)
# for more details. Higher number = higher priority
# This should probably be higher than hdd/ssd swaps.
PRIORITY=100
```

### 6. Configure USB Drive for Cache and Swap

#### **Find and Set USB Drive for Cache**
1. Identify the USB drive using `sudo fdisk -l`.
2. Format the drive (if necessary) and mount it:
```bash
sudo mkfs.ext4 /dev/sdb1
sudo mount /dev/sdb1 /cache
```

3. Update `/etc/fstab` to mount it automatically at boot:
```bash
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=$UUID /cache ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

#### **Configure USB Drive as Swap**
1. Create a swap partition:
```bash
sudo mkswap /dev/sda1
```

2. Enable the swap and add the swap to `/etc/fstab`:
```bash
UUID=$(blkid -s UUID -o value /dev/sda1)
echo "UUID=$UUID none swap sw 0 0" | sudo tee -a /etc/fstab
sudo swapon /dev/sda1
```

### 7. Configure Hostname and Hosts
Edit the following files:

#### `/etc/hostname`
```bash
curly-flix
```
#### `/etc/hosts`
```bash
127.0.0.1 localhost curly-flix
127.0.1.1 curly-flix

# The following lines are desirable for IPv6 capable hosts
#::1     localhost ip6-localhost ip6-loopback
#fe00::0 ip6-localnet
#ff00::0 ip6-mcastprefix
#ff02::1 ip6-allnodes
#ff02::2 ip6-allrouters
```

### 8. Reboot
```bash
sudo reboot
```

### 9. Install Utilities
```bash
sudo apt-get install neofetch -y
sudo apt-get install python3 -y
sudo apt install bpytop -y
sudo apt install zsh -y
sudo apt-get install docker.io -y
sudo apt-get install avahi-daemon -y
```

### 10. Configure network drives
Add the network mounts to `/etc/fstab`:
```bash
//192.168.20.2/YOUR_SHARE/movies /media/movies cifs username=YOUR_USERNAME,password=YOUR_PASSWORD,iocharset=utf8,vers=2.0 0 0
//192.168.20.2/YOUR_SHARE/music /media/music cifs username=YOUR_USERNAME,password=YOUR_PASSWORD,iocharset=utf8,vers=2.0 0 0
//192.168.20.2/YOUR_SHARE/tv /media/tv cifs username=YOUR_USERNAME,password=YOUR_PASSWORD,iocharset=utf8,vers=2.0 0 0
//192.168.20.2/YOUR_SHARE/music_videos /media/music_videos cifs username=YOUR_USERNAME,password=YOUR_PASSWORD,iocharset=utf8,vers=2.0 0 0
```

### 11. Configure Avahi
```bash
sudo systemctl start avahi-daemon
sudo systemctl enable avahi-daemon
```

### 12. Reboot
```bash
sudo reboot
```

## Docker Services Configuration

### 1. Run Jellyfin with Docker and GPU Hardware Acceleration
```bash
#!/bin/bash

# Capture the output of the loop in a variable
DEVICE_ARGS=$(for dev in dri dma_heap mali0 rga mpp_service \
   iep mpp-service vpu_service vpu-service \
   hevc_service hevc-service rkvdec rkvenc vepu h265e ; do \
  [ -e "/dev/$dev" ] && echo -n " --device /dev/$dev"; \
 done)

# Run the Docker container with the generated device arguments
sudo docker run -d \
  --name=jellyfin \
  --network=host \
  --privileged \
  --restart=unless-stopped \
  --user=1000:1000 \
  $DEVICE_ARGS \
  -v /media/movies:/media/Movies \
  -v /media/music:/media/Music \
  -v /media/tv:/media/TV \
  -v /media/music_videos:/media/Music\ Videos \
  -v /cache:/cache \
  -v /cache/logs:/config/log \
  -v /home/apps/docker/jellyfin/config:/config \
  jellyfin/jellyfin
```

### 2. Set Up Additional Services
- **Watchtower**: Automatically updates Docker containers and cleans up old volumes.
- **Glances**: Real-time system monitoring.
- **Portainer**: Docker container management.

```bash
sudo docker run -d \
   --name watchtower \
   -v /var/run/docker.sock:/var/run/docker.sock \
   --cleanup \
   --restart=always \
   containrrr/watchtower

sudo docker run -d \
   --name=glances \
   -p 61208-61209:61208-61209 \
   -v /var/run/docker.sock:/var/run/docker.sock:ro \
   --pid host \
   --restart="always" \
   -e GLANCES_OPT="-w" \
   nicolargo/glances:latest

sudo docker run -d \
   --name=portainer \
   -p 9000:9000 \
   -p 8000:8000 \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -v /home/apps/docker/portainer/data:/data \
   --restart=always \
   portainer/portainer-ce 
```

## Final Steps

1. Reboot the system to ensure all configurations take effect.
2. Verify that Jellyfin and other Docker containers are running properly.

### Local Websites to hit the Docker containers:
1. (Portainer) http://curly-flix:9000/
2. (Jellyfin) http://curly-flix:8096/
3. (Glances) http://curly-flix:61208/

## Troubleshooting:

1. First check ip route.  Did your changes stick?
2. Check your firewall rules.  Did those changes stick?
3. Ping NAS and ping Gateway, look at your network configuration.
4. Restart system
5. Check 1-3 again
6. Restart all containers

## Not Transcoding?
1. Do you have VAAPI selected for Transcoding in Jellyfin?

Note: If you don't sign into portainer soon enough after running it, you'll get locked out.  Delete the container and any volumes, recreate the host mounts and re-run the docker command.

Note 2: If you have a lot of media files, Jellyfin will take 24-48 hours to pull and extract metadata, this is extremely taxing on the Radxa, you'll still be able to play files as your media library builds... but I'd suggest being patient.  After that you should be able to run multiple feeds at a time without the Radxa getting above skin temperatures.
