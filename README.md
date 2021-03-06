pingmachine
===========

Pingmachine - Smokeping-like pinging framework

Introduction
------------
Monitoring components often need to do latency/loss measurements by sending
regularly some probes, typically ICMP ECHO (ping). Pingmachine was born out of
the idea to have a common framework that can take care of doing this "pinging".
That is useful, because pinging is more complicated than it seems: it needs to
be done efficiently (don't for too much), asynchronously (pinging takes time)
and data needs to be stored in a way that the most useful statistics can be
extracted.

The following were requirements when designing pingmachine (which might help
understand the reason for its architecture):

- Configurable targets
- Extensible probing methods (not only ping)
- High performance (should handle 10'000 targets)
- Storage of the measurements distribution plus the median value, if multiple
  probes to the same target are sent at each iteration.
- Consolidation using maximum, minimum and average functions.
- Management of old targets data (the list of targets change rather frequently!)

Smokeping vs. Pingmachine
-------------------------
After initial tests with Smokeping, we decided to write our own framework for
starting "ping-jobs" and collecting measurements. Our script is much simpler
than Smokeping, which is a complete monitoring solution, including a Web-site
to browse results. Nonetheless, "Pingmachine" is heavily inspired by Smokeping.
The main differences are:

- Smokeping is designed as a complete application, including the frontend. This
  is noticed in many places, for example in what must be configured for it to
  run at all (URL address, etc.)
- The Smokeping configuration is not dynamic. If it changes, then Smokeping
  needs to be reloaded. This could be a problem for us if we want to use it for
  highly dynamic applications (sang?)
- Generation of graphs doesn't work out of the box, because the graphs are
  generated by the CGI when displaying a page. We can't only ask Smokeping to
  generate a particular graph.

Interface for Applications
--------------------------
One of the things that pingmachine is optimized for, is high dynamicity of the
configuration. What needs to be measured could change every minute and
pingmachine should be able to handle it.

To make this possible, pingmachine uses a directory where configuration
"snippets" are put by whoever wants something to be done. These configuration
snippets are called *orders*. The configuration of one order is not allowed to
change. Orders can only be added or removed.

Pingmachine monitors that /orders directory, immediately picks up new orders,
and does what is necessary to start with the measurements. If the order file is
deleted, pingmachine will also automatically stop with that monitoring.

The output produced by the monitoring work, will be put in a separate directory
(/output), which can then be used by who give the order to fetch the data:

                       .----> /orders >----,
                      /                     \
      Application----.                       .---- pingmachine <----> fping
                      \                     /
                       `----< /output <----'


Orders Specification
--------------------
An order is typically one target IP address that needs to be monitored. If an
application needs to monitor an IP address, it just writes the corresponding
order file in the "/orders" directory (more about the exact directory structure
later).

Orders also need to be periodically refreshed (at least once an hour), to make
sure that pingmachine continues with the measurements. This mechanism is needed
to make sure that no stale configuration has the consequence of monitoring an
IP address indefinitely.

A example order could be (in YAML):

     user: tmon
     task: tun_eth0_4061
     step: 300
     pings: 20
     probe: fping
     fping:
         host: 62.179.116.250

The file name (the order "id") is determined by calculating the md5 checksum on
the file contents. This makes sure that different orders have different
identifiers and also that, if the same order reappears, it is going to have the
same id.

The file-system tree could look as follows:

     /var/lib/pingmachine/OSAGping/
     |---- orders/
     | |---- 6dd803dc5d29b72564467de7ddbfc695
     | |---- cd7d89acdba05cef56184db4a7b044ea

Note that orders are to be considered dynamic configuration, and are not meant
to be, for example, put in a buildall configuration archive. The "users" (tmon,
etc.) are the high-level programs that are configured by buildall, and they
will just install order files, as needed, to do the measurements, as they were
instructed. In other words: the complete /var/lib/pingmachine/OSAGping
directory can be completely deleted and recreated (with the loss of measured
data, however).

Output
------
The generated RRD file is put in a order-specific directory created under the
output tree. Each directory contains one rrd file: main.rrd. The definition of
the RRD file is the same for all probes and its creation handled by the main
script. This allows us to interchange probe types easily and to offload some
work out of the probe modules.

     |---- output/
     | |---- 6dd803dc5d29b72564467de7ddbfc695/
     | | |---- main.rrd
     | | `---- last_result
     | |---- ...

Also, the output directory also contains a last_result file, with just the
latest result of the pinging. It is meant to be used by programs that only need
information about the latest ping job. The format of the file is as follows:

     time: 1310116500
     updated: 1310116524
     step: 20
     pings: 1
     loss: 0
     min: 5.700000e-04
     median: 5.700000e-04
     max: 5.700000e-04

Archiving
---------
When an order is explicitly deleted or has timed out, the corresponding output
directory is moved to the archive directory:

     |---- archive/
     | |---- 2b45d6a19d2c3684767440fcb2f0b0c9/
     | | `---- main.rrd
     | |---- a36e19d8acd8a47acf06177680775507/

As soon as the "order" file of an archived order is put again into the orders
directory, pingmachine will move the output data into place again. It should
not be possible to have both data in output and in archive for the same order.

Installation
------------
Required perl modules:
- AE
- AnyEvent
- Log::Any
- Log::Any::Adapter::Dispatch
- Mouse
- MouseX::NativeTraits
- RRDs (RRDtool)
- Term::ANSIColor
- Try::Tiny
- YAML::XS

License
-------
See the LICENSE file for usage conditions.
