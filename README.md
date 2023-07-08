# freeNAS-Jail-with-webcam
This is a howto for installing a webcam streamer on a freeNAS system where streaming happens inside a jail.
I use this to stream video to an octoprint instance.

## Architecture
Usually, everything should be installed and modified only inside a FreeNAS jail.
However, the base FreeNAS os has per default no webcam drivers.
The *root* modifications do not survive a reboot, but work for me.

root:
-----

This enables usage of pkg on root. Decide wheter to do this on your own.
```bash
sed 's/enabled: yes/enabled: no/' /usr/local/etc/pkg/repos/local.conf
sed 's/enabled: no/enabled: yes/' /usr/local/etc/pkg/repos/FreeBSD.conf
pkg update
```

Install webcam drivers:
```bash
pkg install webcamd
pkg install cuse4bsd-kmod
kldload cuse4bsd       #todo: somehow survive restart.

echo webcamd_0_flags="-N Image-Processor-USB-2-0-PC-Cam" >> rc.conf # or however your camera is called
echo webcamd_enable="YES" >> rc.conf
```

(maybe fiddle around with startup-script `/etc/local/rc.d/webcamd` to force user to root)

```bash
service webcamd start
ls /dev/video* #should return new devices...
```

jail:
-----

`pkg install ustreamer`

octoprint# cat printerctl.sh
```
#!/usr/local/bin/bash

if [ $# -ne 1 ]; then
        echo "usage: $0 <start|stop>"
        exit 1
fi
if [ "$1" = "start" ]; then
        mosquitto_pub -h 192.168.0.8 -t espRelais/switch -m 1
        /root/uhubctl/uhubctl -p 2 -a on        #camera
        /root/uhubctl/uhubctl -p 5 -a on        #printer
        /root/ustreamer.sh start
elif [ "$1" = "stop" ]; then
        /root/uhubctl/uhubctl -p 5 -a off       #printer
        mosquitto_pub -h 192.168.0.8 -t espRelais/switch -m 0
        /root/ustreamer.sh stop
fi
```

octoprint# cat ustreamer.sh
```
#!/usr/local/bin/bash
#ustreamer -s 0.0.0.0 --brightness-auto --quality 98 --desired-fps 15 --resolution 1280x1024

if [ $# -ne 1 ]; then
        echo "usage: $0 <start|stop>"
        exit 1
fi
if [ "$1" = "start" ]; then
        nohup ustreamer -s 0.0.0.0 --quality 98  --brightness-auto --desired-fps 15 > /var/log/ustreamer.log 2>&1 &
elif [ "$1" = "stop" ]; then
        killall ustreamer
fi
```
