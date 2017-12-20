### Case 01990657
### Issue
* The customer wants to know how the OpenStack handles the situation when the MAC address is duplicate. Their understanding is, there is a chance when two MAC addresses can be duplicate in a single network. And they pointed me the following code:
~~~
/usr/lib/python2.7/site-packages/neutron/db/db_base_plugin_v2.py

    def _create_db_port_obj(self, context, port_data):
        mac_address = port_data.pop('mac_address', None)
        if mac_address:
            if self._is_mac_in_use(context, port_data['network_id'],
                                   mac_address):
                raise exc.MacAddressInUse(net_id=port_data['network_id'],
                                          mac=mac_address)
        else:
            mac_address = self._generate_mac()
        db_port = models_v2.Port(mac_address=mac_address, **port_data)
        context.session.add(db_port)
        return db_port
~~~
* The said if the mac_address is specified, then there will be a check but if mac_address is not specified then there will be no check for whether the MAC address has been occupied. Is this true ?
### Investigation
* We need to take a look at the _generate_mac() function.
~~~
/usr/lib/python2.7/site-packages/neutron/db/db_base_plugin_common.py

 82     def _generate_mac():
 83         return utils.get_random_mac(cfg.CONF.base_mac.split(':'))
~~~
~~~
/usr/lib/python2.7/site-packages/neutron/common/utils.py

239 def get_random_mac(base_mac):
240     mac = [int(base_mac[0], 16), int(base_mac[1], 16),
241            int(base_mac[2], 16), random.randint(0x00, 0xff),
242            random.randint(0x00, 0xff), random.randint(0x00, 0xff)]
243     if base_mac[3] != '00':
244         mac[3] = int(base_mac[3], 16)
245     return ':'.join(["%02x" % x for x in mac])
~~~
* We need to simulate the situation when MAC address is requested by making some changes against the get_random_mac. Currently the scale is too large for us.
~~~
def get_random_mac(base_mac):
    mac = [int(base_mac[0], 16), int(base_mac[1], 16),
           int(base_mac[2], 16), int(base_mac[3], 16),
           int(base_mac[4], 16), random.randint(0x01, 0x03)]
    if base_mac[3] != '00':
        mac[3] = int(base_mac[3], 16)
    return ':'.join(["%02x" % x for x in mac])
~~~
* So we fix the first 11 bits and only leave 3 available MAC address for the last bit (which are 1, 2 and 3.)
* Remove the pyc and pyo file and then restart the services. Try to boot 3 instances. In theory, when booting the 3rd instance, there will be a 67% chance when the get_random_mac returned a duplicate MAC address. After this let's take a look at the neutron/server.log

~~~
DBDuplicateEntry: (pymysql.err.IntegrityError) (1062, u"Duplicate entry '6df9569d-dfbd-4fe5-86d6-fd40a9b2919d-fa:16:3e:12:34:01' for key 'uniq_ports0network_id0mac_address'") [SQL: u'INSERT INTO ports (project_id, id, name, network_id, mac_address, admin_state_up, status, device_id, device_owner, ip_allocation, standard_attr_id) VALUES (%(project_id)s, %(id)s, %(name)s, %(network_id)s, %(mac_address)s, %(admin_state_up)s, %(status)s, %(device_id)s, %(device_owner)s, %(ip_allocation)s, %(standard_attr_id)s)'] [parameters: {'status': 'DOWN', 'name': '', 'admin_state_up': 1, 'network_id': u'6df9569d-dfbd-4fe5-86d6-fd40a9b2919d', 'id': '0c042519-320c-426b-948b-a9b52634111d', 'device_owner': '', 'mac_address': 'fa:16:3e:12:34:01', 'standard_attr_id': 36, 'project_id': u'7bac97aa85d8443191f5f6529837cdf8', 'ip_allocation': None, 'device_id': ''}]
 wrapped /usr/lib/python2.7/site-packages/neutron/db/api.py:124
