---
layout: post
title:  "Moving RHEL JBoss A-MQ to faster disks"
date:   2017-03-13 09:53:00
comments: false
modified: 2017-03-13
---

JBoss A-MQ is RHEL's version of the Apache Active MQ service. It's a messaging queue designed to integrate applications. This service is classified as a "Fire and Forget" service in that it receives a message, which is consumed by a consumer, and then the message is forgotten about. No historical data is saved. Like Apache Active MQ, RHEL J-Boss A-MQ uses KahaDB for message organization. KahaDB is a file based persistence database that is local to the message broker that is using it. It has been optimized for fast persistence. As near as I can tell, KahaDB is developed specifically for messaging services (hence the performance boost, and why I can't find it anywhere else).

## Underlying Drive Configuration
I read more than one blog about people getting down into the micro details of performance tweaking and optimizing (FS Optimization and benchmarking IOPs etc.). I didn't get that carried away, there was no need. I focused ont the macro changes we could make to increase the disk performance of this service as it is was a vital piece of our architecture.

We started with a single drive with RHEL installed and AMQ running under a home profile. Our performance team located a bottle neck in disk utilization, so we made a change; We attached four HDDs, striped and mounted them under /data, then ran the amq service from this location. To anyone not familiar with this process, it might sound very complicated - it's not!

LVM (which comes standard with RHEL) makes disk striping and other activites in the like, very simple. The most confusing concept to me, when getting starting with LVM was getting used to the abstract architecture that it operates on. Enter a helpful picture:

![LVMArchitecture](/images/LVMArchitecture.png)

So, attach disks, create physical volumes on those disks, create a volume group made of those physical volumes, then create a logical volume on the volume group, put a file system on that volume, then mount it. I won’t cover the attaching of disks, that depends on how (if) you’re virtualizing your VM. After you disks are attached, take a look at their device names …

![LVM-lsblk](/images/LVM-1.PNG)

In the example above, you can see three 5G disks named sdb, sdc, sdd, and sde. Those are the disks I just attached to this example VM and want to stripe. Now let’s make an LVM Volume Group from those disks 

![LVM-CreatePhysicalVolumes](/images/LVM-2.PNG)

Now we have four LVM Physical Volumes. Let’s create a volume group named “data_volume_group” from those physical volumes. 

![LVM-CreateVolumeGroup](/images/LVM-3.PNG)

Almost there, next let’s put a Logical Volume called “data_volume” on that Volume Group. Since we plan to only put one LV (Logical Volume) on that VG (Volume Group), we’ll use the entire capacity of the volume group “—extents 100%FREE”. We’ll make sure we create one stripe per physical disk to ensure we’re getting max io from this logical volume (link to further stripe understanding at bottom of page). 

![LVM-CreateLogicalVolume](/images/LVM-4.PNG)

Lastly we need to put a filesystem on this logical volume, and mount it. 

![LVM-CreateFSandMount](/images/LVM-5.PNG)

Our fstab entry to ensure the volume is mounted after restarts … 

![LVM-FStab](/images/LVM-6.PNG)

Lastly, run mount '/data' to mount your ext4 filesystem on the logical volume you created “data_volume”

## Moving RHEL J-Boss A-MQ
This part of the process in practice was very easy - though I had trouble finding helpful documentation on how to do this. So there was much research involved to minimize downtime. 
The RHEL J-Boss A-MQ service is self-contained – that is, it uses relative paths instead of absolute paths. Which means we can simply stop the process, delete the perishable kahadb data (remember there is no value is having historical data for this service), move the binaries and queue xml files, then start the process in its new location. Our instance did not have any auto-start upon restart functionality, we had to start it manually. I imagine that if someone did have their binary written in some auto init script, that would of course need to be updated to point to the new amq binary. 
The psudocode looked like this …
{% highlight bash %}
cd /old/location/jboss-a-mq/
./bin/stop
rm -rf ./data/amq/kahadb/*
mkdir /data/amq
cp -ra ./* /data/amq/
/data/amq/bin/start 
{% endhighlight %}

Done and done. The A-MQ service (and hawtio for web management) is now running from your striped disk. 

## Sources/Additional Reading
[RHEL's Documentation for LVM Architecture](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/LVM_definition.html)

[Ubuntu's Linux FSTAB Documentation](https://help.ubuntu.com/community/Fstab)

[RHEL J-Boss A-MQ](https://www.redhat.com/en/technologies/jboss-middleware/amq)
