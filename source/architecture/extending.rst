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
Message or to any other values stored on the pipelinePack to influence further
processing.

"Appropriate task" is pretty vague, however. What task does a filter perform,
exactly? The specific function performed by a filter plugin is not as narrowly
or clearly defined as those of inputs or decoders. Filters are where the bulk
of Heka's message processing takes place and, as such, a filter might be
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

Message Injection
    A filter might possibly watch the pipeline for certain events to happen so
    that, when triggered, a new message is generated. This can be done by
    making use of `MessageGenerator` API (global to the pipeline package), as
    in this example::

        msgHolder := MessageGenerator.Retrieve()
        msgHolder.Message.Type = "yourtype"
        msgHolder.Message.Payload = "Your message payload"
        MessageGenerator.Inject(msgHolder)

Counting / Aggregation / Roll-ups
    In some cases you might want to count the number of messages of a
    particular type that pass through a Heka pipeline. One possible way to
    handle this is to implement a filter that does the counting. The filter
    could also perform simple roll-up operations by swallowing the original
    individual messages and using message injection to generate messages
    representing the aggregate.

Event / Anomaly Detection
    A filter might be coded to watch for specific message types or message
    events such that it notices when expected behavior is not happening. A
    simple example of this would be if an app generated a heartbeat message at
    regular intervals, a filter might be expecting these and would then notice
    if the heartbeats stopped arriving. This can be combined wiht message
    injection to generate notifications.

Note that this is merely a list of some of the more common uses for Heka
filter plugins. It is certainly not meant to be a comprehensive list of what
filters can do. A filter can perform any message processing that you can code.

Outputs
=======

Finally we come to the output plugins, which are responsible for receiving
Heka messages and using them to generate interactions with the outside world.
As with the other plugin types, the `Output` interface is simple, adding only
a single method to the base `Plugin` interface::

    type Output interface {
            Plugin
            Deliver(pipelinePack *PipelinePack)
    }

Despite the simplicity of the primary interface, however, Heka output plugins
often require a bit more complexity. To understand why, we'll need to
understand a few more `hekad` implementation details.

During `hekad` initialization, a large pool (default size: 1000) of
`PipelinePack` structs are created, to be reused throughout the life of the
process. Each of these pipeline packs contains its own separate set of output
plugins. That means that any `Output` plugins you define will be instantiated
not just once but `PoolSize` times.

If your output's `Deliver` method is simply writing the data out using
`log.Println` then this is fine. But if you're writing to a file, or over a
persistent network connection, this is a problem; you can't have 1000
simultaneous open, writable file handles for the same file, and you won't want
a single `hekad` instance to consume 1000 connections to the same remote
server. Instead, you'll want a way for all of the output plugins to share a
single file handle or network connection.

Heka provides support for sharing such resources among a pool of output
plugins. Rather than all of the work being handled by a single plugin object,
there are three related pieces:

`Output` plugin
    There must of course still be a plugin that must implement the `Output`
    interface, which will be responsible for extracting data from the message
    struct and prepping the data to go out over the wire. `PoolSize` copies of
    this object will be created.

`OutputWriter`
    You must also provide an implementation of the `OutputWriter` interface,
    which is responsible for taking the data that the plugin generates and
    actually sending it to the outside world. This is the object that should
    hold the file handle, network connection, or any other shared resources;
    only one will be instantiated.

`WriteRunner`
    Finally, there needs to be an object implementing the `WriteRunner`
    interface. This object handles the details of passing the output data
    objects back and forth between the `Output` and the `OutputWriter`. Heka
    provides a `WriteRunner` implementation that uses channels for this
    purpose.

More details are of course in order. It is easiest to start with looking at
the `OutputWriter` interface::

    type OutputWriter interface {
            MakeOutputData() interface{}
            Write(outputData interface{}) error
            Stop()
    }

`MakeOutputData`
    Each `OutputWriter` knows how to handle a specific type of data that is to
    be sent along to its destination. These are the objects that will be
    passed to the `Write` method. Many outputs use `[]byte` slices, but
    sometimes a different data type (such as a pointer to a specific struct)
    is required. The `OutputWriter` is responsible for creating these objects.
    `MakeOutputData` should allocate and initialize exactly one data structure
    and return it so it can be used for message passing.

`Write`
    The `Write` method performs the actual write. It should perform the following
    steps:

    * Apply a type assertion to the passed `interface{}` argument to verify
      that you have indeed been passed an output data object of the same type
      as those created by `MakeOutputData`.
    * Extract information as needed from output data object and perform the
      write operation.
    * Zero the output data object, if necessary.
    * Return either an appropriate error code or nil if the write was
      successful.

`Stop`
    Finally your `OutputWriter` should implement a `Stop` method that will be
    called during `hekad` shutdown. This should close any connections and/or
    tear down any structures to ensure clean shutdown.

