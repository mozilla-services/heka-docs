==================================================
Heka - Collect, aggregate, and visualize your data
==================================================

***IMPORTANT NOTE:*** **Heka is very new technology, and is not yet released,
nor is it in production use anywhere, not even by the authors. Developers and
interested parties are welcome to poke around and get involved, but it is
neither recommended nor supported for you to be running `hekad` in production
at this time.**

**Heka** is a tool that will **collect data from applications, databases and
servers** then aggregate them to a few systems **for monitoring, alerting, and
analytics**.

It's made for acquiring data from many sources including:

- Python/Node application performance metrics
- Server load, memory consumption, and other machine metrics
- Database, Cache server, and other daemon metrics
- Various statsd counters
- Log-files

The data is collected per server node by an agent and then flushed at
designated intervals to the aggregator which is responsible for routing them
to the final destination such as:

- Nagios
- Graphite
- Cassandra

The data can be used to trigger alerts, analyzed in real-time by the
aggregators, and fed into systems such as Cassandra for querying, long-term
trends, monitoring, and alerting.

Heka supports a plug-in system to allow for simple addition of new input,
output, or processing functionality.

Aspects of Heka overlap with features found in several open-source and
commercial SaaS offerings such as:

- Logstash
- DataDogHQ
- ServerDensity
- New Relic

Getting Started
===============

Heka is under development and is not ready for production use. Heka's main
component is `hekad`, a message processing and routing daemon that serves as
both the agent and the aggregator, depending on configuration. A good place to
get started is to read the architecture documentation, which provides an
overview of the Heka design as well as details on configuring and writing
plugins for the `hekad` daemon.

Contents:

.. toctree::
   :maxdepth: 2

   architecture/index
   architecture/extending
   configuration/index

Source / Support
================

Heka is an actively developed open-source project being worked on by
the Mozilla Services team. Contributions are welcome.

* Heka project `github page <https://github.com/mozilla-services/heka>`_
* Heka project
  `issue tracker <https://github.com/mozilla-services/heka/issues>`_

* Heka documentation project
  `source code <https://github.com /mozilla-services/heka-docs>`_
* Heka documentation project
  `issue tracker <https://github.com/mozilla-services/heka-docs/issues>`_

Heka developers can be reached via the `services-dev
<https://mail.mozilla.org/listinfo/services-dev>`_ mailing list or via IRC on
the #heka channel of irc.mozilla.org.
