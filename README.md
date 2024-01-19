# LVM-Creation

You can check the belwo post for understandin gof what is LVM? 
[Linkedin POst](https://www.linkedin.com/feed/update/urn:li:activity:7153739876494966785/)

# To set up a new Logical Volume, we need to proceed in the following order:

Create physical volume or volumes from the existing hard drives.
Create a Volume group and add the physical volumes to it.
Create a Logical Volume from the Volume Group.
Format the Logical Volume as required — xfs, ext4 etc.
Finally, mount the new filesystem.

```
sudo fdisk -l
```
This command will show all the disks in the system along with the devices attached that are not at mounted.
Let's assume you monted a device which the above command shows as /dev/sdb

We can see we have /dev/sdb which is raw disk, it is not yet available and cannot be used, so we need to format it.

Let’s look at this quick demo on how we can make this new disk /dev/sdb available:

```
fdisk /dev/sdb
>n
>p (primary)
> partition number(1-4): 1
>First sector (2048- 62914559, default 2048) :2048
> Last Sector (2048-62914559) default 62914559: (WE PRESS ENTER HERE)
>t
>Enter Hexcode: 8e
```

A few things about what we have done:

We are making this raw new drive /dev/sdb available by creating a new partition and assigning the proper label to it.
We have not specified a specific size for this partition but chose the default which is to use the whole disk, hitting enter without specifying any size will allocate the whole space available.
The crucial part here is to choose “8e” Hex code which will label this new partition and assign it as an LVM type, if we wanted to create an ext4 partition then the label code would be different. For every new partition created, it has to be labelled before it can be used.
“w” is required to save the changes we have made.

Now that our disk is ready, let’s start by creating a physical volume with pvcreate:
```
sudo pvcreate /dev/sdb
```

Physical volume has created we can check this using pvs or pvdisplay
```
pvs
pvdisplay
```

We can see we already had a PV /dev/sda2 meaning the root partition is also built on top of a logical volume. The new disk /dev/sdb1 is now a physical volume. We can notice that the VG Name of this new physical volume is empty! that is because it has not been added to a volume group yet!
Let’s do that right now:

```
sudo vgcreate <name> /dev/sdb
```
we use vgcreate to create a new volume assining a name to this new volume group and adding physical volumes or volumes to it, in this case , we make /dev/sdb part of this VG.

This can be checked using vgs
```
vgs
```

Now, we create the logical volume with lvcreate:
```
sudo lvcreate --name <name> -l 100%FREE <vg name>
```
Here vg name is the name that you created previously
-l : is used to specify how much space we wish to take from the volume group, here we allocate 100% of it, we do need to mention the volume group name in our command.

This can be checked using lvdisplay
```
lvdisplay
```

Our Logical Volume has been created!

The final step we should take to be able to use this logical volume is to format this new LV, we do this with the help of the mkfs.xfs command:

```
sudo mkfs.xfs /dev/<vg name>/<lv name>
```

. mkfs stands for “make a filesystem”, which is what it does. In this case, we want to have an xfs filesystem hence mkfs.xfs. The xfs filesystem is an upgrade from ext4 in many aspects and is the default filesystem with RHEL servers, that said, there are advantages of using ext4 over xfs, depending on the situation.

We want to mount this newly created logical volume on a mount point, let’s create it and proceed:
```
sudo mkdir /<name>
sudo mount /dev/<vgname>/<lv name> /<directory name>
```

We created a brand new directory where our new filesystem will be mounted, anything under /data will belong to this new logical volume.

— As a side note — we do need to add any new mount point in the: /etc/fstab file so it persists throughout reboots.

# How to extend a Logical Volume
We have seen how to create a logical volume from scratch, but in most cases, you will need to increase the size of an already existing logical volume so it can accommodate more data.

Let’s say you have plugged in additional hard drives into your server and now need to make them part of an existing Logical Volume (grow an existing LV) Let’s jump right in and do this:

First and foremost, to extend an existing Logical Volume, we must have free space in the Volume Group.
This is the sequence we need to follow to extend a Logical Volume:

-Add new hard drives 
-create new physical volumes 
-Add the new PVs to the exiting Volume Group 
-Extend the Logical Volume.

First, we need to make this new drive usable by creating a new partition
```
fdisk /dev/sdc
>n
>p (primary)
> partition number(1-4): 1
>First sector (2048- 62914559, default 2048) :2048
> Last Sector (2048-62914559) default 62914559: (WE PRESS ENTER HERE)
>t
>Enter Hexcode: 8e
```
Using fdisk, we created a new partition, assigned the whole size by not specifying any size (when pressing enter without any value, it assigns whatever is available)
We need to label it properly, 8e is the label needed to make it the “LVM type”.

Create physical drive

```
pvcreate /dev/sdc
```

Now let's add it to the volume group (vg name) we can do this using the vgextend

```
vgextend <vg name> /dev/sdc
```

Now our volume has extended

We can extend the Logical volume(LV name) using the command lvextend

```
lvextend -l +100%FREE /dev/<vg name> /<lv name>
```
lvextend will extend the lv-data logical volume, the +100%FREE option means that the volume will be extended to all the remaining sizes available from the Volume Group.

We are not done yet! , one last step is to resize the filesystem so the newly added storage capacity is available, we do that with the xfs_growfs command:

```
xfs_growfs /dev/<vg name>/<lv name>
```
we can see that the data blocks have been changed, the filesystem has been extended.
The other great thing we should mention here is we did not even need to umount our mount point /data when growing the filesystem!







