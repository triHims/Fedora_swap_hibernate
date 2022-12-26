# Fedora enable hibernate and then suspend then hibernate

Fedora 36 ships with zram.
I want to use hibernation and  8 gb zram is too low for me since i use sts intellij and vscode. 

My system has 8gb ram , 4 core processor and a ssd(512) 


From the [article](https://j3ff.org/post/framework_fedora_hibernate/) 

### Enable SwapFile
Enable a swap of size 20gb (it should be equal or greater than ram+zram). 
    
    since my system has btrfs we will follow procedure for it.


    
    sudo btrfs subvolume create /swap
    sudo touch /swap/swapfile
    sudo chattr +C /swap/swapfile
    sudo fallocate --length 33GiB /swap/swapfile
    sudo chmod 600 /swap/swapfile
    sudo mkswap /swap/swapfile
    
#### Check the swapfile 
enable swap by `swapon /swap/swapfile`
check by `cat /proc/swaps`


```bash
Filename				Type		Size		Used	Priority
/swap/swapfile                          file		16777212	0	-2
/dev/zram0                              partition	7926780		0	100

```

Add the swapfile entry to end fstab (to enable auto attachment on every boot )

```sh
#open fstab as 
vi /etc/fstab
```
Add the entry
```
/swap/swapfile   swap    swap   defaults 0 0
```
Now on reboot your swap should be visible , check which the command given above
### Configure Dracut

`echo 'add_dracutmodules+=" resume "' | sudo tee /etc/dracut.conf.d/resume.conf`


`sudo dracut -f`


### Configure Grub 

This is where i struggled as system was giving error:  ` writer not found`.

We need to instruct system to boot from swapfile. (In case of BTRFS not possible before kernel 5.0 )

So for booting kernel needs proper params for pointing to partition and offset.

get UUID of partition containing the swapfile.

```
sudo findmnt -no UUID -T /swap/swapfile
```
get the physical offset for resume offset parameter

follow this [article](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file_on_Btrfs)

in my case (only for btrfs filesystem) 
```sh
gcc -O2 -o btrfs_map_physical getBtrfsOffset.c
./btrfs_map_physical /swap/swapfile | head -n2 | tail -n1 | awk '{print $9}'

```
Output was : 21660442624
This is the offset of the file

Now do 
```
getconf PAGESIZE
```
and get the page size . In my case it was 4096 
so the resume_offset is offset / pageSize.

> result = 5288194


Use grubby to append these parameters to kernel boot command


```
grubby --update-kernel=ALL --args="resume=UUID=efe08eeb-6757-4eb9-8097-51920d6368ae resume_offset=57093922816"
```

### Configure SELinux

Lastly we need to configure SELinux to allow us to hibernate and resume.

```sh
sudo audit2allow -a -M systemd_sleep
sudo semodule -i systemd_sleep.pp
sudo audit2allow -a -M fprintd_t
sudo semodule -i fprintd_t.pp
```

try to suspend using '
> systemctl hibernate

It should work. 

### Enable suspend then hybernate

any systemd service can by overriden by
```
systemctl edit service
```
and a service can be viewed by 
> systemctl cat service

On any error a service can be reset by 
> systemctl revert service

So just view the suspend then hibernate service 
and the copy it.
use `systemctl cat` to override suspend.service and pasted the copied suspend-then-hibernate.unit



### Misc 
To test since kernel resume parameters only work after reboot. To check hibernate right away.

```
# echo 251:5 > /sys/power/resume
# echo 5288194 > /sys/power/resume_offset
# echo reboot > /sys/power/disk
# echo disk > /sys/power/state
```

where resume is passed major:minor numbers from lsblk indicating partition which contains our swapfile.

and offset is the calculated offset


## Disable zram and move to zswap 

I have now decided to use zswap compeletly I believe it is better overall.

Disable zram on fedora
> sudo touch /etc/systemd/zram-generator.conf

Enable zswap 
> grubby --update-kernel=ALL --args="zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=20 zswap.zpool=z3fold"


Reboot , try systemctl hibernate if it fails then reconfigure selinux and enjoy


### MISC

View pool stats 
> grep -r . /sys/kernel/debug/zswap

Debug service 

> sudo journalctl -u systemd-hibernate.service
