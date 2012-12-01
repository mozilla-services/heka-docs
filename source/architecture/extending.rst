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

Plugins and Config
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

Input plugins are responsible for injecting messages into the Heka pipeline.
They might be passively listening for incoming network data, actively scanning
external sources (either on the local machine or over a network), or even just
creating messages from nothing based on triggers internal to the `hekad`
process. The input plugin interface is very simple::

    type Input interface {
            Plugin
            Read(pipelinePack *PipelinePack, timeout *time.Duration) error
    }

That is, in addition to the `Init` method that must be provided by all
plugins, there is only a single additional `Read` method that accepts a
pointer to a `PipelinePack` (in which to store the message data) and pointer
to a `time.Duration` (which specifies how much time the read operation should
allow to pass before a timeout is considered to have occurred). The only
return value is an error (or `nil` if the read succeeds).

Note that it is very important that your input plugin honors the specified
read timeout value by returning an appropriate error if the duration elapses
before the input can get the requested data. Heka creates a fixed number of
pipeline goroutines, and if your input's `Read` method does not return, then
it will be consuming a goroutine and removing it from the pool.

An input plugin that reads successfully can either output raw message bytes or
a fully decoded `Message` struct object. In the former case, the message bytes
should be written into the `pipelinePack.MsgBytes` byte slice attribute, and
the length of the slice should be adjusted to match the actual length of the
message content. In the latter case, the `pipelinePack.Message` attribute
points to a `Message` object that should be populated w/ the appropriate
values, and the `pipelinePack.Decoded` attribute should be set to `true` to
indicate that further decoding is not required.

In either case, for efficiency's sake, it is important to ensure that you are
actually writing the data into the memory that has already been allocated by
the `pipelinePack` struct, rather than creating new objects and repointing the
pipelinePack attributes to the ones you've created. Creating new objects each
time will end up causing a lot of allocation and garbage collection to occur,
which will definitely hurt Heka performance. A lot of care has been put into
the Heka pipeline code to reuse allocated memory where possible in order to
minimize garbage collector performance impact, but a poorly written plugin can
undo these efforts and cause significant (and unnecessary) slowdowns.

If an input generates raw bytes and wishes to explicitly specify which decoder
should be used (overriding the specified default), the input can modify the
`pipelinePack.Decoder` string value. The value chosen here *must* be one of
the keys of the `pipelinePack.Decoders` map or there will be an error
condition and the message will not be processed. And, obviously, the decoder
in question must know how to work with the provided message bytes, or the
decoding will fail, again resulting in the message being lost.

Decoders
========

Decoder plugins are responsible for converting raw bytes containing a message
into actual `Message` struct objects that the Heka pipeline can process. As
with inputs, the `Decoder` interface is quite simple::

    type Decoder interface {
            Plugin
            Decode(pipelinePack *PipelinePack) error
    }

A decoder's `Decode` method should extract the raw message data from
`pipelinePack.MsgBytes` and attempt to deserialize this and use the contained
information to populate the Message struct pointed to by the
`pipelinePack.Message` attribute. Again, to minimize GC churn, it's a good
idea to reuse the already allocated memory rather than creating new objects
and overwriting the existing ones.

If the message bytes are decoded successfully then `Decode` should return
`nil`. If not, then an appropriate error should be returned. The error message
will be logged and the message will be dropped, no further pipeline processing
will occur.

Filters
=======

As with inputs and decoders, the filter plugin interface adds just a single
method to the default `Plugin` interface shared by all Heka plugins::

    type Filter interface {
            Plugin
            FilterMsg(pipelinePack *PipelinePack)
    }

The `pipelinePack` (which, by the time filters are invoked, should always
contain a valid decoded Message struct pointed to by `pipelinePack.Message`)
will be passed by the Heka pipeline engine into the filter plugin, where the
filter can perform the appropriate task, making any changes to either the
Message or any other values stored on the pipelinePack to influence further
processing.

"Appropriate task" is pretty vague, however. What task should a filter be
performing? Unlike inputs and decoders, the exact function performed
by a filter plugin is not clearly defined. Filters are where the bulk of
Heka's message processing takes place and, as such, a filter might be
performing one of any number of possible jobs:

Filtering
    As the name suggests, one possible action a filter plugin can take is to
    block a message from any further processing. This immediately scraps the
    message, preventing it from being passed to any further filters or to any
    output plugins. This is accomplished by setting `pipelinePack.Blocked` to
    `true`.

Output Selection
    The set of output plugins to which the message will be provided is
    indicated by the `pipelinePack.OutputNames` map. Any filter can change the
    set of outputs for a given message by adding or removing keys to or from
    this set.

