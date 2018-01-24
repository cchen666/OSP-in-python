# Example 1 

### Issue: ERROR: 'unicode' object has no attribute 'get' when creating a heat template
~~~
# heat stack-create --template-file create_dual_fortinet_vmac.debug  test1  -P slave_fw_port=192.168.1.122  -P forti_groupid=1
WARNING (shell) "heat stack-create" is deprecated, please use "openstack stack create" instead
ERROR: 'unicode' object has no attribute 'get'
~~~
The following is the heat template which has been adjusted. The original heat template is very long. 
~~~
# cat create_dual_fortinet_vmac.debug
heat_template_version: 2016-10-14
description: A very basic Heat template.

parameters:

  slave_fw_port:
    type: string
    description: slave vm ip address (ask your cloudadmin)

  forti_groupid:
    type: string
    description: group id of firewall servers (ask your cloudadmin)

resources:
  provider2_port:
    type: OS::Neutron::Port
    properties:
      network_id: 116bcd56-2a4b-4f77-b83f-b87044401213
      admin_state_up: True
      port_security_enabled: True
      name : provider2_port
      fixed_ips:  { get_param: slave_fw_port }
     
      allowed_address_pairs:
        - ip_address:
            str_replace:
              template: "$sprov_port1/32"
              params:
                $sprov_port1: { get_param: slave_fw_port }
          mac_address:
            str_replace:
              template: "00:09:0f:09:$GID:00"
              params:
                 $GID: { get_param: forti_groupid }
~~~
### First let's take a look at the heat-engine.log:
~~~
<Snip>

2017-09-26 09:49:46.609 54394 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/heat/engine/translation.py", line 293, in _exec_replace
2017-09-26 09:49:46.609 54394 ERROR oslo_messaging.rpc.server     if translation_data and translation_data.get(translation_key):
2017-09-26 09:49:46.609 54394 ERROR oslo_messaging.rpc.server AttributeError: 'unicode' object has no attribute 'get'
~~~

### Let's follow the code file and line 293

/usr/lib/python2.7/site-packages/heat/engine/translation.py
~~~
290     def _exec_replace(self, translation_key, translation_data,
291                       value_key, value_data, value):
292         value_ind = None
293         if translation_data and translation_data.get(translation_key):
294             if value_data and value_data.get(value_key):
295                 value_ind = value_key
~~~

This get() method should come from a dictionary. So this error means, a string(unicode) was filled with the translation_data. So at first I'd like to print the problematic translation_data.

Don't forget to delete the compiled files if we decide to modify the python file.
~~~
# rm -rf /usr/lib/python2.7/site-packages/heat/engine/translation.pyc
# rm -rf /usr/lib/python2.7/site-packages/heat/engine/translation.pyo
~~~
Also backup the original files
~~~
# cp /usr/lib/python2.7/site-packages/heat/engine/translation.py /usr/lib/python2.7/site-packages/heat/engine/translation.py.bak
~~~
In order to get the translation_data, we need to print the value in the logs so we chose oslo_log module. In this example we use isinstance to see whether translation_data is a dictionary or not and if not print it to the logs. (Forgive me if this is really a simple one...)
~~~
# diff -u /usr/lib/python2.7/site-packages/heat/engine/translation.py.bak /usr/lib/python2.7/site-packages/heat/engine/translation.py
--- /usr/lib/python2.7/site-packages/heat/engine/translation.py.bak    2017-09-26 09:55:29.003812937 -0400
+++ /usr/lib/python2.7/site-packages/heat/engine/translation.py    2017-09-26 10:01:13.620490126 -0400
@@ -21,6 +21,9 @@
 from heat.engine import function
 from heat.engine.hot import functions as hot_funcs
 from heat.engine import properties
+from oslo_log import log as logging
+LOG = logging.getLogger(__name__)
+

 
 class TranslationRule(object):
@@ -290,6 +293,8 @@
     def _exec_replace(self, translation_key, translation_data,
                       value_key, value_data, value):
         value_ind = None
+        if not isinstance(translation_data, dict):
+            LOG.info("translation_data is %s",translation_data)
         if translation_data and translation_data.get(translation_key):
             if value_data and value_data.get(value_key):
                 value_ind = value_key
~~~

Restart heat-engine service. Reproduce the issue again and check the heat-engine.log
~~~
2017-09-26 10:01:22.178 58687 INFO heat.engine.service [req-0a52eccd-8c13-4e00-a283-1655fd9e0287 f2f28444995340d5ae182c1387d1348e 2c7c740188c44bc99ea1e565fe636eb9 - - -] Creating stack test1
2017-09-26 10:01:22.220 58687 INFO heat.engine.translation [req-0a52eccd-8c13-4e00-a283-1655fd9e0287 f2f28444995340d5ae182c1387d1348e 2c7c740188c44bc99ea1e565fe636eb9 - - -] translation_data is 192.168.1.122
~~~
Okay we got the translation_data and that is 192.168.1.122. That is the parameter of slave_fw_port. So we need to recheck the lines with slave_fw_port. 
~~~
      fixed_ips:  { get_param: slave_fw_port }
~~~
If we are familiar with the syntax of the heat or we search the fixed_ips syntax online, then we will find that the above is not correct and a valid fixed_ips is a list like following:
~~~
      fixed_ips:  
        - subnet_id: e28593f6-f102-4d8f-8a1b-6b0563af87bb
       ip_address: { get_param: slave_fw_port }
~~~
So the solution is to change the fixed_ips to a list. In this example, the customer's heat template is very long and hard to identify the problem. With the help of printing the translation_data, the problem can be isolated quickly.
