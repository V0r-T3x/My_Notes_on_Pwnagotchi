-Original post for the solution:  
https://github.com/evilsocket/pwnagotchi/issues/1046  
-Original post for the LCD installation  
https://www.reddit.com/r/pwnagotchi/comments/ilqwyd/how_to_get_an_lcd_screen_working_on_a_pwnagotchi/  
-Drivers installation guide  
https://github.com/juj/fbcp-ili9341  
-Pisugar2 installation guide  
https://github.com/PiSugar/pisugar-power-manager-rs  
  
-Link for the base img for the pwnagotchi on the rpi02w  
https://anonfiles.com/9fE0e0M1xe/pwnagotchi-raspberry-pi-os-lite-master-05032022_zip  
  
-Configure the `/boot/config.txt`  
```
[all]
dtoverlay=dwc2
dtoverlay=spi1-3cs
dtparam=spi=on
dtparam=i2c_arm=on
dtparam=i2c1=on
gpu_mem=16
dtoverlay=dpi24
display_rotate=0
enable_dpi_lcd=1
display_default_lcd=1
dpi_group=2
dpi_mode=87
dpi_output_format=0x6016
hdmi_timings=240 1 38 10 20 240 1 38 10 20 0 0 0 60 0 6400000 1
framebuffer_width=240
framebuffer_height=240

hdmi_group=2
hdmi_mode=87
hdmi_cvt=320 240 60 1 0 0 0
hdmi_force_hotplug=1

#joystick click simulate enter key
dtoverlay=gpio-key,gpio=13,keycode=28
```