Your output plugin won't be interacting directly with the `OutputWriter`,
however. Instead it will talk to the `WriteRunner`::

    type WriteRunner interface {
        RetrieveDataObject() interface{}
        SendOutputData(outputData interface{})
    }

As mentioned above, you don't have to provide this yourself, a channels-based
implementation of this interface already exists in the `heka/pipeline`
package.  In order to use these components, your output plugin's `Init` method
should create an `OutputWriter` of the correct type, and then call
`pipeline.NewWriteRunner(outputWriter)`, passing in the created writer. This
should be done **exactly once**, i.e. only a single
`WriteRunner`/`OutputWriter` pair should be created even though the `Init`
method will be called `PoolSize` times.

Your output plugin's `Deliver` method, then, should call the `WriterRunner`s
`RetrieveDataObject` method to get a data object into which the output data
can be placed. This data object should be populated and then passed in to
`WriteRunner.SendOutputData`.

.. rubric:: Output plugin / WriteRunner / OutputWriter

.. graphviz:: writerunner.dot

Output Example
==============

To put this together, let's construct an output that simply sends data out
over a generic network connection. For this we will need to implement
`NetworkOutput` and `NetworkOutputWriter` structs, using Heka's provided
`WriteRunner` implementation to pass byte slices containing the output data
between them.

First, the `NetworkOutputWriter`::

    type NetworkOutputWriter struct {
            outputBytes []byte
            conn        *net.Conn
    }

    func (self *NetworkOutputWriter) MakeOutputData() interface{} {
            return make([]byte, 0, 2000)
    }

    func (self *NetworkOutputWriter) Write(outputData interface{}) error {
            self.outputBytes = outputData.([]byte)
            n, err := self.conn.Write(self.outputBytes)
            if err == nil && n < len(self.outputBytes) {
                err = errors.New("MyOutputWriter message truncated.")
            }
            self.outputBytes = self.outputBytes[:0] // zero for reuse
            return err
    }

    func (self *NetworkOutputWriter) Stop() {
            self.conn.Close()
    }

Now the `NetworkOutput` plugin itself::

    // Used to make sure we only have one WriteRunner/NetworkOutputWriter
    // pair for each URL.
    var NetworkWriteRunners map[string]pipeline.WriteRunner

    type NetworkOutputConfig struct {
            URL string
    }

    type NetworkOutput struct {
            writeRunner pipeline.WriteRunner
            outBytes    []byte
    }

    func (self *NetworkOutput) ConfigStruct() interface{} {
            return new(NetworkOutputConfig)
    }

    func (self *NetworkOutput) Init(config interface{}) error {
            conf := config.(*NetworkOutputConfig)
            // Outputs are created serially so we don't need to mutex the map
            // access.
            self.writeRunner, ok = NetworkWriteRunners[conf.URL]
            if !ok {
                    conn, err := SomeNetConnObjectFactory(conf.URL)
                    if err != nil {
                        return err
                    }
                    writer := &NetworkOutputWriter{conn: conn}
                    self.writeRunner = pipeline.NewWriteRunner(writer)
                    NetworkWriteRunners[conf.URL] = self.writeRunner
            }
       return nil
    }

    func (self *NetworkOutput) Deliver(pack *pipeline.PipelinePack) {
            self.outBytes = self.writeRunner.RetrieveDataObject.([]byte)
            self.outBytes = append(self.outBytes, []byte(pack.Message.Payload)...)
            self.writeRunner.SendOutputData(self.outBytes)
    }

Once the `Deliver` method has passed the output data on to the `WriteRunner`
we're done. The `WriteRunner` will safely queue up the message to be delivered
by the `NetworkOutputWriter` at the next available opportunity.

A good example of a full, working output plugin using this system can be found
in the `CEF output plugin <https://github.com/mozilla-services/heka-mozsvc-
plugins/blob/master/outputs.go>`_. This uses a pointer to a `SyslogMsg` struct
as the data object, a `SyslogOutputWriter` as the output writer, and a
`CefOutput` as the actual output plugin.

Registering Your Plugin
=======================

The last step you have to take after implementing your plugin is to register
it with `hekad` so it can actually be configured and used. In
`pipeline/config.go <https://github.com/mozilla-
services/heka/blob/dev/pipeline/config.go>`_ an `AvailablePlugins` map (of
type `map[string]func() Plugin`) is defined. To make a new plugin available
for use, you must add your plugin identifier and a factory function returning
one of your plugins to this map. A sample of how to do so is provided in the
`hekad/plugin_loader.go.in <https://github.com/mozilla-
services/heka/blob/dev/hekad/plugin_loader.go.in>`_ file. Just copy this file
to `hekad/plugin_loader.go`, edit the code to insert your own plugin into the
`AvailablePlugins` map, rebuild, and you should be able to use your new plugin
by referencing it in the Heka config file (see :ref:`configuration`).
