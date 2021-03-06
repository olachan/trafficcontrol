..
..
.. Licensed under the Apache License, Version 2.0 (the "License");
.. you may not use this file except in compliance with the License.
.. You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing, software
.. distributed under the License is distributed on an "AS IS" BASIS,
.. WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
.. See the License for the specific language governing permissions and
.. limitations under the License.
..

******************************
Traffic Monitor Administration
******************************

.. _tm-golang:

Installing Traffic Monitor
==========================

The following are hard requirements requirements for Traffic Monitor to operate:

* CentOS 7+
* Successful install of Traffic Ops (usually on a separate machine)
* Administrative access to the Traffic Ops (usually on a separate machine)


These are the recommended hardware specifications for a production deployment of Traffic Monitor:

* 8 CPUs
* 16GB of RAM
* It is also recommended that you know the physical address of the site where the Traffic Monitor machine lives for optimal performance

#. Enter the Traffic Monitor server into Traffic Portal

	.. note:: For legacy compatibility reasons, the 'Type' field of a new Traffic Monitor server must be 'RASCAL'.

#. Make sure the Fully Qualified Domain Name (FQDN) of the Traffic Monitor is resolvable in DNS.
#. Install Traffic Monitor, either from source or by running the command ``yum install traffic_monitor`` as the root user, or with ``sudo``.
#. Configure Traffic Monitor. See :ref:`here <tm-configure>`
#. Start the service, usually by running the command ``systemctl start traffic_monitor`` as the root user, or with ``sudo``
#. Verify Traffic Monitor is running by e.g. opening your preferred web browser to port 80 on the Traffic Monitor host.

Configuring Traffic Monitor
===========================

Configuration Overview
----------------------

.. _tm-configure:

Traffic Monitor is configured via two JSON configuration files, ``traffic_ops.cfg`` and ``traffic_monitor.cfg``, by default located in the ``conf`` directory in the install location. ``traffic_ops.cfg`` contains Traffic Ops connection information. Specify the URL, username, and password for the instance of Traffic Ops of which this Traffic Monitor is a member. ``traffic_monitor.cfg`` contains log file locations, as well as detailed application configuration variables such as processing flush times and initial poll intervals. Once started with the correct configuration, Traffic Monitor downloads its configuration from Traffic Ops and begins polling caches. Once every cache has been polled, :ref:`health-proto` state is available via RESTful JSON endpoints.

Stat and Health Flush Configuration
-----------------------------------

The Monitor has a health flush interval, a stat flush interval, and a stat buffer interval. Recall that the monitor polls both stats and health. The health poll is so small and fast, a buffer is largely unnecessary. However, on a large CDN, the stat poll may involve thousands of caches with thousands of stats each, or more, and CPU may be a bottleneck.

The flush intervals, ``health_flush_interval_ms`` and ``stat_flush_interval_ms``, indicate how often to flush stats or health, if results are continuously coming in with no break. This prevents starvation. Ideally, if there is enough CPU, the flushes should never occur.

The default flush times are 200 milliseconds, which is suggested as a reasonable starting point, from which operators may adjust them higher or lower, depending on the need to get health data and stop serving to unhealthy caches as quickly as possible, balanced by the need to reduce CPU usage.

The stat buffer interval, ``stat_buffer_interval_ms``, also provides a temporal buffer for stat processing. Stats will not be processed except after this interval, whereupon all pending stats will be processed, unless the flush interval occurs as a starvation safety. The stat buffer and flush intervals may be thought of as a state machine with two states: the "buffer state" accepts results until the buffer interval has elapsed, whereupon the "flush state" is entered, and results are accepted while outstanding, and processed either when no results are outstanding or the flush interval has elapsed.

Note that this means the stat buffer interval acts as "bufferbloat," increasing the average and maximum time a cache may be down before it is processed and marked as unhealthy. If the stat buffer interval is non-zero, the average time a cache may be down before being marked unavailable is half the poll time plus half the stat buffer interval, and the maximum time is the poll time plus the stat buffer interval. For example, if the stat poll time is 6 seconds, and the stat buffer interval is 4 seconds, the average time a cache may be unhealthy before being marked is 6/2 + 4/2 = 6 seconds, and the maximum time is 6+4=10 seconds. For this reason, if operators feel the need to add a stat buffer interval, it is recommended to start with a very low number, such as 5 milliseconds, and increase as necessary.

The default stat buffer interval is 0, which results in all stats being processed as quickly as possible. This is recommended, if a CDN operator is okay with the CPU usage and processing time, to minimize the time a cache may be unhealthy before being marked unavailable. If operators feel the need to set the stat buffer interval to a nonzero value, a recommended starting point is 5 milliseconds.

It is not recommended to set either flush interval to 0, irrespective of the stat buffer interval. This will cause new results to be immediately processed, with little to no processing of multiple results concurrently. Result processing does not scale linearly, for example, it is not significantly more CPU or time to process 100 results than 10 at once. Thus, a flush interval which is too low will cause increased CPU usage, and potentially increased overall poll times, with little or no benefit. The default value of 200 milliseconds is recommended as a starting point for configuration tuning.


Troubleshooting and Log Files
=============================
Traffic Monitor log files are in ``/opt/traffic_monitor/var/log/``.
