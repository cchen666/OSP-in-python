## Support Case 01934638
### Issue: gnocchi measures show \<id of disk.usage\> is always empty
~~~
$ gnocchi measures show 6a2ee37f-d034-4b4e-8036-70464ddea3bf | wc -l
0
~~~

The first thought should be checking the `ceilometer/compute.log` on compute side. Surprisingly, we don't see any disk.usage at all.
~~~
$ grep 'disk.usage' /var/log/ceilometer/compute.log | wc -l
0
~~~
* Then we check the logs in details and we found:
~~~
2017-12-01 09:09:15.868 28983 INFO ceilometer.pipeline [-] Config file: {'sources': [{'interval': 600, 'meters': ['cpu', 'memory.usage', 'network.incoming.bytes', 'network.incoming.packets', 'network.outgoing.bytes', 'network.outgoing.packets', 'disk.read.bytes', 'disk.read.requests', 'disk.write.bytes', 'disk.write.requests', 'hardware.cpu.util', 'hardware.memory.used', 'hardware.memory.total', 'hardware.memory.buffer', 'hardware.memory.cached', 'hardware.memory.swap.avail', 'hardware.memory.swap.total', 'hardware.system_stats.io.outgoing.blocks', 'hardware.system_stats.io.incoming.blocks', 'hardware.network.ip.incoming.datagrams', 'hardware.network.ip.outgoing.datagrams'], 'name': 'some_pollsters', 'sinks': ['meter_sink']}, {'interval': 600, 'meters': ['cpu'], 'name': 'cpu_source', 'sinks': ['cpu_sink', 'cpu_delta_sink']}, {'interval': 600, 'meters': ['disk.read.bytes', 'disk.read.requests', 'disk.write.bytes', 'disk.write.requests', 'disk.device.read.bytes', 'disk.device.read.requests', 'disk.device.write.bytes', 'disk.device.write.requests'], 'name': 'disk_source', 'sinks': ['disk_sink']}, {'interval': 600, 'meters': ['network.incoming.bytes', 'network.incoming.packets', 'network.outgoing.bytes', 'network.outgoing.packets'], 'name': 'network_source', 'sinks': ['network_sink']}], 'sinks': [{'publishers': ['notifier://'], 'transformers': None, 'name': 'meter_sink'}, {'publishers': ['notifier://'], 'transformers': [{'name': 'rate_of_change', 'parameters': {'target': {'max': 100, 'scale': '100.0 / (10**9 * (resource_metadata.cpu_number or 1))', 'type': 'gauge', 'name': 'cpu_util', 'unit': '%'}}}], 'name': 'cpu_sink'}, {'publishers': ['notifier://'], 'transformers': [{'name': 'delta', 'parameters': {'target': {'name': 'cpu.delta'}, 'growth_only': True}}], 'name': 'cpu_delta_sink'}, {'publishers': ['notifier://'], 'transformers': [{'name': 'rate_of_change', 'parameters': {'source': {'map_from': {'name': '(disk\\.device|disk)\\.(read|write)\\.(bytes|requests)', 'unit': '(B|request)'}}, 'target': {'map_to': {'name': '\\1.\\2.\\3.rate', 'unit': '\\1/s'}, 'type': 'gauge'}}}], 'name': 'disk_sink'}, {'publishers': ['notifier://'], 'transformers': [{'name': 'rate_of_change', 'parameters': {'source': {'map_from': {'name': 'network\\.(incoming|outgoing)\\.(bytes|packets)', 'unit': '(B|packet)'}}, 'target': {'map_to': {'name': 'network.\\1.\\2.rate', 'unit': '\\1/s'}, 'type': 'gauge'}}}], 'name': 'network_sink'}]}
~~~
* The above seems to be the configurations which the openstack-ceilometer-compute will load. But we don't see disk.usage there either.
* In order to find out which configuration file is used, let's have some verbose log.
* Checking the related code first:
~~~
/usr/lib/python2.7/site-packages/ceilometer/pipeline.py

    632     def load_config(self, cfg_info):
    633         """Load a configuration file and set its refresh values."""
    634         if isinstance(cfg_info, dict):
    635             conf = cfg_info
    636         else:
    637             if not os.path.exists(cfg_info):
    638                 cfg_info = cfg.CONF.find_file(cfg_info)
    639             with open(cfg_info) as fap:
    640                 data = fap.read()
    641                 
    642             conf = yaml.safe_load(data)
    643             self.cfg_loc = cfg_info
    644             self.cfg_mtime = self.get_cfg_mtime()
    645             self.cfg_hash = self.get_cfg_hash()
    646         LOG.info("Config file: %s", conf)
    647         return conf
