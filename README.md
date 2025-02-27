# How to update SV07/SV07+ as well as SV06/SV06+ with KliPad to Debian Bookworm, flash latest firmware on the MCU (for SV07/SV07+ only) and QOL stuff for the printer.

Tools and software needed:
1. 2 x 8 gig or more USB Sticks (can be done with one USB stick, but this guide assumes 2, as I did it with 2 before I figured I could only use one)
2. SD card or microSD card with SD adapter, any size. It must have MBR partition table and formatted in FAT32.
3. Latest release of [unofficial Armbian for MKSPI](https://github.com/redrathnure/armbian-mkspi/releases) (bookworm current or edge, I will be referencing Armbian-unofficial_24.2.0-trunk_Mkspi_bookworm_current_6.6.17)
4. Sovol's recompiled rk3328-roc-cc.dtb that I uploaded to [this repository](https://github.com/vasyl83/sv07update/raw/main/rk3328-roc-cc.dtb). Huge thanks to [Thorsten Maerz](https://netztorte.de/3d/doku.php?id=start) for figuring it out and making this file available on his site.
5. Computer (or laptop) with a free USB slot
6. Cable to connect you computer (or laptop) to USB-C port on the KliPad, so USB-A to USB-C or USB-C to USB-C

This guide assumes that you have the default Sovol image flashed on the KliPad, it doesn't matter what firmware version nor if it boots properly or is in a boot loop.

**Also backup your printer.cfg and any other config files you may need.**

## 1. Preparing USB sticks
Download [Balena Etcher portable](https://etcher.balena.io/#download-etcher) and run it.

Click "Flash from file" and select the image you downloaded (I will be using Armbian-unofficial_24.2.0-trunk_Mkspi_bookworm_current_6.6.17.img.xz as example)

![balena main window](<images/Screenshot 2024-03-14 201122.png>)

![file selection](<images/Screenshot 2024-03-14 203117.png>)

Click "Select target" and point it to USB stick

![usb selection](<images/Screenshot 2024-03-14 203303.png>)

Click Select 1. Then wait for it to finish.

Afterwards extract the .xz file into .img and copy that file to the second USB stick as well as rk3328-roc-cc.dtb.

The contents of the second USB should look like this:

![2nd usb](<images/Screenshot 2024-03-14 203954.png>)

## 2. Plugging everyhting into the printer.

Now connect both USBs and your PC into the KliPad.

![all connected](images/20240314_165251.jpg)

On you computer Open up Device Manager and expand `Ports (COM & LTP)`. You should see `USB-SERIAL CH340` followed by a COM port number (take note ot if, in my case its COM3):

![com3](<images/Screenshot 2024-03-14 195202.png>)

Start up [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

Input the following settings:
1. Conenction type: Serial
2. Serial line: COM3 (or whatever COM Device Manager shows)
3. Speed: 1500000

![putty](<images/Screenshot 2024-03-14 195248.png>)

Click Open, you should now see a blank black window, press Enter a few times untill you see a login prompt:

![putty login](<images/Screenshot 2024-03-14 195332.png>)

Enter the username and password you use (defaults are user:mks password:makerbase)

halt the system:
`sudo halt`

## 3. Booting from USB

Make sure that the KliPad screen is blank, PuTTY window should remain like this (what you see shouldn't be exactly the same, only the result of the command should be the same):

![putty halted](<images/Screenshot 2024-03-14 210833.png>)

Now simultaniously hold spacebar on your PC and press the power button on the KliPad (the small one on the right side of the screen) for about 10 seconds. Once you release the button you should see text scrolling on PuTTY window, once it stops scrolling, the last line should say `Hit any key to stop autoboot:  0` and you see the cursor adding spaces, release the spacebar, it should look something like this:

![boot interrupt](<images/Screenshot 2024-03-14 211755.png>)

Delete all the spaces and type in:
`run bootcmd_usb0`

It will now boot from the USB.

![usb boot](<images/Screenshot 2024-03-14 212446.png>)

Once the boot is completed you should see on the screen `mkspi login: root (automatic login)`

![root auto login](<images/Screenshot 2024-03-14 212713.png>)

Once it finishes booting, you will be prompted to create root password, enter anything. Select your preferored shell (I use zsh), and once it asks you to create a new user, press Ctrl-C to abort and drop into the shell. You are now booted from USB.

## 4. Confirming USB boot

To make sure the system booted from USB type in `lsblk` it will list all the block devices discovered on the KliPad (block devices are basically storage devices). If everything worked properly you should see the following:

![lsblk](<images/Screenshot 2024-03-14 155921.png>)

sda is our boot USB, as you can see /boot and / are mounted on sda1 and sda2.  

sdb is the second USB containing the img and dtb files.

mmcblk1 is the internal EMMC module.

## 5. Mounting second USB stick

To acces the second USB we need to mount it, first lets create a folder to mount it on.

`mkdir /mnt/temp` - create a folder temp under /mnt

`mount /dev/sdb1 /mnt/temp` - mount the first partition of sdb under /mnt/temp

If you get a hint about fstab being modified, it's normal, don't pay attention to that hint.

`cd /mnt/temp` - change directory to /mnt/temp

`ls` - list contents

![ls](<images/Screenshot 2024-03-14 214406.png>)

You should see the list of all the files on your second USB stick. (The screeshot doesn't have rk3328-roc-cc.dtb on it, but if you followed every step you should see it there)

## 6. Flashing the image onto EMMC

`dd if=Armbian-unofficial_24.2.0-trunk_Mkspi_bookworm_current_6.6.17.img of=/dev/mmcblk1 status=progress` - this command will copy block by block the contents of the .img file onto /dev/mmcblk1 (EMMC module) and will show the progress. This can take a while so be patient, once it finishes you should see something like this:

![dd](<images/Screenshot 2024-03-14 215026.png>)

## 7. Rebooting into the new environment

You can noow shut down the KliPad by issuying `halt`, or simply pull out the bootable usb stick. You should then see a message `EXT4-fs (sda2): shut down requested (2)`.

Now you should only have the second USB stick connected (the one with dtb and img files).

Press and hold the power buttong on KliPad for about 10 seconds (do not press the spacebar this time, let it boot normally).

If all went well you should see `mkspi login: root (automatic login)` and once again it will ask you to create root password. This time enter the password you will use for root. 

Then, follow all the prompts, select the shell, enter a username for a new user. If you don't want to mess much with printer.cfg later, I suggest you name the user `mks`, like it was before. If you don't you will have to adjust any references from /home/mks to /home/"username you selected". 

After username enter a real name for your user, it can be anything. 

When it asks you for the timezone, it will fail to determine it automatically since there is no internet connection yet. For locale, I suggest you enter 99 to use en_US.UTF-8 which is the default US English locale. 

Continue by following the promts and selecting your timezone. Confirm that the info is correct to finally be dropped into the shell.

![locale-shell](<images/Screenshot 2024-03-14 220532.png>)

## 8. Mounting USB stick and enabling the wifi.

Running `lsblk` should now only show sda and mmcblk1 (sdb from before becomes sda). / and /boot should also be mounted on mmcblk1.

![new lsblk](<images/Screenshot 2024-03-14 220342.png>)

`sudo mkdir /mnt/temp`

`sudo mount /dev/sda1 /mnt/temp`

`ls /mnt/temp`

![ls after mount](<images/Screenshot 2024-03-14 232315.png>)

The USB stick is now mounted to /mnt/temp and we can see the contents.

Lets copy rk3328-roc-cc.dtb and get the wifi working.

`sudo cp /mnt/temp/rk3328-roc-cc.dtb /boot/dtb/rockchip` - this will copy the dtb file into the correct location, 

For newer armbian we need to copy the same file under different name (rk3328-mkspi.dtb): 

`sudo cp /mnt/temp/rk3328-roc-cc.dtb /boot/dtb/rockchip/rk3328-mkspi.dtb`
`sudo cp /mnt/temp/rk3328-roc-cc.dtb /boot/dtb/rk3328-mkspi.dtb`

then we need to reboot, `sudo reboot`

![reboot after dtb](<images/Screenshot 2024-03-14 233024.png>)

Wait for it to reboot, login again and run `ip a` - it will show all the ethernet and wifi adapters, we are looking for the presence of `wlan0`

![wlan0](<images/Screenshot 2024-03-14 233309.png>)

As you can see, for me it appears as the 4th device.

To connect you need to run `sudo nmtui` it will open Network Manager. Once KlipperScreen is installed it can be done from the KlipperScreen.

![nmtui](<images/Screenshot 2024-03-14 233543.png>)

Select `Activate Connection` then thelect the SSID click Enter.

![SSID select](<images/Screenshot 2024-03-14 233738.png>)

Enter the password and click Enter.

![ssid pass](<images/Screenshot 2024-03-14 234029.png>)

Once connected you should see `*` Next to SSID name.

![activated ssid](<images/Screenshot 2024-03-14 234118.png>)

Exit the program and run `ip a` again, now under wlan0 you should see the IP adress:

![ip assigned](<images/Screenshot 2024-03-14 234346.png>)

Now you can connect to the KliPad with PuTTY using the IP instead of USB cable. No need to stay next to the printer anymore.

![putty ip connection](<images/Screenshot 2024-03-14 234709.png>)

## 9. Updating

`sudo apt-get update`

`sudo apt-get upgrade -y`

![upgrade held back](<images/Screenshot 2024-03-14 235300.png>)

As you can see there were no updates and a few packages were held back, wich is normal, to update the kernel the image must be reflashed.

## 10. Installing KIAUH

`git` is installed by default, so all you need to do is follow the install instructions on [KIAUH site](https://github.com/dw-0/kiauh)

`cd ~ && git clone https://github.com/dw-0/kiauh.git`

![kiauh git](<images/Screenshot 2024-03-14 235800.png>)

`./kiauh/kiauh.sh` will run KIAUH

![kiauh](<images/Screenshot 2024-03-15 000058.png>)

Install Klipper, Moonraker, Mainsail (or Fluidd or both), KlipperScreen and Crowsnest (if you will use a webcam) plus anything else you use.

Now we need to install klipper_mcu process to use MKSPI as MCU.

```
cd ~/klipper/
sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
sudo systemctl enable klipper-mcu.service
```
Reboot:
```
sudo reboot
```

## 11. KlipperScreen orientation

After reboot the screen should now work but will be in the wrong direction. Let's correct that.

![wrond direction](images/20240315_003131.jpg)

```
sudo nano /etc/X11/xorg.conf.d/01-armbian-defaults.conf
```

then add the following into that file:
```
Section "Device"
Identifier "default"
Driver "fbdev"
Option "Rotate" "CW"
EndSection
Section "InputClass"
Identifier "libinput touchscreen catchall"
MatchIsTouchscreen "on"
MatchDevicePath "/dev/input/event*"
Driver "libinput"
Option "TransformationMatrix" "0 1 0 -1 0 1 0 0 1"
EndSection
```
![nano](<images/Screenshot 2024-03-15 004128.png>)

Ctrl-X then Enter to save the file.

Restart KlipperScreen with: 
```
sudo systemctl restart KlipperScreen.service
```

Now everything is in correct direction and touch works!

![good direction KlipperScreen](images/20240315_004409.jpg)

## 12. Updating the host MCU RPI & MCU firmware

This section is pretty much a copy/paste from excellent post by [3DPrintDemon](https://github.com/3DPrintDemon/How-to-Update-Sovol-Klipper-Screen-To-Latest-Klipper-SV06-and-SV07?tab=readme-ov-file#updating-the-host-mcu-rpi--mcu-firmware)

Log into you printer with SSH (using PuTTY)

```
cd ~/klipper/
make menuconfig
```
This will open a window with options for klipper.

![menuconfig default](<images/Screenshot 2024-03-15 233117.png>)

In the menu, set `Microcontroller Architecture` to `Linux process`, then save and exit by pressing `Q`.

![menuconfig linux process](<images/Screenshot 2024-03-15 233556.png>)

Next, we stop klipper install firmware and start klipper again.

```
sudo systemctl stop klipper.service
make flash
sudo systemctl start klipper.service
```

![klipper restarted](<images/Screenshot 2024-03-15 234156.png>)

Now MKS PI MCU is up to date, next lets prepare the firmware update the MCU inside the printer.

```
cd ~/klipper/
make menuconfig
```

This time we will select `STMicroelectronics STM32` as `Micro-controller Architecture`, next `Processor model (STM32F103)` should be selected by default. Then set `Bootloader offset` to `28KiB bootloader`. Finally ` Communication interface` to `Serial (on USART1 PA10/PA9)`.

![mcu setup](<images/Screenshot 2024-03-15 235401.png>)

Again, `Q` to save and exit. Then,

```
make
```

![klipper.bin](<images/Screenshot 2024-03-15 235920.png>)

Now let's copy klipper.bin that we just compiled to /home/mks so it would be easy to get it from the printer and onto SD card.

```
cp out/klipper.bin ~
```

![alt text](<images/Screenshot 2024-03-16 000423.png>)

Now start [WinSCP](https://winscp.net/eng/index.php) or any other SFTP program, connect to the printer (by default it will connect to /home of the user, so in our case /home/mks where we copied firmware.bin)

![winscp](<images/Screenshot 2024-03-16 000901.png>)

![copy interface](<images/Screenshot 2024-03-16 000957.png>)

Now rename the file to anything, I named mine klipper-16-03-2024.bin, you will need to change the file name everytime you attempt a flash, once a name is used it won't work if you use it again.

And copy it to the SD card. **Make sure the SD card is MBR and the partition is FAT32** you can get that information from Disk Managment tool (right-click start button and select Disk Managment). If your SD card is bigger than 32 Gigs it will not let you format a partition with FAT32, if that is the case, simply create a partition smaller than 32 Gigs. Here what is should look like, as you can see I have a partition smaller than 32 Gigs in order to format it FAT32:

![mbr](<images/Screenshot 2024-03-16 001728.png>)

Power down the printer, fully disconnect the KliPad and remove printer's front panel. You should see the MCU with a prominent SD card slot. 

![no front panel](images/20240316_003404.jpg)

Put your SD card into the slot. Power on the printer **(with KliPad disconnected, this is important!)**, wait for 30 seconds and then power off the printer. Nothing will happen visually. Reconnect everything, power on the printer again and lets see if everything worked and finally set up printer.cfg

## 13. printer.cfg

Now simply copy back your backed up printer.cfg to /home/mks/printer_data/config or copy/paste its contents into printer.cfg using Mainsail or Fluidd (even if you have an error saying can't connect to the mcu, which is normal because the default printer.cfg lack info about SV07, you should still be able to access the configs).

Restart klipper.

```
sudo systemctl restart klipper.service
```
Or use Mainsail or Fluidd.

Now we need to comment out or remove all the references to plr (power loss recovery). It should be possible to reenable it at the end, but for now, since we are not using Sovol's image we need to remove references to it.

Find the following lines and comment them out or delete them:

```
[include plr.cfg]
```
```
SAVE_VARIABLE VARIABLE=was_interrupted VALUE=False
RUN_SHELL_COMMAND CMD=clear_plr
clear_last_file
G31
```
```
[gcode_macro PRINT_START]
gcode:
    SAVE_VARIABLE VARIABLE=was_interrupted VALUE=True
[gcode_macro PRINT_END]
gcode:
    SAVE_VARIABLE VARIABLE=was_interrupted VALUE=False
    RUN_SHELL_COMMAND CMD=clear_plr
    clear_last_file
```
Again, huge thanks to [Thorsten Maerz](https://netztorte.de/3d/doku.php?id=start) for figuring it all out.

If you had any custom includes in your priner.cfg make sure to copy everything over or comment out any includes, if not you may get errors from klipper for not finding the files.

If you are like me and used klipper-backup you can simply `git clone` your backup repository and everything will be restored.

Finally, either reboot the host via Mainsail or by:
```
sudo reboot
```

Once the host is booted, log into Mainsail, click on Machine and now you should see that the version on mcu, mcu rpi and host are all the same (0.12.0-124 as the time of writing), that the OS is `Armbian-unofficial 24.2.0-trunk bookworm` and not buster like on the Sovol image. Finally, everything (including system packages) is up to date.

![final machine](<images/Screenshot 2024-03-16 012621.png>)

## 14. Enabling boot splash screen

```
sudo nano /boot/armbianEnv.txt
```

Change `bootlogo=false` to `bootlogo=true`

![bootlogo](<images/Screenshot 2024-03-16 184455.png>)

List all installed themes:
```
sudo plymouth-set-default-theme -l
```

Select one like `solar` and then

```
sudo plymouth-set-default-theme solar
sudo update-initramfs -u
```
![initramfs](<images/Screenshot 2024-03-16 184938.png>)

Now reboot and you will see a cute animation instead of lines of text.

```
sudo reboot
```

## 15. Accelerometer (Input Shaper)

Resonance measurement require additionnal stuff to be installed.

```
sudo apt-get update
sudo apt-get install python3-numpy python3-matplotlib libatlas-base-dev libopenblas-dev
~/klippy-env/bin/pip install -v numpy
```

Now using Input Shaper on KilpperScreen will not give out any errors.

## 16. Automount USB

Use [makerbase-beep-service.deb](makerbase-beep-service.deb) made by [Thorsten Maerz](https://netztorte.de/3d/doku.php?id=start).

```
sudo dpkg -i makerbase-beep-service.deb
```

## 17. Beeper

First, make sure that you have G-Code Shell Command installed (run KIAUH, option 4, the 8), then we need to add new rules to udev:

```
sudo nano /etc/udev/rules.d/90-gpio.rules
```

Add 2 following lines to the file:

```
SUBSYSTEM=="gpio", KERNEL=="gpiochip*", ACTION=="add", PROGRAM="/bin/sh -c 'chown root:dialout /sys/class/gpio/export /sys/class/gpio/unexport ; chmod 220 /sys/class/gpio/export /sys/class/gpio/unexport'"
SUBSYSTEM=="gpio", KERNEL=="gpio*", ACTION=="add", PROGRAM="/bin/sh -c 'chown root:dialout /sys%p/active_low /sys%p/direction /sys%p/edge /sys%p/value ; chmod 660 /sys%p/active_low /sys%p/direction /sys%p/edge /sys%p/value'"
```

Next, we need to add 2 macros to printer.cfg:

```
[gcode_macro BEEP]
gcode:
  {% set beep_count = params.BC|default("3") %}
  {% set beep_duration = params.BD|default("0.2") %}
  {% set pause_duration = params.PD|default("1") %}
  RUN_SHELL_COMMAND CMD=beep PARAMS='{beep_count} {beep_duration} {pause_duration}'

[gcode_shell_command beep]
command: bash /home/mks/printer_data/config/macro/macro-beep.sh
timeout: 10
verbose: False
```

Finally let's create the shell script the macro is referencing:

```
nano /home/mks/printer_data/config/macros/macro_beep.sh
```

Paste the followting into that shell script:

```
#!/bin/bash
# usage: beep.sh [BEEPCOUNT] [BEEPDURATION] [PAUSEDURATION]

# Output raw passed parameters
echo "Raw parameters: $@"

# Default values
BEEPCOUNT=${1:-3}
BEEPDURATION=${2:-0.1}
PAUSEDURATION=${3:-0.5}

# Output all passed parameters
echo "Beep count: $BEEPCOUNT, beep duration: $BEEPDURATION, pause duration: $PAUSEDURATION"

# Function to play a beep
play_beep() {
    echo 1 > /sys/class/gpio/gpio82/value
    sleep $BEEPDURATION
    echo 0 > /sys/class/gpio/gpio82/value
}

# Play the beep for the specified count
for (( i=0; i<BEEPCOUNT; i++ )); do
    play_beep
    sleep $PAUSEDURATION
done
```

I don't know if the next command is strictly necessary, but shell scripts usually should be executable, so let's make it executable:

```
sudo chmod +x /home/mks/printer_data/config/macros/macro_beep.sh
```
Now you have a macro that you can reference in other macros that can beep.

## 18. Cleanup

In order to get some free space back on the emmc, it is wise to delete cached packages. Simply run

```
sudo apt clean
```

I was able to get 400 megs back.

## 19. Additionnal tools

| Addon Name | Usecase |
| :----------- | :----------- |
| [Klipper-backup](https://staubgeborener.github.io/klipper-backup/) | backup your configs to GitHub |
| [Klipper-Adaptive-Meshing-Purging](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) | eventhough Klipper 12 has adaptive meshing backed in, I still use KAMP for the purging |
| [Moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) | best add on I found for timelapses |
| [Shell-command](https://github.com/dw-0/kiauh/blob/master/docs/gcode_shell_command.md) | Installed via KIAUH, option 4. Advanced. Required for Klipper-backup and plr |
| [Moonraker-telegram-bot](https://github.com/nlef/moonraker-telegram-bot) | Get printer updates sent to your telegram channel |

## 20. Restoring plr

## 21. My printer.cfg and other config files.

If anyone wants to see [them for reference](https://github.com/vasyl83/sv07/tree/main/printer_data/config). I changed some macros a little and I use BTT filament sensor.
