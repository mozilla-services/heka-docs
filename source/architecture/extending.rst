.. _architecture_extending:

==============
Extending Heka
==============

The core of the Heka engine is written in the `Go <http://golang.org>`_
programming language. The rest of this document assumes some familiarity with
Go.

Messages, Pipelines, and Packs, oh my!
======================================

The atomic unit of data that Heka deals with is the **message**. The structure
of a Heka message is defined in the `heka/message <https://github.com/mozilla-
services/heka/tree/dev/message>`_ package, specifically in the `message.go
file <https://github.com/mozilla-services/heka/blob/dev/message/message.go>`_.
As described in the :ref:`architecture_overview` section, Heka messages flow
through a series of Heka **plugins** (inputs, decoders, filters, outputs),
which collectively is known as the Heka **pipeline**. The Heka pipeline is the
meat of the `hekad` daemon, and is implemented in the `heka/pipeline
<https://github.com/mozilla-services/heka/tree/dev/pipeline>`_ package.

When a message is being pushed through the Heka pipeline, however, more
information is needed than just the values on the Message struct. For this
reason Heka defines what is known as a `PipelinePack`, defined in the pipeline
package's `runner.go file <https://github.com/mozilla-
services/heka/tree/dev/pipeline/runner.go>`_.

With these fundamental concepts in mind, the rest of this document will try to
provide enough information for Go developers to extend Heka by implementing
their own custom plugins.
