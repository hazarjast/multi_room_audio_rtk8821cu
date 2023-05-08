# LMS / Squeezebox Multi-Room Audio Deployment using Fujitsu Futro S520n Thin Clients and Realtek 8821cu Wi-Fi Adapters Under Ubuntu Server 22.04.1
With the deprecation/death of inexpensive Wi-Fi audio adapters such as Chromecast Audio and Echo Link, I was searching for a cost effective means of synced, multi-room audio using my own speakers which could be controlled from my iOS devices. While I have some of my own media, mostly I stream music using Spotify so I needed something which ideally supported Spotify Connect integration as well.

My searches on this topic ultimately led me to Squeezebox Server / Logitech Media Server (LMS) which supports a wide variety of hardware and operating systems and checked the boxes I was looking for. The instructions in this repository cover an LMS multi-room audio deployment leveraging inexpensive x86_64 hardware and wi-fi adapters for the speaker clients. My local IT refurbisher was unloading a pallet of older Fujitsu thin clients which cost less than $12 USD each and I was able to find Realtek 8821cu based Wi-Fi 5 adapters which support both 2.4Ghz and 5Ghz on eBay for ~$5 USD. In my case the server is connected via Ethernet and only the clients use Wi-Fi, thus, a 3 room setup cost less than $65 USD. Basically, a poor man's Sonos ;)

## LMServer

1. Download and use Rufus to write Ubuntu 22.04.1 .iso to a USB thumbdrive and install.
2. Perform 'sudo apt update && sudo apt upgrade -y' and reboot.
3. Download .deb file from https://mysqueezebox.com/download and install with 'dpkg -i'.
4. Run initial LMS setup (http://lmserver:9000). Enable Spotty plugin. Go to settings of Spotty plugin to connect to Spotify.
5. Go to 'Advanced > File Types > Spotty' and set 'Ogg Vorbis' & 'FLAC' to 'Disabled'. Failing to do this leads to dropouts and erratic behavior with Spotify Connect in my testing.
6. Proceed with client setup steps below.


## LMClients

Compile driver for Realtek 8821cu based USB wifi adapter. Since we already need some dependent packages to compile, we can add the ALSA utilities and squeezelite player (LMS client) dependencies while we at it:
```
sudo apt update && sudo apt upgrade -y
sudo reboot now
sudo apt install -y build-essential dkms git iw rfkill wpasupplicant alsa alsa-utils squeezelite
[ 'apt install' will take ~5-10 minutes to complete package downloads and installations ]
git clone https://github.com/morrownr/8821cu-20210916.git
cd 8821cu-20210916
sudo ./install-driver.sh
[ 'install-driver.sh' will run for ~15-20 minutes as it has to build a lot of kernel driver .ko's ]
```
Driver compile process will run for ~15 minutes and will end asking if you want to update the options for for the adapter. Default options are fine so we select 'n' for 'no'. It will ask to reboot, select 'y' for yes.

Grep out the wifi device name and append it to the netplan yaml file for easy addition:
```
sudo -u root -i
select-editor
[select 'vi-basic', option '2']
cd /etc/netplan
cp 00-installer-config.yaml 00-installer-config.yaml.orig
ip a show |grep -o 'wl.*:' >> 00-installer-config.yaml
```

Finish adding wifi config which should look like the below, replacing or adding SSIDs where needed (and setting ethernet iface to 'optional: true'):
```
network:
  ethernets:
  ...
  version: 2  
  wifis:
    [wifi_iface]: 
      access-points: 
        [SSID]: 
         password: [password] 
      dhcp4: true
```

Apply netplan changes once finished:
```
netplan apply
```

Adjust ALSA default sound card to use the analog outputs instead of trying to use the non-existent HDMI audio outputs via the AMD integrated graphics (there's not even an HDMI output on these units). Also add our 'lmclient' default OS user to the 'audio' group for easy changes of levels with 'alsamixer' and change the power button behavior to perform graceful shutdowns:
```
cat <<EOF | sudo tee /etc/asound.conf  
defaults.pcm.card 1 
defaults.ctl.card 1
EOF
adduser [lmclient_user] audio
hostnamectl set-chassis vm
reboot now
```
To confirm which number sound card to use cat '/proc/asound/cards' which will list all that are available.

Once rebooted,  use 'alsamixer; to ensure levels for 'PCM', 'Line', and 'Master' are turned up and any 'auto-mute' feature is deactivated. From there we can use iPeng on iOS or iPadOS (cost: $8.99 USD) to control LMServer.

## Acknowledgements
Shout out to **morrownr** for his excellent Realtek 8821cu driver repository which is the cornerstone of this project. Also, thanks to all the great folks that continue to develop and support Logitech Media Server (Squeezebox Server) especially **michaelherger** who continues to make Spotify integration possible with LMS

