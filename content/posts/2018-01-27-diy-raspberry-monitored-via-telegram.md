---
title: DIY raspberry monitored via telegram
author: Aurelien Ginolhac
date: '2018-01-27'
slug: diy-raspberry-monitored-via-telegram
categories: []
tags:
  - DIY
  - pi
output:
  blogdown::html_page:
    toc: true
    toc_depth: 1
    number_sections: true
---

## Rationale

The [raspberry pi](https://www.raspberrypi.org) has always been appealing to me, but I needed a project to 
really get involved. After discussing with [Eric Koncina](https://github.com/koncina) who made several
great applications with Pis, I decided to go for a home surveillance system.

The main objective was to see how often the neighbor cats are coming to our garden, because they are scaring our cat. It's not a big deal, rather a justification for the **pi** project.

## Materials

I bought a [Pi3 starter budget kit](https://www.kubii.fr/fr/kits-raspberry-pi/1992-starter-kit-raspberry-pi-3-budget-3272496008496.html) that contains:

- Pi3
- power, 5V, 2.5A
- case
- SD card 16 Go

Additionally, I purchased:

- [Pi camera NoIR]()

Was hoping to get some decent pictures / videos with low light. Turned out that IR leds are needed. That goes in the [TODO](#todo) section.

## Setting-up the pi

I won't go into details, I mostly followed the instructions in this [tutorial](https://howchoo.com/g/ndy1zte2yjn/how-to-set-up-wifi-on-your-raspberry-pi-without-ethernet). Briefly, here are the main steps

### download raspbian lite

Since I have no screen, no keyboard and the pi comes with a wifi controler, the **stretch lite** is sufficient. Image can be found at [raspberrypi.org](https://www.raspberrypi.org/downloads/raspbian/)


### format SD card

using disk utility, choose `MS-DOS FAT` file system

### install raspbian

Ensure your SD card is the second disk (`/dev/disk2`), otherwise **do** adapt to the correct one!

```
unzip 2017-11-29-raspbian-stretch-lite.zip
sudo dd bs=1m if=2017-11-29-raspbian-stretch-lite.img of=/dev/rdisk2
```

### enable ssh

Once copied, you can enable **ssh** by creating an empty file at the SD card root

```
cd /Volumes/boot/
touch ssh
```

### enable wifi

In order to connect to the pi without screen / keyboard, **wifi** needs to be configured right away. At the same location (`/Volumes/boot`) add a file named `wpa_supplicant.conf`

which contains:

```
network={
        ssid="your_network_ssid"
        psk="xxx"
        key_mgmt=WPA-PSK
}
```

Of note, I recently acquired a pi **zeroWH**, for which I had to add 3 lines ([SO question]()).

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=FR
network={
        ssid="your_network_ssid"
        psk="xxx"
        key_mgmt=WPA-PSK
}
```

### connect to pi

once the raspberrypi booted, try to find its IP

```
nmap -sn 192.168.1.0/24
```

which gives:

```
Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-08 22:44 CET
Nmap scan report for 192.168.1.27
Host is up (0.0071s latency).
Nmap scan report for 192.168.1.254
Host is up (0.0048s latency).
Nmap done: 256 IP addresses (2 hosts up) scanned in 11.71 seconds
```

`192.168.1.254` was the rooter, so pi was assigned `192.168.1.27`

`ssh pi@192.168.1.27` works.

you can also assign a fixed IP to your pi

### final configuration

once connected to the pi:

- run `sudo raspi-config` to activate the camera
- change password for the `pi` user
- set up `locales` and timezone
- update && upgrade raspbian strech
- add public `ssh` key to `.ssh/authorized_keys` for password less connection

## install the surveillance system

Even if, I'd like to have [openCV](https://github.com/opencv) like in this [tutorial](https://www.pyimagesearch.com/2017/09/04/raspbian-stretch-install-opencv-3-python-on-your-raspberry-pi/), it was way more work. Hence, the choice of [motion](https://motion-project.github.io)

### motion software

I followed the instructions provided in this great [tutorial](https://www.bouvet.no/bouvet-deler/utbrudd/building-a-motion-activated-security-camera-with-the-raspberry-pi-zero) by **Bouvet**. With some changes described below.


#### compilation

see from here https://motion-project.github.io/motion_build.html

```
wget https://github.com/Motion-Project/motion/releases/download/release-4.1.1/pi_stretch_motion_4.1.1-1_armhf.deb
sudo apt-get install gdebi-core
sudo gdebi pi_stretch_motion_4.1.1-1_armhf.deb
```

### run `motion`

first, as **Bouvet** suggested, I copied the main config file

`mkdir ~/motion && cp /etc/motion/motion.conf ~/motion/`

and alter the new copy.

running **motion**:

`motion -c ~/motion/motion.conf`


### tweaks to the initial tutorial

I choose to get the videos 

`gdiff /etc/motion/motion.conf ~/motion/motion.conf`

```
lighswitch 80
auto_brightness on
```

## Communication between motion _via_ telegram

Now comes the fun part. Receiving the motion detection by emails is fine, but it can be done via **Telegram** and the awesome API [`telepot`](http://telepot.readthedocs.io). **Eric** told me about telegram **bots** and it looked promising.
Actually, you can even _send_ commands to your pi using your phone using those telegram **bots**.

The useful feature I implemented are:

- **alerts**. A motion is detected. Send the best picture to your telegram account.
- **pause** / **resume** motion detection. Imagine you are away and for some reason (shadows, your own cat) you keep receiving alerts, you may want to remotely _pause_ the dectection. And of course, being able to _resume_ it. Those commands are already in `motion`, we just need to talk to it.
- **status**. You haven't received alerts, is the system running smoothly? You can ask for a confirmation that detection is on. Also, check if the camera is on.
- **snapshot**. No alerts, but you'd like to get a snapshot at any time.
- **video**. Maybe the nicest feature IMHO. Sending picture to your phone for every detection is fine, but not all videos. Based on the picture you see, you'd like to get the video of the detection. Once again, by a command to a telegram bot, you receive the last video recorded.


### create mybot

Eric gave me the link to this [tutorial](http://www.instructables.com/id/Set-up-Telegram-Bot-on-Raspberry-Pi/)

Of course, I am assuming you already have our own telegram account.

Talk to the `BotFather` and create `mybot`, you will receive a private token.

### install telepot

back on the pi, install `telepot` with `pip`, assuming you installed `python` and `pip`.

```
pip install telepot
```

### test sending message

```{python}
import telepot
bot = telepot.Bot('your-token')
bot.getMe()
```

returns `{u'username': u'mybot', u'first_name': u'cat tracker', u'is_bot': True, u'id': 00000008}`

to get your telegram id:

- send a messages from telegram to `mybot`
- fetch your message on the pi
```
from pprint import pprint
response = bot.getUpdates()
pprint(response)
```

your id appears, such as: **u'id': 00000004**

#### basic tests

- for text

`bot.sendMessage(00000004, 'Hey!')`

- for picture

`bot.sendPhoto(00000004, photo=open('/home/pi/motion/detected/07-2018-01-06_205746-13.jpg', 'rb'), caption='motion detected')`

- take snapshot on command

- in `bash`:
`curl -s http://localhost:8080/0/action/snapshot > /dev/null`
- in python:
`import requests; requests.get('http://localhost:8080/0/action/snapshot')`

- disable the join group with botfather

- Commands, after sending `/setcommands` to the `BotFather`

```
time - Returns current time on pi
check - Returns status of the camera
status - Returns status of motion
pause - Pauses the motion detection
resume - Resumes motion detection
snapshot - Returns current image
video - Returns last recorded video
```

### run at startup

in `/etc/rc.local`

add the following lines

```
# start listening to picatbot
/home/pi/motion/telegrambot.sh &
# start motion
motion -c /home/pi/motion/motion.conf
```

### TODO

- change the command line arg of `send*py` scripts with `argparse`
- remove pics/videos older than _xx_ days to save space
- run the 2 services as a user without sudo
- look into better settings for NoIR using [this thread](https://raspberrypi.stackexchange.com/questions/13818/auto-brightness-bypass-whilst-using-motion)



