.. _architecture_overview:

====================
Architecture of Heka
====================

The big picture:

.. graphviz:: overview.dot

Metlog
======

Metlog (short for Metrics & Logging) is a group of client libraries
available for multiple languages for reporting application specific
logging and metrics data. It's capable of collecting:

- Logging output
- Custom counters + timers
- Exceptions

metlog-py
---------

Code: https://github.com/mozilla-services/metlog-py

Status: **Production**

Python library for Metlog.

metlog-cef
----------

Code: https://github.com/mozilla-services/metlog-cef

Status: **Development**

metlog-py extension to support sending CEF messages.

metlog-raven
------------

Code: https://github.com/mozilla-services/metlog-raven

Status:: **Development**

metlog-py extension to support capturing logging exceptions utilizing
raven to capture frame information for reporting to Sentry.

metlog-node
-----------

Code: https://github.com/mozilla-services/metlog-node

Status: **Development**

Node.js library for Metlog.

Heka Applications
=================

heka-agent
----------

Code: https://github.com/mozilla-services/heka

Status: **Development**

The heka-agent application is installed on every server that needs to
report and log information from a variety of sources:

- Log-file data, applying formatters for input lines
- Statsd counter information
- Metlog data

The agent runs an embedded statsd that handles counter/timer roll-ups
with a customizable flush period. The agent processes log-file data
and sends it to the heka-aggregator.

heka-grater
-----------

Code: https://github.com/mozilla-services/heka

Status: **Development**

The heka-grater application can be installed on a single machine
for smaller clusters, for larger clusters the heka-aggregator should be
installed on its own machine (or multiple machines depending on the
amount of data being recorded) with the heka-agents sending their data
to it.

heka-grater handles filtering the data from the heka-agents and routing
it to the appropriate end-point (Cassandra, Graphite, Nagios, etc.)
