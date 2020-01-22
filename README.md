# Fast Switch Prime-Ubuntu-18.04

Nvidia Prime without rebooting. Assumes lightdm is installed.

Why does this exist? It restores the pre-18.04 'bbswitch' approach; it's much faster to change profiles and more reliable. However, it also takes advantage of recent improvements, so you can swap modes without rebooting.

This is not for Ubuntu beginners. If things go wrong, you need to know about virtual consoles and recovery mode and some basic systemd admin.

To install it, you need to know about git clone and you need to be able to change your display manager to lightdm.

Good support comes from this [thread](https://devtalk.nvidia.com/default/topic/1032482/linux/optimus-on-ubuntu-18-04-is-a-step-backwards-but-i-found-the-first-good-solution/).


# Dependencies:

You need [Rust](https://www.rust-lang.org/), to install from apt: `sudo apt install rustc cargo`.

Properly install the nvidia drivers the standard ubuntu way, from Additional Drivers.

If you have done this already, make sure you do 
```prime-select nvidia ``` 
to ensure that nvidia drivers are installed in your initramfs.

If you are reading this after installing the fast prime-select (this module), then you access the standard prime-select via `/usr/bin/prime-select`.
* This code assumes the nvidia drivers are installed. They are installed by the standard Ubuntu module, but if you did the standard 18.04 `prime-select intel`, they are no longer installed in the kernel. So do `prime-select nvidia` and reboot. Make sure you have the nvidia drivers running before proceeding.


* Ubuntu 18.04 (might works with other distros of similar age which are based on the vendor-neutral library approach, if you change some paths).

* [bbswitch](https://packages.ubuntu.com/bionic/bbswitch-dkms) (via `sudo apt install bbswitch-dkms`).

* [lightdm](https://wiki.ubuntu.com/LightDM) as the display manager.

```bash
sudo apt install lightdm
```

You can swap between display managers with `sudo dpkg-reconfigure lightdm`.
The Ubuntu install of the nvidia driver will also install nvidia-prime, Ubuntu's optimus module. The code supersedes that but you should leave the ubuntu package installed. 

Note: while testing this in a reinstall of Ubuntu 18.04, lightdm did not install properly on one laptop. Work-around: install xubuntu-desktop which relies on lightdm, but still kept ubuntu as the log-in session.


# How to build & install
Naturally, make sure you have `git` and `git clone` this repository.

```bash
cd prime_socket/src
sudo make install
```

# Usage

```bash
sudo prime-select intel|nvidia|query
```

Don't use the graphical switcher of the nvidia-control panel. It uses the standard debian way, which will rebuild your kernel image: it does this to remove the nvidia drivers with extreme prejudice when you swap to intel mode, which will stop this fast-switch method from working.

If you remove the nvidia modules using Ubuntu's standard (slow) method, you will need to use the standard method to put them back (by using the nvidia control panel to swap back to nvidia).
If you want to use the standard prime-select script, it is untouched at `/usr/bin/prime-select`.

The modified version at `/usr/local/bin` has priority in the path so if you need to use the standard script, be explicit about the path.

# Did it work?

After entering Intel mode, start a `sudo` shell and test if the card is off:
```bash
sudo -i
modprobe bbswitch
cat /proc/acpi/bbswitch
```


# Notes

You must have the nvidia drivers installed in your initramfs.
This will be true if you have installed the standard Ubuntu nvidia-drivers but it will not be true if you did the standard ```prime-select intel```.
See notes above. 

The first time you use sudo prime-select nvidia to change, you may get an error about a missing file `/usr/share/X11/xorg.conf.d/20-intel.conf` which the script tries to delete. 
Do: `sudo touch /usr/share/X11/xorg.conf.d/20-intel.conf` and repeat `sudo prime-select nvidia`.


## Prime sync for tear free laptop panel
* todo: fix this, it's a paper-cut.

# Uninstall

This code doesn't really disturb your system much. 
You could rename `/usr/local/bin/prime-select` to `/usr/local/bin/prime-select-fast` so that the standard script is no longer masked by the modified one.

Purge and reinstall the package nvidia-prime:
```bash
sudo apt purge nvidia-prime; sudo apt install nvidia-prime
sudo /usr/bin/prime-select nvidia
```

Revert to gdm3, install and select it as the default:
```bash
sudo apt install gdm3
sudo dpkg-reconfigure gdm3
```


You should be back to standard Ubuntu now. 

## Uninstall bbswitch-dkms
You installed the bbswitch-dkms module to get this working.
The standard Ubuntu approach doesn't use bbswitch (the decision which causes all the problems). I wouldn't expect any problems by leaving it installed, but it is unnecessary if you want to use the standard Ubuntu 18.04 approach to Optimus.


# Prime sync for tear free laptop panel

This tip applies to standard Ubuntu too. 

In nvidia mode, you'll get tearing on the laptop unless you enable prime sync.\
`sudo vi /etc/modprobe.d/zz-nvidia-modeset.conf`
and include this:
```bash
#enable prime-sync
options nvidia-drm modeset=1
```
and \
`sudo update-initramfs -u`

Tearing you see on non-laptop panels won't be fixed by prime sync. For that problem, you need to turn on pipeline-composition on the affected screens (via the nvidia control panel). Learn more on the nvidia developer linux forums.



# Troubleshooting: Display manager doesn't start?

This is not for Ubuntu beginners. You need to know about virtual consoles and recovery mode and some basic systemd admin.

First, you need access to a virtual console. 
Depending on what has gone wrong, you may be able to access a virtual console.
`sudo apt install openssh-server` is always helpful too.

Starting in recovery mode usually works to get a GUI login. (choose resume boot twice during the boot process). 
make sure the systemctl lines in the makefile worked by using systemctl status prime-socket.
This is what it should look like: the service should be active and running.
```bash
● prime-socket.service - Socket service for on the fly prime switching
   Loaded: loaded (/etc/systemd/system/prime-socket.service; enabled; vendor pre
   Active: active (running) since Fri 2018-06-15 08:41:00 AEST; 31min ago
 Main PID: 808 (prime_socket)
    Tasks: 2 (limit: 4915)
   CGroup: /system.slice/prime-socket.service
           └─808 /usr/local/bin/prime_socket

Jun 15 08:41:00 raffles systemd[1]: Started Socket service for on the fly prime
```

## Display manager doesn't start in intel mode
If you swap to intel, reboot and can't get the display manager working, this is probably because the nvidia drivers were not unloaded. 

## Intel-mode fix attempt 1:

boot in recovery mode, and choose "resume boot" (possibly twice)
This will probably get lightdm started, allowing you to log in.

Check if the service which unloads the nvidia drivers is working:
```bash
sudo -i
systemctl status nvidia-prime-boot.service
```

Here is an example of healthy output:
```bash
root@raffles:~# systemctl status nvidia-prime-boot.service
● nvidia-prime-boot.service - dGPU off during boot
   Loaded: loaded (/etc/systemd/system/nvidia-prime-boot.service; disabled; vendor preset: enabled)
   Active: inactive (dead)

Jun 10 10:24:09 raffles systemd[1]: Starting dGPU off during boot...
Jun 10 10:24:09 raffles systemd[1]: Started dGPU off during boot.
```

You may also find something useful in 
```bash
journalctl -e
```

You should not see an error telling you that bbswitch is not installed, because that means you didn't read the instructions above. Also, you should not see errors that no nvidia modules are installed, because that means you either did not install the nvidia drivers, or you removed them (perhaps by 18.04-standard `prime-select intel`, in which case `sudo /usr/bin/prime-select nvidia` and reboot. Please carefully read the installation instructions above ...

## Intel-mode Fix attempt 2
if you can't get to a graphical session even with recovery boot,
 then try to get to a virtual console and 
check with `lsmod|grep nvidia`. 
and to be safe, check `lsmod|grep nouveau`
You should never see the nouveau driver, this would be a nasty bug, please open an issue.

If the nvidia drivers are present:
then from the virtual terminal:

```bash
sudo systemctl stop lightdm
sudo rmmod nouveau #in case it is loaded
sudo rmmod nvidia_drm
sudo rmmod nvidia_modeset
sudo rmmod nvidia_uvm
sudo rmmod nvidia
sudo systemctl start lightdm
```
but you will have to work out why the nvidia-prime-boot.service did not do its job.


## Display manager doesn't start in nvidia mode?

You probably don't have the nvidia drivers installed in your kernel image, which can happen even if think you have the nvidia modules installed, because the standard 18.04 optimus logic uninstalls the drivers when you choose intel mode. We don't want that. 

Try ```sudo /usr/bin/prime-select nvidia```. If it complains that you are already in nvidia mode, do ```sudo /usr/bin/prime-select intel``` and then ```sudo /usr/bin/prime-select nvidia```

## Still stuck?

An idea: 
turn off nvidia-prime-boot.service
`systemctl disable nvidia-prime-boot.service`

swap to a virtual terminal (eg ctrl-alt F4)

run `sudo /usr/local/bin/prime_socket`

now go back to your GUI session, or some other virtual terminal, and do 
`prime-select intel`
and see what you see in the prime_socket VT

# How does it work?

It uses a modified version of prime-select.

The modified version is installed into /usr/local/bin which comes first in the standard path, so it masks the version of the nvidia-prime package

This version uses bbswitch to disable the nvidia card, which was the standard Ubuntu method until 18.04

There are virtually no reports of bbswitch not working in ubuntu 18.04 and there are many reports of the new way not working. 

The script calls a background service which kills lightdm, takes a few steps to change state, and restarts lightdm. Killing the display manager is necessary to remove the nvidia drivers.

The steps to change state:

* create or delete an xorg config file, 

* and remove or add the nvidia drivers to the running kernel. 

* and remove or add the nvidia drivers to the running kernel. It never adds nvidia drivers which are missing; it assumes they are always in a booting-kernel, and unloads them & turns off the card if you are in intel mode. Therefore, it doesn't need to do much at all if you want nvidia mode; nvidia mode is basically the default situation. 

The nvidia drivers are always present in the kernel image when you start the machine as a consequence of the standard ubuntu install of the nvidia drivers as long as you have not removed them by standard prime-select intel

The rust code prepares the state change. 
 nvidia-prime-boot.service is what removes the nvidia drivers and powers off the card; it obviously only runs if you selected intel mode.

# How is this different to the standard 18.04 approach?

Two things are different. Firstly, nvidia-prime in Ubuntu 18.04 does not use bbswitch to power-off the nvidia card when you are in intel-only mode. Instead, the developers swapped to an officially-supported kernel feature, which apparently only works when the nouveau driver is present. 
Unfortunately, this means the nvidia drivers have to be removed. So prime-select intel goes through an elaborate process of removing the nvidia drivers, rebuilding the initramfs image and rebooting, solely to load nouveau so the nvidia card can be turned off. 

Swapping back to nvidia then requires the basically the same process to repeat, except this time the nvidia modules are re-added to the kernel image. 

It is a very time consuming approach, mandating a reboot. Also, quite a few users have trouble getting the nouveau-power-off to work, so for those users it is slow, intrusive and broken.

bbswitch is not officially in the kernel. However, it is well used in just about all other distribution; it has a significant user-base, and there is no sign that it will stop working. Right now, it works with kernel 4.17 even as Ubuntu 18.04 is based on 4.15.
