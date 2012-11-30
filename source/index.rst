==================================================
Heka - Collect, aggregate, and visualize your data
==================================================

**Heka is** the overarching project name for **a set of tools that
collects data from applications, databases and servers**; then
aggregates them to a few systems **for monitoring, alerting, and
analytics**.

It's made for acquiring data from many sources including:

- Python/Node application performance metrics
- Server load, memory consumption, and other machine metrics
- Database, Cache server, and other daemon metrics
- Various statsd counters
- Log-files

The data is collected per server node by an agent and then flushed
at designated intervals to the aggregator which is responsible for
routing them to the final destination such as:

- Nagios
- Graphite
- Cassandra

The data can be used to trigger alerts, analyzed in real-time by the
aggregators, and fed into systems such as Cassandra for querying, long-
term trends, monitoring, and alerting.

Aspects of Heka overlap with features found in several open-source and
commercial SaaS offerings such as:

- DataDogHQ
- ServerDensity
- New Relic

Getting Started
===============

Heka is under development at the moment, a good place to get started is
to read the architecture documentation for a complete idea of how the
various parts work together.

The architecture document lays out the big picture and provides links
to individual components. Detailed API references for client libraries
is contained in their respective documentation.

Configuration for the core `hekad` daemon and its roles is located
here.

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

Bugs and issues on the Heka project may be filed on the project's github page
`issue tracker <https://github.com/mozilla-services/heka/issues>`_. Github is
also where you can find the Heka `source code <https://github.com/mozilla-
services/heka>`_.

Bugs and issues related to the Heka documentation that you are now reading can
be filed on the Heka-Docs project's `issue tracker <https://github.com
/mozilla-services/heka-docs/issues>`_. The Heka-Docs `source code
<https://github.com /mozilla-services/heka-docs>`_ is similarly available.
