## Pdb
Add following line in  /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
~~~
   [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# diff -u /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py.orig  /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py -p
    --- /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py.orig    2017-10-23 04:07:07.076283417 +0900
    +++ /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py 2017-10-23 04:24:37.348444329 +0900
    @@ -811,6 +811,8 @@ class ISCSIConnector(base.BaseLinuxConne
             return (out, err)

         def _run_iscsiadm_bare(self, iscsi_command, **kwargs):
    +        #import rpdb2;rpdb2.start_embedded_debugger("pass")
    +        import pdb; pdb.set_trace()
             check_exit_code = kwargs.pop('check_exit_code', 0)
             (out, err) = self._execute('iscsiadm',
                                        *iscsi_command,
~~~
Run /usr/bin/nova-compute normally, and stop instance, then logout from iscsi, then start instance..
(Pdb) prompt will appear after while, when /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
 Is executed.
~~~
[root@el73-osp10-all-virbr1-gnocchi winpdb(keystone_project1-admin)]# /usr/bin/nova-compute
Option "rpc_backend" from group "DEFAULT" is deprecated for removal.  Its value may be silently ignored in the future.
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(816)_run_iscsiadm_bare()
-> check_exit_code = kwargs.pop('check_exit_code', 0)
(Pdb) list
811              return (out, err)
812      
813          def _run_iscsiadm_bare(self, iscsi_command, **kwargs):
814              #import rpdb2;rpdb2.start_embedded_debugger("pass")
815              import pdb; pdb.set_trace()
816  ->            check_exit_code = kwargs.pop('check_exit_code', 0)
817              (out, err) = self._execute('iscsiadm',
818                                         *iscsi_command,
819                                         run_as_root=True,
820                                         root_helper=self._root_helper,
821                                         check_exit_code=check_exit_code)
(Pdb) n
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(817)_run_iscsiadm_bare()
-> (out, err) = self._execute('iscsiadm',
(Pdb) 
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(819)_run_iscsiadm_bare()
-> run_as_root=True,
(Pdb) 
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(820)_run_iscsiadm_bare()
-> root_helper=self._root_helper,
(Pdb) 
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(821)_run_iscsiadm_bare()
-> check_exit_code=check_exit_code)
(Pdb) 
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(822)_run_iscsiadm_bare()
-> LOG.debug("iscsiadm %(iscsi_command)s: stdout=%(out)s stderr=%(err)s",
(Pdb) 
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(823)_run_iscsiadm_bare()
-> {'iscsi_command': iscsi_command, 'out': out, 'err': err})
(Pdb) p iscsi_command
['-m', 'session']
(Pdb) n
> /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py(824)_run_iscsiadm_bare()
-> return (out, err)
(Pdb) b 818
(Pdb) c
~~~
Check value with ‘list’ , ‘b(reakpoint) line#’ , ‘p(rint) $value’, and so ons.
## WinPDB
Download from https://drive.google.com/drive/u/0/folders/0B__MWvTGF3KqdDJWRmowM0JrMkU for winpdb and wxPython for RHEL/CentOS and install on RHEL7 packstack.
Then add following line in  /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py
~~~
   [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# vim /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# diff -u /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py.orig  /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py -p
    --- /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py.orig    2017-10-23 04:07:07.076283417 +0900
    +++ /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py 2017-10-23 04:08:19.193705435 +0900
    @@ -811,6 +811,7 @@ class ISCSIConnector(base.BaseLinuxConne
             return (out, err)

         def _run_iscsiadm_bare(self, iscsi_command, **kwargs):
    +        import rpdb2;rpdb2.start_embedded_debugger("pass")
             check_exit_code = kwargs.pop('check_exit_code', 0)
             (out, err) = self._execute('iscsiadm',
                                        *iscsi_command,

~~~
Run /usr/bin/nova-compute normally.

~~~
    [root@el73-osp10-all-virbr1-gnocchi winpdb(keystone_project1-admin)]# /usr/bin/nova-compute
    Option "rpc_backend" from group "DEFAULT" is deprecated for removal.  Its value may be silently ignored in the future.
    ...

~~~
Stop instance, then logout from iscsi, then start instance..
~~~
    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova stop 35747a83-46a2-42d3-9e3d-8f08008f7f18Request to stop server 35747a83-46a2-42d3-9e3d-8f08008f7f18 has been accepted.

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# iscsiadm -m node -T iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22 -p 192.168.123.110 --logout
    Logging out of session [sid: 3, target: iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22, portal: 192.168.123.110,3260]
    Logout of [sid: 3, target: iqn.2010-10.org.openstack:volume-8c67fc05-5178-44e1-8448-d7871dc31a22, portal: 192.168.123.110,3260] successful.

    [root@el73-osp10-all-virbr1-gnocchi ~(keystone_project1-admin)]# nova start 35747a83-46a2-42d3-9e3d-8f08008f7f18
    Request to start server 35747a83-46a2-42d3-9e3d-8f08008f7f18 has been accepted.
~~~

Run winpdb and click attach from menu, then enter password “pass” , then wait on starting  embedded debugger in /usr/lib/python2.7/site-packages/os_brick/initiator/connectors/iscsi.py

Click refresh till we can see debugee in the dialog box in GUI. Connect when appear.
~~~
    (Launch winpdb) $ winpdb
    (Enter pass and Click Refresh)
    (Connect)
~~~

Reference: http://winpdb.org/tutorial/WinpdbTutorial.html

## Reference:
https://chianingwang.blogspot.jp/2015/01/how-to-setup-eclipse-django-for-open.html?showComment=1491485387283#c6608459779620174299
https://stackoverflow.com/questions/543196/how-do-i-attach-a-remote-debugger-to-a-python-process
https://centos.pkgs.org/6/epel-x86_64/winpdb-1.4.8-2.el6.noarch.rpm.html
