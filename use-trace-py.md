# trace.py
Trace module is a useful module which could help us to trace the python code path.

For example, we want to dig down the code in iscsi attaching on cinder LVM iscsi driver with instance using cinder volume.

We can monitor the status as follows:
~~~
[root@el73-osp10-all-virbr1-gnocchi ~]# watch -n1 'targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm'
Every 1.0s: targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm                                                                                                                                          Mon Oct 23 04:39:09 2017

o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 1]
  | | o- iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22  [/dev/cinder-volumes/volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (1.0GiB) write-thru activated]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 ............................................. [TPGs: 1]
  |   o- tpg1 .......................................................................................... [no-gen-acls, auth per-acl]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.1994-05.com.redhat:7b6a95be53c9 ...................................................... [1-way auth, Mapped LUNs: 1]
  |     |   o- mapped_lun0 ................. [lun0 block/iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0  [block/iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (/dev/cinder-volumes/volume-8c67fc05-5178-44e1-8448-d7871dc31a22)]
  |     o- portals .................................................................................................... [Portals: 1]
  |      o- 192.168.123.110:3260 ............................................................................................. [OK]
  o- loopback ......................................................................................................... [Targets: 0]

tcp: [5] 192.168.123.110:3260,1 iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (non-flash)

11358 /usr/libexec/qemu-kvm -name guest=instance-0000001a,debug-threads=on -S -object secret,id=masterKey0,format=raw,file=/var/lib/libvirt/qemu/domain-9-instance-0000001a/master-key.aes -machine pc-i440fx-rhel7.3.0,accel=kvm,usb=off -m 5
12 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 35747a83-46a2-42d3-9e3d-8f08008f7f18 -smbios type=1,manufacturer=Red Hat,product=OpenStack Compute,version=14.0.3-9.el7ost,serial=23aeab88-1a9e-4086-8905-97519022ab9a,uuid=35
747a83-46a2-42d3-9e3d-8f08008f7f18,family=Virtual Machine -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-9-instance-0000001a/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode
=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=/dev/disk/by-path/ip-192.168.123.110:3260-iscsi-iqn.2010-
10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22-lun-0,format=raw,if=none,id=drive-virtio-disk0,serial=8c67fc05-5178-44e1-8448-d7871dc31a22,cache=none,aio=native -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-v
irtio-disk0,id=virtio-disk0,bootindex=1 -netdev tap,fd=26,id=hostnet0,vhost=on,vhostfd=28 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:9e:6d:5a,bus=pci.0,addr=0x3 -add-fd set=2,fd=33 -chardev file,id=charserial0,path=/dev/f
dset/2,append=on -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 -device isa-serial,chardev=charserial1,id=serial1 -device usb-tablet,id=input0,bus=usb.0,port=1 -vnc 0.0.0.0:0 -k en-us -device cirrus-vga,id=v
ideo0,bus=pci.0,addr=0x2 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5 -msg timestamp=on
18515 watch -n1 targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm
18516 sh -c targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm
19369 watch -n1 targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm
~~~
When start/stop the VM,
~~~
    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova start 35747a83-46a2-42d3-9e3d-8f08008f7f18
    Request to start server 35747a83-46a2-42d3-9e3d-8f08008f7f18 has been accepted.

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova stop 35747a83-46a2-42d3-9e3d-8f08008f7f18
    Request to stop server 35747a83-46a2-42d3-9e3d-8f08008f7f18 has been accepted.
~~~
But still logging into iSCSI target.
~~~
    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m node
    192.168.123.110:3260,-1 iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m session
    tcp: [1] 192.168.123.110:3260,1 iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (non-flash)
~~~

Logging out manually to observe what’s going on when instance is starting.
~~~
    19369 watch -n1 targetcli ls /;echo;iscsiadm -m session;echo;pgrep -fla qemu-kvm
    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m node -T iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 -p 192.168.123.110 --logout

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m session
    iscsiadm: No active sessions.

~~~
With python trace module, stop openstack-nova-compute.service, then run following script to check what script is executed while starting.
~~~
    [root@el73-osp10-all-virbr1-gnocchi winpdb(keystone_project1-admin)]# cat nova-compute.sh
    /usr/bin/python -m trace -t --ignore-dir=/usr/lib64/python2.7:/usr/lib/python2.7/site-packages/nova/api/validation:/usr/lib/python2.7/site-packages/oslo_config:/usr/lib/python2.7/site-packages/eventlet:/usr/lib/python2.7/site-packages/requests/packages/urllib3/packages:/usr/lib/python2.7/site-packages/nova/objects:/usr/lib/python2.7/site-packages/oslo_log:/usr/lib/python2.7/site-packages/enum --ignore-module=os,pkg_resources,six /usr/bin/nova-compute

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# ./nova-compute.sh | egrep --color=auto '^/.*(cinder|iscsi|libvirt)' | cut -d\( -f1 | uniq -c

    ...

          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
          4 /usr/lib/python2.7/site-packages/nova/virt/libvirt/utils.py
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
         10 /usr/lib/python2.7/site-packages/nova/virt/libvirt/config.py
          3 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/config.py
         31 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/iscsi.py
         12 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
         21 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/base_iscsi.py
        158 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
          8 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/base_iscsi.py
         49 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/iscsi.py
          6 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/iscsi.py
          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/volume.py
         40 /usr/lib/python2.7/site-packages/nova/virt/libvirt/config.py
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/volume.py
         11 /usr/lib/python2.7/site-packages/nova/virt/libvirt/host.py
          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/volume.py
          3 /usr/lib/python2.7/site-packages/nova/virt/libvirt/utils.py
         18 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/volume.py
          4 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/iscsi.py
         17 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/config.py
          5 /usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py
         12 /usr/lib/python2.7/site-packages/nova/virt/libvirt/vif.py
         32 /usr/lib/python2.7/site-packages/nova/virt/libvirt/config.py
         49 /usr/lib/python2.7/site-packages/nova/virt/libvirt/vif.py
          5 /usr/lib/python2.7/site-packages/nova/virt/libvirt/designer.py
         12 /usr/lib/python2.7/site-packages/nova/virt/libvirt/vif.py
          4 /usr/lib/python2.7/site-packages/nova/virt/libvirt/designer.py
          1 /usr/lib/python2.7/site-packages/nova/virt/libvirt/vif.py
~~~
Need following modification to get full path of executed python script.
~~~
[root@el73-osp10-all-virbr1-gnocchi winpdb(keystone_project1-admin)]# diff -u /usr/lib64/python2.7/trace.py.orig /usr/lib64/python2.7/trace.py
--- /usr/lib64/python2.7/trace.py.orig  2017-11-08 08:05:27.527073555 -0500
+++ /usr/lib64/python2.7/trace.py	2017-11-08 08:06:15.340276781 -0500
@@ -188,7 +188,7 @@
 
     base = os.path.basename(path)
     filename, ext = os.path.splitext(base)
-    return filename
+    return path
 
 def fullmodname(path):
     """Return a plausible module name for the path."""
@@ -621,7 +621,7 @@
             if self.start_time:
                 print '%.2f' % (time.time() - self.start_time),
             bname = os.path.basename(filename)
-            print "%s(%d): %s" % (bname, lineno,
+            print "%s(%d): %s" % (filename, lineno,
                                   linecache.getline(filename, lineno)),
         return self.localtrace
 
@@ -634,7 +634,7 @@
             if self.start_time:
                 print '%.2f' % (time.time() - self.start_time),
             bname = os.path.basename(filename)
-            print "%s(%d): %s" % (bname, lineno,
+            print "%s(%d): %s" % (filename, lineno,
                                   linecache.getline(filename, lineno)),
         return self.localtrace

~~~
In case we want to check one of following scripts from trace output,
~~~
          2 /usr/lib/python2.7/site-packages/nova/virt/libvirt/volume/iscsi.py
         12 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py   ⇐========================
         21 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/base_iscsi.py
        158 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
          8 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/base_iscsi.py
         49 /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
~~~
Run script again and get detailed info from output as follows,
~~~
    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova list
    +--------------------------------------+------------------------------+--------+------------+-------------+---------------+
    | ID                                   | Name                         | Status | Task State | Power State | Networks      |
    +--------------------------------------+------------------------------+--------+------------+-------------+---------------+
    | 35747a83-46a2-42d3-9e3d-8f08008f7f18 | cirros-0.3.4-x86_64-disk.vol | ACTIVE | -          | Running     | net1=10.5.5.6 |
    +--------------------------------------+------------------------------+--------+------------+-------------+---------------+

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova stop 35747a83-46a2-42d3-9e3d-8f08008f7f18
    Request to stop server 35747a83-46a2-42d3-9e3d-8f08008f7f18 has been accepted.

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m session
    tcp: [2] 192.168.123.110:3260,1 iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 (non-flash)

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m node -T iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 -p 192.168.123.110 --logout
    Logging out of session [sid: 2, target: iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22, portal: 192.168.123.110,3260]
    Logout of [sid: 2, target: iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22, portal: 192.168.123.110,3260] successful.

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m session
    iscsiadm: No active sessions.


    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# ./nova-compute.sh | egrep --color=auto '^/usr/lib/python2.7/site-packages/os_brick/'
    ...

    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(820):         LOG.debug("iscsiadm %(iscsi_command)s: stdout=%(out)s stderr=%(err)s",
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(821):                   {'iscsi_command': iscsi_command, 'out': out, 'err': err})
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(822):         return (out, err)
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(712):         portals = [{'portal': p.split(" ")[2], 'iqn': p.split(" ")[3]}
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(713):                    for p in out.splitlines() if p.startswith("tcp:")]
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(715):         stripped_portal = connection_properties['target_portal'].split(",")[0]
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(716):         if len(portals) == 0 or len([s for s in portals
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(723):             try:
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(724):                 self._run_iscsiadm(connection_properties,
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(725):                                    ("--login",),
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(726):                                    check_exit_code=[0, 255])
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(578):         check_exit_code = kwargs.pop('check_exit_code', 0)
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(579):         attempts = kwargs.pop('attempts', 1)
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(580):         delay_on_retry = kwargs.pop('delay_on_retry', True)
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(581):         (out, err) = self._execute('iscsiadm', '-m', 'node', '-T',
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(582):                                    connection_properties['target_iqn'],
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(583):                                    '-p',
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(584):                                    connection_properties['target_portal'],
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(585):                                    *iscsi_command, run_as_root=True,
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(586):                                    root_helper=self._root_helper,
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(587):                                    check_exit_code=check_exit_code,
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(588):                                    attempts=attempts,
    /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(589):                                    delay_on_retry=delay_on_retry)
~~~
