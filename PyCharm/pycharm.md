# You have to disable multi thread first. The method to do this is to add thread=False to monkey_patch() function. For every component the location might be slightly different but finally I got the place of core components:
## nova
~~~
cmd/__init.py__
    eventlet.monkey_patch(os=False, thread=False)
~~~

## cinder
~~~
cmd/api.py

eventlet.monkey_patch(thread=False)
~~~

## neutron
~~~
common/eventlet_utils.py
    else:
        eventlet.monkey_patch(thread=False)
~~~

