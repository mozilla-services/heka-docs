====
Heka
====

.. centered:: *Data Acquisition and Collection MadeEasy*

**Heka** simplifies the collection, analysis, and transportation of data
across your cluster/server. Counting downloads, keeping track of downloads
from Apache logfiles, reporting security issues is handled in a single
unified program thats easy to install locally for development and on servers
for deployment.

Features
========

**Multiple Data Sources**
    Data can be sent directly in via client libraries or using plug-ins that
    can read logfiles, handle statsd UDP input, or other custom data sources.

**Unified Daemon**
    Heka is capable of routing messages like logstash_, aggregating counters
    and submitting them to other sources like statsd_, and transporting log
    messages from various nodes like syslog.

**Plugin Architecture**
    In case existing plug-ins are insufficient, additional plug-ins may be
    written in Go or Lua (to run in the embedded Lua sandbox). Plug-ins can
    be utilized for input, decoding unstructured data, filtering, or as
    outputs.

**Performance**
    Written in Go_, heka utilizes light-weight goroutines for fast, efficient
    message routing, and parallel plug-in operation that can utilize multi-core
    systems to move hundreds of thousands of messages per second while using
    a very modest amount of memory.

**Usability**
    A single static binary with a library module is all that's needed to run
    heka locally, making it easy to develop with and deploy.

Common Uses
===========

- Application performance metrics
- Server load, memory consumption, and other machine metrics
- Database, Cache server, and other daemon metrics
- Various statsd counters
- Log-file transportion and statsd counter generation

Getting Started
===============

1. Install `hekad <http://hekad.readthedocs.org/en/latest/>`_.
2. Install one of the :ref:`client libraries <available clients>` or
   use `hekad` directly as a message router for various types of messages
   and/or reading logfiles.
3. Read the `hekad documentation <http://hekad.readthedocs.org/en/latest/>`_.

.. _support:

Support / Help
==============

Heka developers can be reached via the `services-dev
<https://mail.mozilla.org/listinfo/services-dev>`_ mailing list or via IRC on
the #heka channel of irc.mozilla.org.

Contents:

.. toctree::
   :maxdepth: 2

   architecture/index
   developing


* :ref:`glossary`

.. _Go: http://golang.org/
.. _logstash: http://www.logstash.net/
.. _statsd: https://github.com/etsy/statsd/
