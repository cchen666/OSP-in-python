#### This example is provided by Martin @ Red Hat
#### Issue
With `preallocate_images = space` set in `nova.conf` the `disk_over_committed` calculation is wrong because in the end the size of the ephemeral disk is deducted twice when checking the available resources. As a result `available_least` which is used for the disk filter shows no available disk, even if there is.

E.g. creating a single instance with 20GB ephemeral disk on a compute with ~33GB available:

* `preallocate_images = space`

~~~
2018-03-08 11:46:54.444 12793 DEBUG nova.virt.libvirt.driver [req-e41619b7-f10d-407f-bc71-95d7e4fe610f - - - - -] disk_free_gb 13 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5503
...
2018-03-08 11:46:54.674 12793 DEBUG nova.virt.libvirt.driver [req-e41619b7-f10d-407f-bc71-95d7e4fe610f - - - - -] DISK info 21474573824 _get_disk_over_committed_size_total /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:7111
2018-03-08 11:46:54.674 12793 DEBUG nova.virt.libvirt.driver [req-e41619b7-f10d-407f-bc71-95d7e4fe610f - - - - -] disk_over_committed out 21474573824 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5505
2018-03-08 11:46:54.675 12793 DEBUG nova.virt.libvirt.driver [req-e41619b7-f10d-407f-bc71-95d7e4fe610f - - - - -] available_least -7515930112 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5507
~~~

* `preallocate_images = none`

~~~
2018-03-08 11:48:35.348 13987 DEBUG nova.virt.libvirt.driver [req-fff3e2a9-c70b-4e62-8f33-c761f3337930 - - - - -] disk_free_gb 33 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5503
...
2018-03-08 11:48:35.581 13987 DEBUG nova.virt.libvirt.driver [req-fff3e2a9-c70b-4e62-8f33-c761f3337930 - - - - -] DISK info 21474573824 _get_disk_over_committed_size_total /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:7111
2018-03-08 11:48:35.582 13987 DEBUG nova.virt.libvirt.driver [req-fff3e2a9-c70b-4e62-8f33-c761f3337930 - - - - -] disk_over_committed out 21474573824 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5505
2018-03-08 11:48:35.582 13987 DEBUG nova.virt.libvirt.driver [req-fff3e2a9-c70b-4e62-8f33-c761f3337930 - - - - -] available_least 13958906368 get_available_resource /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py:5507
~~~

#### Steps to Reproduce:
1. set `preallocate_images = space` on compute
2. check `disk_available_least stats`
3. create instance 
4. verify `disk_available_least again`

#### Additional info:

~~~
   5463     def get_available_resource(self, nodename):
   5464         """Retrieve resource information.
   5465 
   5466         This method is called when nova-compute launches, and
   5467         as part of a periodic task that records the results in the DB.
   5468 
   5469         :param nodename: unused in this driver
   5470         :returns: dictionary containing resource info
   5471         """
   5472 
   5473         disk_info_dict = self._get_local_gb_info()  <----- here we get the local available disk space AFTER the qcow file got created so "free from before instance start - full image size", in my example 33 - 20 = 13
   5474         data = {}
...
   5502         disk_free_gb = disk_info_dict['free']       <---- this is our remaining 13GB free
   5503         disk_over_committed = self._get_disk_over_committed_size_total() <---- returns 20GB even if overcommit should be 0 for pre-allocated image because disk_over_committed should be the sum of not yet allocated space of all instance disks configured/requested.
   5504         available_least = disk_free_gb * units.Gi - disk_over_committed  <---- with this bug we reduce here again the disk_free_gb by the amount of disk_over_committed
   5505         data['disk_available_least'] = available_least / units.Gi
~~~

Here is the issue - `virt/libvirt/driver.py` in `_get_instance_disk_info`:

~~~
   6935     def _get_instance_disk_info(self, instance_name, xml,
   6936                                 block_device_info=None):
...
   7008                 else:
   7009                     dk_size = int(os.path.getsize(path))          <----- 'preallocated = space' uses falloc which is not shown in getsize(). we do not use the info from qemu-img info on the actual size. we check the disk size on the device, which is not 20GB
   7011             elif disk_type == 'block' and block_device_info:
   7012                 dk_size = lvm.get_volume_size(path)
   7013             else:
   7014                 LOG.debug('skipping disk %(path)s (%(target)s) - unable to '
   7015                           'determine if volume',
   7016                           {'path': path, 'target': target})
   7017                 continue
   7018 
   7019             disk_type = driver_nodes[cnt].get('type')
   7020 
   7021             if disk_type in ("qcow2", "ploop"):
   7022                 backing_file = libvirt_utils.get_disk_backing_file(path)
   7023                 virt_size = disk_api.get_disk_size(path)
   7025                 over_commit_size = int(virt_size) - dk_size          <----- with the above, the over_commit_size is still 20GB even if the file is pre allocated.
~~~

For reference `get_disk_size` to check the `virt_size` of the qcow2 file - `/usr/lib/python2.7/site-packages/nova/virt/disk/api.py`:

~~~
    141 def get_disk_size(path):
    142     """Get the (virtual) size of a disk image
    143 
    144     :param path: Path to the disk image
    145     :returns: Size (in bytes) of the given disk image as it would be seen
    146               by a virtual machine.
    147     """
    148     return images.qemu_img_info(path).virtual_size
~~~

* `/usr/lib/python2.7/site-packages/nova/virt/libvirt/imagebackend.py`

~~~
    235         if size:
    236             # create_image() only creates the base image if needed, so
    237             # we cannot rely on it to exist here
    238             if os.path.exists(base) and size > self.get_disk_size(base):
    239                 self.resize_image(size)
    240 
    241             if (self.preallocate and self._can_fallocate() and
    242                     os.access(self.path, os.W_OK)):
    243                 utils.execute('fallocate', '-n', '-l', size, self.path)   <----- here we preallocate the space using fallocate
~~~

E.g. a ~1MB file

~~~
# fallocate -n -l 1000000 /tmp/test
# ll /tmp/test 
-rw-r--r--. 1 root root 0 Mar  9 11:06 /tmp/test
# du -mshc /tmp/test 
980K    /tmp/test

# python
Python 2.7.5 (default, Aug  2 2016, 04:20:16)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-4)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.path.getsize("/tmp/test")
0
~~~

It works if we introduce - `virt/disk/api.py`:

~~~
    151 def get_phy_disk_size(path):
    152     """Get the (virtual) size of a disk image
    153 
    154     :param path: Path to the disk image
    155     :returns: Size (in bytes) of the given disk image as it would be seen
    156               by a virtual machine.
    157     """
    158     return images.qemu_img_info(path).disk_size
~~~

and change `dk_size` to be get in `virt/libvirt/driver.py` using `get_phy_disk_size instead` of `int(os.path.getsize(path))`

~~~
   6935     def _get_instance_disk_info(self, instance_name, xml,
   6936                                 block_device_info=None):
...
   7008                 else:
   7009                     #dk_size = int(os.path.getsize(path))
   7009                     dk_size = disk_api.get_phy_disk_size(path)
...
   7021             if disk_type in ("qcow2", "ploop"):
   7022                 backing_file = libvirt_utils.get_disk_backing_file(path)
   7023                 virt_size = disk_api.get_disk_size(path)
   7024                 over_commit_size = int(virt_size) - dk_size
   7025             else:
~~~
#### Root Cause

`os.path.getsize(path)` does note reflect if an instance disk space is preallocated using `fallocate -n`. However if we remove `-n` parameter then both `fallocate` and `ls` can reflect the actual size of the file so that the getsize() can get the actual size of the preallocated file.