-Booting on the rpi02w  
-Expand img and enable auto login:  
```
sudo raspi-config  
```
-6 Advanced options > A1 Expand Filesystem > OK  
-1 System options > S5 Boot / Auto Login > B2 Console Autologin  
- Change date, time and timezone:  
```
sudo dpkg-reconfigure tzdata  
sudo date --set ****-**-**  
sudo date --set **:**:**  
```
-Create a file in /etc/profile.d/ called pwn.sh  
```
sudo nano /etc/profile.d/pwn.sh  
```
-Add this:  
```
sudo bash /home/pi/go-photo.sh &
sudo bash /home/pi/go-fbi.sh
```
-In /home/pi/ create a file called go-photo.sh  
```
sudo nano /home/pi/go-photo.sh
```
-Add this:  
```
#!/bin/bash
# go-photo.sh script (run in background, then launch go-fbi.sh)
# It copies the next photo into the free buffer and then updates the display alias
# source photos are held in the /photos/ folder (fetching and resizing is performed by the fetch-resize script)
# both buffers1&2.jpg and display.jpg are held in the /ram-disk/ folder (actual display is performed by the fbi script)
showtime=4 #photo show time -1 (so 4 = 5 seconds total)
cfile="0" # current file name
ctop="0" # current top by date
nbuf="buffer1.png" # next (free) buffer
# loop forever
while [ true ]; do
        top="0" # no top found
        next="0" # no next found
        # Scan the photos dir for the current file
        for filename in $(ls -A1c /home/pi/*.png); do
        # note, when you ls a different folder, the path is added to each file name (so file is /photos/name.jpg )
        if [ $top = "0" ]; then
                top=$filename
                if [ $filename != $ctop ]; then next=$filename; fi
        fi
        # find the next file AFTER the current
        if [ $next = "0" ]; then
                if [ $cfile = "0" ]; then next=$filename; fi
                if [ $filename = $cfile ]; then cfile="0"; fi
        fi
        done
        # if no next found, use top
        if [ $next = "0" ]; then next=$top; fi
        ctop=$top
        # OK, we have the next file, pause in case fbi is still reading the free buffer
        sleep 1
        cp -f $next /home/pi/ram-disk/$nbuf
        cfile=$next
        if [ $nbuf = "buffer1" ]; then
                nbuf="buffer2.png"
        else nbuf="buffer1.png"
        fi
        # now switch the link on first pass should creates display.jpg)
        ln -s -f /home/pi/ram-disk/$nbuf /home/pi/ram-disk/display.png
        # wait for the rest of the show time
        sleep $showtime
        # now go find the next
done
```
in /home/pi/ create a file called go-fbi.sh w  
```
sudo nano /home/pi/go-fbi.sh
```
-Add this:  
```
#!/bin/bash
# go-fbi script (launch last)
# (the go-photo (background) script will update 'display.jpg' to point at buffer1 or buffer2 to step the display
#  and will have already copied the first image into buffer1.jpg and alias linked it to display.jpg)
# this script still needs to generate the 3 fbi files and set the display alias for each
# (-f means overwrite any existing link of the same name)
ln -s -f /var/tmp/pwnagotchi/pwnagotchi.png alias-1.png
ln -s -f /var/tmp/pwnagotchi/pwnagotchi.png alias-2.png
ln -s -f /var/tmp/pwnagotchi/pwnagotchi.png alias-3.png
# launch fbi on a 3 second cycle
fbi -T 1 -d /dev/fb0 -a -noverbose -t 3 -cachemem 0 alias-1.png alias-2.png alias-3.png
```
-Also make a folder for the temp files to go  
```
mkdir /home/pi/ram-disk
```
-configure the /etc/pwnagotchi/config.toml  
```
sudo nano /etc/pwnagotchi/config.toml
```
-Edit this:  
```
ui.display.enabled = true
ui.display.type = "inky"
ui.display.color = "black"
ui.display.rotation = 180
```
- Install librairies for the BCM2835   
```
wget http://www.airspayce.com/mikem/bcm2835/bcm2835-1.68.tar.gz  
tar zxvf bcm2835-1.68.tar.gz  
cd bcm2835-1.68/  
./configure && sudo make && sudo make check && sudo make install  
```
- Install font  
```
sudo apt-get install ttf-wqy-zenhei  
```
- Install pip3  
```
sudo apt-get install python3-pip  
```
- Install RPi.GPIO  
``` 
sudo pip3 install RPi.GPIO  
```
- Install cmake  
```
sudo apt-get install cmake -y  
```
- Install 7zip  
```
sudo apt-get install p7zip-full  
```
- Move to the root folder  
```
sudo su  
cd ~  
```
- Install FBCP driver  
```
cd ~
git clone https://github.com/juj/fbcp-ili9341.git
cd fbcp-ili9341
mkdir build
cd build
cmake -DWAVESHARE_ST7789VW_HAT=ON -DST7789VW=ON -DSPI_BUS_CLOCK_DIVISOR=20 ..
make -j
sudo ./fbcp-ili9341
 
```
-Quit the script with `CTRL+C`  
- Auto-start when Power on  
```
sudo cp /home/pi/fbcp-ili9341/build/fbcp-ili9341 /usr/local/bin/fbcp  
```
- Create new file  
```
nano /etc/rc.local  
```
-add `fbcp&` before `exit 0`  
-----------------------------------------pisuagr & debug-------------------------------

```
wget http://cdn.pisugar.com/release/pisugar-power-manager.sh  
sudo bash pisugar-power-manager.sh -c release  
```
-enabling pisugar-server on startup  
```
sudo systemctl enable pisugar-server
sudo systemctl start pisugar-server
```
-Installing wiringpi for gpio tools  
```
git clone https://github.com/WiringPi/WiringPi.git
cd WiringPi
./build
```
-----------prep rpi-update----------------------
-rpi-update test for the i2c module  
```
cd /
```
-Create this file:  
```
sudo nano .curlrc
```
-Add this:  
```
--keepalive-time 2
```
-Start firmware update  
```
sudo rpi-update
```
--------------------build module for i2c pisugar2-------------------- https://gist.github.com/kellertk/ceeda942f6470b8ba8211a80f024004d
```
uname -a
```
-output:  
```
Linux Pwnagotchi 5.10.63-v7+ #1450 SMP Wed Sep 8 14:29:23 BST 2021 armv7l GNU/Linux
```
```
lsb_release -a
```
-output:
```
No LSB modules are available.
Distributor ID: Raspbian
Description:    Raspbian GNU/Linux 10 (buster)
Release:        10
Codename:       buster
```  
