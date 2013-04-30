.. _architecture_overview:

====================
Architecture of Heka
====================

.. rubric:: The big picture

.. graphviz:: overview.dot

Client Libraries
================

Client libraries for Heka are capable of collecting:

- Logging output
- Custom counters + timers
- Exceptions

The client libraries send data to `hekad` for routing.

.. _available_clients:

.. list-table:: Available Clients
    :widths: 15 10 30 5 20 20
    :header-rows: 1

    * - Client
      - Language
      - Features
      - Status
      - Code
      - Docs
    * - heka-py
      - Python
      - Metrics, Log Messages
      - **Production**
      - https://github.com/mozilla-services/heka-py
      - https://heka-py.readthedocs.org/en/latest/
    * - heka-cef
      - Python
      - CEF Messages for heka-py
      - **Development**
      - https://github.com/mozilla-services/heka-py-cef
      -
    * - heka-raven
      - Python
      - Exception Capturing
      - **Development**
      - https://github.com/mozilla-services/heka-py-raven
      -
    * - heka-node
      - Javascript (Node)
      - Metrics, Log Messages
      - **Development**
      - https://github.com/mozilla-services/heka-node
      -

Did you write a client for heka? :ref:`Let us know <support>`!

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

This model of message routing is inspired by the routing of messages in
`logstash <http://logstash.net/>`_, although heka introduces an explicit
decoding step and the idea of filter chains (see below).

hekad
-----

Binaries: https://docs.services.mozilla.com/_static/binaries/

Docs: https://hekad.readthedocs.org/en/latest/

Code: https://github.com/mozilla-services/heka

Status: **Beta**

Configurable daemon that can behave differently based on configuration.