~~~
* The above means, the INSERT SQL command failed due to some errors. See the CONSTAINTS of the `ports` table.
~~~
| ports | CREATE TABLE `ports` (
  `project_id` varchar(255) DEFAULT NULL,
  `id` varchar(36) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `network_id` varchar(36) NOT NULL,
  `mac_address` varchar(32) NOT NULL,
  `admin_state_up` tinyint(1) NOT NULL,
  `status` varchar(16) NOT NULL,
  `device_id` varchar(255) NOT NULL,
  `device_owner` varchar(255) NOT NULL,
  `standard_attr_id` bigint(20) NOT NULL,
  `ip_allocation` varchar(16) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_ports0network_id0mac_address` (`network_id`,`mac_address`),
  UNIQUE KEY `uniq_ports0standard_attr_id` (`standard_attr_id`),
  KEY `ix_ports_network_id_mac_address` (`network_id`,`mac_address`),
  KEY `ix_ports_network_id_device_owner` (`network_id`,`device_owner`),
  KEY `ix_ports_device_id` (`device_id`),
  KEY `ix_ports_project_id` (`project_id`),
  CONSTRAINT `ports_ibfk_1` FOREIGN KEY (`network_id`) REFERENCES `networks` (`id`),
  CONSTRAINT `ports_ibfk_2` FOREIGN KEY (`standard_attr_id`) REFERENCES `standardattributes` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
~~~ 
* For column `network_id` and `mac_address`, there is a UNIQUE Key Constraint and it means the `network_id-mac_address` must be unique. So in one single network it is not allowed to create a port whose MAC address has been occupied by other ports.
* But what's next ? We caught the exception but the instance creation didn't fail acutally.
~~~
2017-12-11 02:19:18.445 24287 DEBUG oslo_db.api [req-e780f3a8-b9cd-4b46-95c9-36fc8918b31e 17a9a9bec778456f87360a2f95416783 7bac97aa85d8443191f5f6529837cdf8 - - -] Performing DB retry for function neutron.plugins.ml2.plugin.create_port wrapper /usr/lib/python2.7/site-packages/oslo_db/api.py:153
~~~
* According to the above message, there is a DB retry mechanism. So the next question is, what is the retry number ?
~~~
/usr/lib/python2.7/site-packages/neutron/db/api.py

110     @_tag_retriables_as_unretriable
111     @_retry_db_errors
112     @six.wraps(f)
113     def wrapped(*args, **kwargs):
114         try:
115             # copy mutable args and kwargs to make retries safe. this doesn't
116             # prevent mutations of complex objects like the context or 'self'
117             dup_args = [_copy_if_lds(a) for a in args]
118             dup_kwargs = {k: _copy_if_lds(v) for k, v in kwargs.items()}
119             return f(*dup_args, **dup_kwargs)
120         except Exception as e:
121             with excutils.save_and_reraise_exception():
122                 if is_retriable(e):
123                     LOG.debug("Retry wrapper got retriable exception: %s",
124                               traceback.format_exc())
125     return wrapped
~~~
* The code came to is_retriable() and we go to oslo_db_api.wrap_db_retry class in line 67.
~~~
 54 def is_retriable(e):
 55     if getattr(e, '_RETRY_EXCEEDED', False):
 56         return False
 57     if _is_nested_instance(e, (db_exc.DBDeadlock, exc.StaleDataError,
 58                                db_exc.DBConnectionError,
 59                                db_exc.DBDuplicateEntry, db_exc.RetryRequest)):
 60         return True
 61     # looking savepoints mangled by deadlocks. see bug/1590298 for details.
 62     return _is_nested_instance(e, db_exc.DBError) and '1305' in str(e)
 63 
 64 is_deadlock = moves.moved_function(is_retriable, 'is_deadlock', __name__,
 65                                    message='use "is_retriable" instead',
 66                                    version='newton', removal_version='ocata')
 67 _retry_db_errors = oslo_db_api.wrap_db_retry(
 68     max_retries=MAX_RETRIES,
 69     retry_interval=0.1,
 70     inc_retry_interval=True,
 71     exception_checker=is_retriable
 72 )
~~~
* By checking at line 143, there is a `remaining` parameter which came from self.max_retries. And the default self.max_retries is 20. But if we print the remaining to the logs, then we will find out the max_retires is 10. line 152 LOG.debug("remaining is %s", remaining)
"
~~~
/usr/lib/python2.7/site-packages/neutron/db/api.py

 85 class wrap_db_retry(object):
112     def __init__(self, retry_interval=1, max_retries=20,
113                  inc_retry_interval=True,
114                  max_retry_interval=10, retry_on_disconnect=False,
115                  retry_on_deadlock=False, retry_on_request=False,
116                  exception_checker=lambda exc: False):
131     def __call__(self, f):
132         @six.wraps(f)
133         def wrapper(*args, **kwargs):
134             next_interval = self.retry_interval
135             remaining = self.max_retries
136 
137             while True:
138                 try:
139                     return f(*args, **kwargs)
143                         if remaining > 0:
144                             ectxt.reraise = not expected
145                         else:
146                             if expected:
147                                 LOG.exception(_LE('DB exceeded retry limit.'))
148                             # if it's a RetryRequest, we need to unpack it
152                     LOG.debug("Performing DB retry for function %s",
153                               reflection.get_callable_name(f))
157                     time.sleep(next_interval)
158                     if self.inc_retry_interval:
159                         next_interval = min(
160                             next_interval * 2,
161                             self.max_retry_interval
162                         )
163                     remaining -= 1
165         return wrapper
~~~
* We have go back to the code when wrap_db_retry instance is created. 
~~~
 50 MAX_RETRIES = 10
 67 _retry_db_errors = oslo_db_api.wrap_db_retry(
 68     max_retries=MAX_RETRIES,
 69     retry_interval=0.1,
 70     inc_retry_interval=True,
 71     exception_checker=is_retriable
 72 )
~~~
* So it is clear that the retry number is 10, which is hardcoded.
### Conclusion: since we have a large amount of MAC address pool, it is very unlikely to have a duplicate MAC address when creating the port. But if we hit the chance, then the SQL command will fail and the creation will retry until the 10 times


