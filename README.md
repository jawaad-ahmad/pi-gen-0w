# pi-gen-0w

_Tool used to create Raspbian images customized from the standard
raspberrypi.org Raspbian Lite image._


## Objective

To build a custom Raspbian Lite image that can be burned onto a minimal microSD
card to allow a potentially headless Raspberry Pi Zero W to immediately come
up, connect to a known WPA2 AES wireless network, and provide an SSH server
for remote access on the LAN with a read-only root and a tmpfs overlay.


## Differences

This section outlines the high-level differences from the parent repository
at the time this repo was forked.

 * Stage 1:
    * `ssh` file placed in boot partition to enable ssh by default
 * Stage 2:
    * Modified `wpa_supplicant.conf` to capture the configuration for a WPA2
      AES network, using variables `WIFI_SSID` and `WIFI_PSK` that can be
      provided in the config.
    * Removed fake-hwclock and some other packages not needed for a headless
      box with read-only root.
 * Stages 3, 4, and 5 disabled.
 * Created new Stage 3 installing the following packages:
    * openssh-server
    * python
 * Added Stage 6:
    * Added docker and docker-compose. (TODO)
    * Added initial-setup script infrastructure for future stages to hook-into
      a first-run/run-once setup finalization that will run the first time the
      Raspbian is booted with the SD card.
 * Added Stage 98 making changes appropriate for a read-only root filesystem.
 * `export-image`:
    * Added a second `8.8.4.4` name server to `resolv.conf`.
    * Added a file to `/etc/network/interfaces.d` for configuring `wlan0`.
       * The file contains the stanza to set up wlan0 as a DHCP client.
       * The file contains commented-out stanza for convenience in case I want
         to set up wlan0 with a static IP address if desired.
    * Modified `prerun.sh` to reduce the amount of space allocated to the root
      partition from 800 MB extra down to 200 MB extra.
 * Added Stage 99 for tmpfs overlay over root filesystem.


## Config

The modified wpa_supplicant.conf takes advantage of the following additional
environment variables:

 * `WIFI_SSID` (Default: unset)

   The SSID of the WPA2 network to which the Raspberry Pi Zero W will connect.

 * `WIFI_PSK` (Default: unset)

   The WPA2 pre-shared key to be used when connecting to the `WIFI_SSID`.


## Output

I'm only interested in running the Docker build. Running the build produces
the usual Stage 2 "lite" build, and now also produces a new Stage 3 "sshd"
build that I can use as an SD card image on a headless Raspberry Pi Zero W
once the image is tweaked to include the actual wi-fi SSID and password.


## Build Procedure

To build the image, I run the following:

   ```
   $ cat << END_OF_FILE > config
   IMG_NAME=Raspbian_0w
   IMG_NAME=Raspbian_0w
   WIFI_SSID=MySsid
   WIFI_PSK=MyPsk
   END_OF_FILE
   $ nohup make clean all
   ```

Monitor nohup.out. When complete, images will be stored in the `deploy`
subfolder.


### Clean-up

   ```
   $ docker rm -v pigen_work
   ```


## Deploy Procedure

On success, the `deploy` subfolder contains the zip files of the images. The
desired zip file can be unzipped and written on my SD card.

**Warning:** Proceeding will overwrite all data on the SD card.

**Warning:** Ensure the desired device name is used. Specifying an incorrect
device name will cause data on an unintended device to be overwritten, such
as your computer's hard drive.

To minimize data loss due to accidental copy/paste, the device used below is
a fake `/dev/sdX`. Substitute as necessary for your own SD card's device.

Run the following:

   ```
   $ cd deploy
   $ unzip image_2018-xx-xx-Raspbian_0w-stage3.zip
   ```

Insert SD card, ensure it is available, and get its device name. (Run in a
separate window to monitor.)

   ```
   $ sudo tail -F /var/log/syslog
   ```