~~~

* So here we need to focus on the cfg_info: if that's a dictionary then it is the configuration and no need to translate. But if it is not the it should be a file and some translation work needs to be done. So let's add some additional info here. 
~~~
# diff -u /usr/lib/python2.7/site-packages/ceilometer/pipeline.py /usr/lib/python2.7/site-packages/ceilometer/pipeline.py.bak
--- /usr/lib/python2.7/site-packages/ceilometer/pipeline.py	2017-12-01 09:25:06.590945959 -0500
+++ /usr/lib/python2.7/site-packages/ceilometer/pipeline.py.bak	2017-12-01 09:24:45.996107381 -0500
@@ -633,7 +633,10 @@
         """Load a configuration file and set its refresh values."""
         if isinstance(cfg_info, dict):
             conf = cfg_info
+            LOG.info("cfg_info is a dictionary")
         else:
+	    LOG.info("cfg_info is not a dictionary")
+            LOG.info("cfg_info is %s", cfg_info)
             if not os.path.exists(cfg_info):
                 cfg_info = cfg.CONF.find_file(cfg_info)
             with open(cfg_info) as fap:
~~~

* Then we restart the `openstack-ceilometer-agent` and then check the log:
~~~
2017-12-01 09:10:39.807 29245 INFO ceilometer.pipeline [-] cfg_info is not a dictionary
2017-12-01 09:10:39.808 29245 INFO ceilometer.pipeline [-] cfg_info is pipeline.yaml
~~~
* In this way we found out that the openstack-ceilometer-compute will read `pipeline.yaml` and if we check that file, we will find out that there is no disk.usage defined there. The solution is to append the disk.usage in that file and then restart the openstack-ceilometer-compute service and wait for the next interval to let the ceilometer-compute collect the disk.usage and then hand the data to gnocchi.
* Check the logs again to confirm `disk.usage` is correctly collected.
~~~
2017-12-01 09:36:36.432 2369 INFO ceilometer.agent.manager [-] Polling pollster disk.read.bytes in the context of some_pollsters
2017-12-01 09:36:36.436 2369 INFO ceilometer.agent.manager [-] Polling pollster network.incoming.packets in the context of some_pollsters
2017-12-01 09:36:36.444 2369 INFO ceilometer.agent.manager [-] Polling pollster network.outgoing.bytes in the context of some_pollsters
2017-12-01 09:36:36.447 2369 INFO ceilometer.agent.manager [-] Polling pollster disk.write.requests in the context of some_pollsters
2017-12-01 09:36:36.451 2369 INFO ceilometer.agent.manager [-] Polling pollster disk.read.requests in the context of some_pollsters
2017-12-01 09:36:36.455 2369 INFO ceilometer.agent.manager [-] Polling pollster cpu in the context of some_pollsters
2017-12-01 09:36:36.467 2369 INFO ceilometer.agent.manager [-] Polling pollster disk.usage in the context of some_pollsters
2017-12-01 09:36:36.481 2369 INFO ceilometer.agent.manager [-] Polling pollster network.incoming.bytes in the context of some_pollsters
2017-12-01 09:36:36.484 2369 INFO ceilometer.agent.manager [-] Polling pollster network.outgoing.packets in the context of some_pollsters
~~~

