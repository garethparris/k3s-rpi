# Raspberry Pi Hardware Build

K3S Build and Configuration with Raspberry PIs

## Hardware

I am using the following hardware:

- 2x Raspberry PI Model 4B with 4GB RAM using a PoE hat
- 2x Raspberry PI Model 4B with 8GB RAM using a PoE hat
- 1x Raspberry PI Model 3B+ with 1GB RAM using a PoE hat
- 1x Raspberry PI Model 3B with 1GB RAM using power adapter

On the Raspberry PI 4B models I am using a 64GB SD card connected via a USB 3.0 SD Card reader instead of the built-in SD card slot for increased data transfer speed.

![Hardware Front View](./images/rack-front.png)

In the Raspberry PI 3B models I am using a 64GB SD card directly.

Network connectivity is provided via a Ubiquity Unifi 16 port PoE switch.

![Ubiquity Switch Lite 16 PoE](./images/unifi-switch.png)

## Initial Setup

The steps below are all executed from Mac OS using Kitty and Terminus.

- Download [Raspberry PI Imager](https://www.raspberrypi.com/software/)
- Download latest [64-bit Raspberry PI OS Lite](https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz)
- Install the OS on each SD card using the imager. I created a profile to allow SSH access, set a username and password, and set the localisation.
- I am using DHCP to allocate each device with a fixed IP address in a dedicated development network (`192.168.3.0/27`) created in the Ubiquity Network management interface.
  ![Device Network Settings](./images/unifi-nodes.png)

## Raspberry PI Setup

- Connect to each RPi using SSH:

```bash
ssh username@192.168.3.xxx
```

- Update the Raspberry PI:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

- Turn off the swapfile:

```bash
sudo swapoff -a
sudo nano /etc/dphys-swapfile
```

- Set: `CONF_SWAPSIZE=0`

- Enable cgroup:

```bash
sudo nano /boot/firmware/cmdline.txt
```

- Add:

```bash
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

- Disable other services:

```bash
sudo nano /etc/modprobe.d/raspi-blacklist.conf
```

- Add:

```bash
# WiFi
blacklist brcmfmac
blacklist brcmutil
# Bluetooth
blacklist btbcm
blacklist hci_uart
```

- Continue disabling services:

```bash
sudo systemctl disable bluetooth && sudo systemctl stop bluetooth
sudo systemctl disable avahi-daemon && sudo systemctl stop avahi-daemon
sudo systemctl disable triggerhappy && sudo systemctl stop triggerhappy
```

- If using the UCTronics Rack with their OLED screens you need to enable I2C interface and install the OLED display software.

![UCTronics OLED Display](./images/oled-display.png)

- Emable the I2C interface using `raspi-config`:

```bash
sudo raspi-config
```

- Install GIT and pull down the repository:

```bash
sudo apt-get install git
git clone https://github.com/UCTRONICS/U6143_ssd1306.git
sudo nano /etc/rc.local
```

- Add:

```bash
cd /home/pi/U6143_ssd1306/C
sudo make clean
sudo make
sudo ./display &
```

- Reboot:

```bash
sudo reboot
```

## References

[plone.lucidsolutions.co.nz/hardware/raspberry-pi/3/disable-unwanted-raspbian-services](https://plone.lucidsolutions.co.nz/hardware/raspberry-pi/3/disable-unwanted-raspbian-services)

## More Images

![Hardware Top View](./images/rack-top.png)

## Notes

- To get Raspberry PI memory:

```bash
  grep MemTotal /proc/meminfo`
```
