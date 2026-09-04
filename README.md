# tp-link-t2u-nano-ubuntu-24.04
Working fix for TP-Link Archer T2U Nano 2357:011e on Ubuntu Server 24.04 / kernel 6.8 using rtw88 DKMS.
TP-Link Archer T2U Nano (2357:011e) on Ubuntu Server 24.04 / Kernel 6.8

Working fix using lwfinger/rtw88

I ran into this while installing a TP-Link Archer T2U Nano AC600 USB Wi-Fi adapter on a fresh Ubuntu Server 24.04.4 LTS system.

Tested environment

Ubuntu Server: 24.04.4 LTS

Kernel: 6.8.0-139-generic

Adapter: TP-Link Archer T2U Nano

USB ID: 2357:011e

Chipset family: RTL8811AU / RTL8821AU

Secure Boot: disabled

The final working solution was the lwfinger/rtw88 DKMS backport, which provides the rtw_8821au module.

1. Identify the adapter

Run:

lsusb

Mine showed:

ID 2357:011e TP-Link AC600 wireless Realtek RTL8811AU [Archer T2U Nano]

Before installing a driver, it is also useful to check whether Linux has already created a Wi-Fi interface:

iw dev
ip link

In my case, there was no wireless interface initially.

2. What did NOT work

Ubuntu rtl8812au-dkms package

I first tried:

sudo apt install rtl8812au-dkms
sudo modprobe 8812au

The module loaded, but the adapter never appeared in iw dev.

Checking the module aliases showed that the packaged driver did not include USB ID 2357:011e.

modinfo 8812au | grep -i 2357

The package was therefore removed:

sudo apt remove -y rtl8812au-dkms

An rtl8812au GitHub driver

A newer rtl8812au repository appeared to list the T2U Nano in its README, but the actual driver build had CONFIG_RTL8821A = n.

After enabling RTL8821A, the source failed to compile because required RTL8821A files were missing.

That path was abandoned.

Legacy 8821au-20210708 driver

I then tried the dedicated RTL8811AU / RTL8821AU driver.

The adapter ID 2357:011e is included in its supported-device list, but on this Ubuntu 24.04 / kernel 6.8 installation, DKMS failed to compile because of cfg80211 API type mismatches.

Representative errors were:

error: passing argument 1 of 'cfg80211_new_sta' from incompatible pointer type
error: passing argument 1 of 'cfg80211_del_sta' from incompatible pointer type

and several cfg80211_ops callbacks expected struct net_device * while the driver supplied struct wireless_dev *.

I removed that failed driver before continuing.

3. Working solution: lwfinger/rtw88

Install the build requirements:

sudo apt update
sudo apt install -y git dkms build-essential linux-headers-generic iw

Clone the driver:

mkdir -p ~/src
cd ~/src
git clone https://github.com/lwfinger/rtw88.git
cd rtw88

Install it through DKMS:

sudo dkms install $PWD

On my system, DKMS successfully built and installed modules including:

rtw_core
rtw_usb
rtw_8821a
rtw_8821au

Verify:

dkms status

You should see an installed rtw88 entry for your running kernel.

Then reboot:

sudo reboot

4. Verify after reboot

Run:

iw dev

The adapter finally appeared:

phy#0
    Interface wlx...
        type managed

Also check:

ip link

You should now see a third interface in addition to lo and your Ethernet interface.

For example:

3: wlx40ed008a623c: <BROADCAST,MULTICAST> ...

At this point the driver is working. The interface may still show state DOWN until you configure or connect it to a Wi-Fi network.

5. Scan for networks

Bring the interface up:

sudo ip link set wlxYOURMAC up

Then scan:

sudo iw dev wlxYOURMAC scan | grep SSID

Replace wlxYOURMAC with the wireless interface name shown by iw dev.

If your SSID appears, the adapter and driver are operating correctly.

Notes

Why 2357:011e matters

With USB Wi-Fi adapters, the marketing name alone is not enough. Always check:

lsusb

and use the vendor ID when researching Linux support.

For this adapter:

2357:011e

is the useful identifier.

Avoid multiple competing out-of-tree drivers

Before trying another driver, remove failed or conflicting DKMS modules:

dkms status

Multiple drivers attempting to claim the same Realtek adapter can make troubleshooting much harder.

Kernel updates

Because this installation uses DKMS, the module should be rebuilt automatically when compatible new kernels are installed. It is still worth checking after a major kernel upgrade:

dkms status
iw dev

Short version

For Ubuntu Server 24.04 + kernel 6.8 + TP-Link Archer T2U Nano 2357:011e, the working path in my case was:

sudo apt install -y git dkms build-essential linux-headers-generic iw
mkdir -p ~/src
cd ~/src
git clone https://github.com/lwfinger/rtw88.git
cd rtw88
sudo dkms install $PWD
sudo reboot

After reboot:

iw dev
ip link

The Wi-Fi interface appeared successfully.

Why I am sharing this

This took several driver attempts and a fair amount of troubleshooting to isolate. I am posting it so the next person with the same USB ID and Ubuntu/kernel combination has a shorter path to a working system.

If your kernel, adapter revision, or USB ID is different, verify support before assuming the exact same steps will work.
