.. _architecture_overview:

====================
Architecture of Heka
====================

.. rubric:: The big picture

.. graphviz:: overview.dot

Client Libraries (Metlog)
=========================

Metlog (short for Metrics & Logging) is a group of client libraries
available for multiple languages for reporting application specific
logging and metrics data. It's capable of collecting:

- Logging output
- Custom counters + timers
- Exceptions

The client libraries send data to an appropriate heka agent/aggregator
**Input**.

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

Both the heka-agent and heka-grater applications are generally
distinguished by their configuration. Internally their architecture is
similar as they both handle routing messages through a series of
filters which then are sent to outputs.

This model of message routing is heavily based on and inspired by the
routing of message in `logstash <http://logstash.net/>`_.


.. rubric:: Internal heka architecture

.. graphviz:: hekaflow.dot


Internally, all data sent into the heka applications becomes a message
which is then sent through a configured series of filters called a
*filter chain*.

An **input** is responsible for acquiring data. It may listen on a port
or multiple ports for network traffic, bind to ZeroMQ, or tail log
files for data. Some inputs require a decoder to translate the raw data
into a *message*.

A **decoder** may be used if the result from an input is not a message.
The decoder can then translate the data from the input into a message.
Some inputs may skip the decoding process.

A **message** may correspond to an event or line of data from a log file
or a statsd style timer/gauge/counter. Messages are created either by a
decoder or an input, and contain a set of basic fields as well as an
arbitrary set of key/values.

A **filter** may mark the outputs a message should be sent to, perform
side effects (like logging), mutate the message, or destroy the message
preventing other filters in the chain from being called. Filters are
applied to a message based on the input in a user configured chain.

An **output** takes a message and usually commits it to a back-end
service such as a database.

A **chain** refers to a series of filters and outputs that will be used
for a **message** of a specific type. Multiple chains can be configured
to handle specific message types.

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
