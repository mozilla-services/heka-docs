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

Plugins and config
==================

As mentioned above, there are four different types of Heka plugins: inputs,
decoders, filters, and outputs. Each are expected to have access to a
`PipelinePack` object, in which the message and related configuration and
other data is embedded. Each plugin type provides different functionality
and thus has its own specific interfaces, but they all share a small set of
features and interfaces related to hooking up to Heka's configuration engine.

The minimal shared interface that a Heka plugin must implement is (surprise,
surprise) `Plugin`, defined in `runner.go <https://github.com/mozilla-
services/heka/blob/dev/pipeline/runner.go>`_::

    type Plugin interface {
            Init(config interface{}) error
    }

`Init` should, as it implies, initialize your plugin. If the plugin doesn't
require any specific configuration details, then the `config` argument will
be `nil`, and `Init` can just perform any necessary setup and return `nil` for
the `error` value.

Most plugins **will** need custom configuration values, however, which will
have been loaded from the JSON config file. How will the Heka engine know what
these config options are, and how they should be parsed into values that your
plugin can actually use? That problem is solved via the use of a
`HasConfigStruct` interface defined in the `config.go file <https://github.com
/mozilla-services/heka/blob/dev/pipeline/config.go>`_::

    type HasConfigStruct interface {
            ConfigStruct() interface{}
    }

If your plugin requires custom configuration, then you should define a struct
can hold the required config values, and you should then implement a
`ConfigStruct` method on your plugin which will initialize one of the structs
and return it to Heka, which will then populate it from the config file and
will pass it in to the `Init` method.

For example, one of Heka's provided output plugins is the `FileOutput`, which
simply writes message data out to a file on the file system. Here are the
definitions for the config struct and the `ConfigStruct` method::

    type FileOutputConfig struct {
            Path string
            Format string
            Prefix_ts bool
            Perm os.FileMode
    }

    func (self *FileOutput) ConfigStruct() interface{} {
            return &FileOutputConfig{Format: "text", Perm: 0666}
    }

You'll notice that `ConfigStruct` not only creates a `FileOutputConfig` struct
object, but also populates it with suitable default configuration options.
Heka will then load any config from the config file and will pass the further
populated `FileOutputConfig` struct back in to the plugin's `Init` method. The
`Init` method will then use a type assertion to specify that the passed
`interface{}` is, indeed, a `FileOutputConfig`::

    conf := config.(*FileOutputConfig)

and then it can use the config values (i.e. `conf.Path`, `conf.Format`,
`conf.Prefix_ts`, and `conf.Perm`) to perform the necessary plugin
initialization.

Inputs
======
