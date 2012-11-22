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

The client libraries send data to an appropriate hekad **Input**.

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

Hekad Application
=================

The hekad application daemon is distinguished by the role its
configured for. Depending on the configuration, hekad's role may be:

- Agent
- Aggregator
- Router
- Analyzer

`hekad` may be configured to act in one or more roles as necessary for
the environment its deployed in. Regardless of the configured role,
internally `hekad` works the same, moving messages through a series of
filters which then are sent to outputs. More information on the roles
and setting them up can be found in the :ref:`configuration` section.

This model of message routing is heavily based on and inspired by the
routing of message in `logstash <http://logstash.net/>`_.

.. rubric:: Internal heka architecture

.. graphviz:: hekaflow.dot

Internally, all data sent into `hekad` becomes a message which is then
sent through a configured series of filters called a *filter chain*.

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
applied to a message type based their filter chain configuration.

An **output** takes a message and usually commits it to a back-end
service such as a database.

A **filter chain** refers to a series of filters and outputs that will
be used for a **message** of a specific type. Multiple chains can be
configured to handle specific message types.

When `hekad` has a message, a set of filters and outputs configured for
the message type are loaded based on the configuration. The filters are
then run in the order specified and can indicate which of the
designated outputs the message will be sent to (or none of them). If no
filters are configured for a message type (or they don't alter the
output list), its sent to each of the configured outputs.

hekad
-----

Code: https://github.com/mozilla-services/heka

Status: **Development**

Configurable daemon that can behave differently based on configuration.
