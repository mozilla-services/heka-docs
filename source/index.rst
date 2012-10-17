==================================================
Heka - Collect, aggregate, and visualize your data
==================================================

Heka is the overarching project name for a set of tools that collects
data from applications, databases and servers; then aggregates them to
a few systems for monitoring, alerting, and analytics.

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


Contents:

.. toctree::
   :maxdepth: 2

   architecture/index
