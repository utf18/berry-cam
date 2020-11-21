# berry-cam

Simple raspberry pi zero camera and audio streaming with ffmpeg and rtsp

## Motivation

i was in need of a streaming solution for audio and video.
After a journey of searching a viable solution, i chose rtsp streaming, since i use the stream in my synology camp app.
Video and Audio should work without very high latency, so vlc streaming was out.
CPU is not considered a beast in the zero w devices, so i had to choose a solution without stressing the cpu 100% all the time.

Solution should be both easy and stable.
I found a very good tutorial here: 
https://codecalamity.com/raspberry-pi-hardware-accelerated-h264-webcam-security-camera/


## my Hardware:
- raspberry pi zero w
- rasperry pi cam noIR
- raspberry pi camera case
- some usb mic i had laying around

## overview

at the end the raspberry pi will output an rtsp stream to a local rtsp server with the help of ffmpeg, started via systemd

## getting started

### requirements
i assume installed raspbian (buster at the time of writing) and turned on ssh and camera in raspi-config.
gpu memory split is set to 160.
system is up to date: 

`sudo apt update && sudo apt upgrade -y`

### install

i assume we are in the home directory for the user *pi* so jump to that folder

`cd /home/pi`

install ffmpeg

`sudo apt install ffmpeg`


download the rtsp streaming server from the releases here: 
(raspberry pi zero means armv6, arch info can be found in /proc/cpuinfo)

`wget https://github.com/aler9/rtsp-simple-server/releases/download/v0.12.1/rtsp-simple-server_v0.12.1_linux_arm6.tar.gz`

extract the archive:

`tar -xvf rtsp-simple-server_v0.12.1_linux_arm6.tar.gz` 

now there should be 2 files in the current directory:

*rtsp-simple-server* and *rtsp-simple-server.yml*

first one is the binary, the other one is the config for it.

the command below needs to be inserted into the config file under the path section on par with *all*


```
cam:
  runOnInit: ffmpeg -nostdin -hide_banner -loglevel error -input_format h264 -f video4linux2 -s 1280x720 -r 20 -i /dev/video0 -f alsa -ac 1 -ar 44100 -i hw:1,0  -map 0:0 -map 1:0 -c:a aac -b:a 128k -c:v copy -r 20 -b:v 2M -f rtsp -rtsp_transport tcp rtsp://127.0.0.1:$RTSP_PORT/$RTSP_PATH
  runOnInitRestart: yes
```

this will run a ffmmpeg stream against the local port 8554. 
this port will be opended by the rtsp-simple-server, via systemd.

### systemd service

stream should start on every boot, so a systemd job it is.
```
cat > berry-cam.service <<'EOF'
[Unit]
Description=berry-cam
After=network.target rc-local.service
[Service]
Restart=always
RestartSec=20s
User=pi
ExecStart=/home/pi/rtsp-simple-server /home/pi/rtsp-simple-server.yml
[Install]
WantedBy=multi-user.target
EOF
```

copy the systemd service file to the system dir

`sudo cp berry-cam.service /etc/systemd/system/berry-cam.service`

enable it on boot, and start it now.

`sudo systemctl daemon-reload && sudo systemctl enable berry-cam && sudo systemctl start berry-cam`

status check via:

`sudo systemctl status berry-cam`


## View the rtsp stream

either via vlc network stream or in my case the synology surveilance station.
url is:

`rtsp://ip-of-your-raspberry-pi:8554/cam`

## wrap up

now on every boot the systemd job is starting the ffmpeg stream and the rtsp server. The Stream will start on demand, when the first client connects. CPU load is between 70-85% in my case.


### troubleshooting

- microphone is not found:

    check if device is hw 1,0 with `arecord -l` if not, change it in the ffmpeg command to the correct device
- microphone volume is low:

    can be set with alsamixer (change to correct device and press f4 and increase caputure volume)
- microphone is stereo

    set the -ac from 1 to 2 in the ffmpeg command