Write the image to the SD card:

   ```
   $ sudo dd if=2018-xx-xx-Raspbian_0w-stage3.img of=/dev/sdX bs=4M conv=fsync; sync
   ```

Eject the SD card and place in Raspberry Pi Zero W device.

Apply power to the device.


### First Boot

If an HDMI monitor is connected to the device, then boot-up will indicate that
the device is booting successfully. It should eventually display a log-in
prompt, and it should also display its IP address on the wlan0 interface just a
few lines above the log-in prompt if it connected to the target wireless
network successfully.

Without an HDMI monitor, tools such as `nmap` from another computer should be
able to find the new device, or examine the list of connected devices from your
router's web interface.

For example:

   ```
   $ sudo nmap -sP 192.168.1.0/24
   ```

It might help to run this command once before applying power, and then again a
few minutes after applying power after giving time for the device to boot up
and connect to the network, and finally, comparing the output of the two runs.

With the SSH server running and the IP address known, we should be able to
log-into the device from another computer on the network. The following example
assumes IP address 192.168.1.99 for the device:

   ```
   $ ssh pi@192.168.1.99
   ```

Once logged-in, we might want to get some information about the device.

   ```
   $ cat /proc/cpuinfo
   $ cat /sys/class/net/wlan0/address
   ```

The serial number would be good to note down, but the MAC address would be
useful if configuring static DHCP settings on the wireless router.

Before anything else, we likely want to change the default password:

   ```
   pi$ passwd
   ```

We might also want to update packages:

   ```
   pi$ sudo apt-get -y update
   pi$ sudo apt-get -y upgrade
   ```

We might want to run raspi-config to change other common settings:

   ```
   pi$ sudo raspi-config
   ```

Finally, we might want to clean up apt:

   ```
   pi$ sudo apt-get clean
   ```

# Troubleshooting

I had difficulty putting this together or adding to the customization. I picked
up the following along the way.

 * Enter the container with `make enter-container` to troubleshoot what's going
   on and for any one-time hacks.
 * Take advantage of the SKIP files mentioned in the parent repo's README to
   skip earlier stages when rebuilding and troubleshooting the current stage;
   `make rebuild-last-stage` helps with this.
 * The rootfs will not be rebuilt (i.e. carried over from an earlier stage) if
   it already exists in the container. If unsure about its contents or to play
   it safe, enter the container and blow it away under
   `/pi-gen/work/.../stageX` before running `make rebuild-last-stage`.
 * Prior to running `make clean all`, it helps to clear the clutter from past
   runs. If rebuilding everything from scratch:
    * Remove all SKIP files from earlier stages (`rm stage{0,1,2,...}/SKIP`).
    * Remove docker containers (`docker ps -a` followed by `docker rm`).
    * Remove applicable docker images (`docker images` followed by
      `docker rmi`). The only one to really keep would be the `debian:buster`
      image to avoid downloading it over and over.
    * Remove/prune unreferenced docker volumes (`docker volume ls` and
      `docker volume prune -f`).
    * Repeat these commands as needed to ensure all applicable containers,
      images, and volumes are cleared out and then run `make clean all` to
      rebuild from scratch.

## `64 Bit Systems`
Please note there is currently an issue when compiling with a 64 Bit OS. See https://github.com/RPi-Distro/pi-gen/issues/271

## `binfmt_misc`

Linux is able execute binaries from other architectures, meaning that it should be
possible to make use of `pi-gen` on an x86_64 system, even though it will be running
ARM binaries. This requires support from the [`binfmt_misc`](https://en.wikipedia.org/wiki/Binfmt_misc)
kernel module.

You may see the following error:

```
update-binfmts: warning: Couldn't load the binfmt_misc module.
```

To resolve this, ensure that the following files are available (install them if necessary):

```
/lib/modules/$(uname -r)/kernel/fs/binfmt_misc.ko
/usr/bin/qemu-arm-static
```

You may also need to load the module by hand - run `modprobe binfmt_misc`.
