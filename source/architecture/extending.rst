.. _architecture_extending:

==============
Extending Heka
==============

The core of the Heka engine is written in the `Go <http://golang.org>`_
programming language. Heka supports a plugin system, and plugins are also
written in Go. This document will try to provide enough information for
developers to extend Heka by implementing their own custom plugins. It assumes
a small amount of familiarity with Go, although any reasonably experienced
programmer will probably be able to follow along with no trouble.

Definitions
===========

We'll start by defining some terms that are crucial to understanding how Heka
works:

Message
    A message is the atomic unit of data that Heka deals with. It is a data
    structure related to a single event happening in the outside world, such
    as a log file entry, a counter increment, an application exception, a
    notification message, etc. It is specified as a `Message` struct in the
    `heka/message` packages `message.go file <https://github.com/mozilla-services/heka/tree/dev/message/message.go>`_.

Plugin
    Heka plugins are functional units that perform specific actions on or with
    messages. There are four distinct types of plugins: inputs, decoders,
    filters, and outputs.

Pipeline
    Messages being processed by Heka are passed through a specific set of
    plugins. A set of plugins to be applied make up what is called a Heka
    pipeline. Many Heka pipelines can be processed simultaneously, each
    running in its own goroutine.

PoolSize
   `PoolSize` is a Heka configuration setting which specifies the number of
   pipelines that can be concurrently processed in a running Heka server.
   (default: 1000)

PipelinePack
    In addition to the core message data, Heka needs to track some related
    state and configuration information for each message flows. To this end
    there is a `PipelinePack` struct defined in the `heka/pipeline` package's
    `pipeline_runner.go file <https://github.com/mozilla-
    services/heka/tree/dev/pipeline/pipeline_runner.go>`_. `PipelinePack`
    objects are what get passed in to Heka plugins as a message flows through
    a pipeline.

PluginWithGlobal / PluginGlobal
    When Heka starts up, `PoolSize` copies of each decoder, filter, and output
    plugin are created and are stored in `PipelinePack` objects. Many plugins,
    however, need to share access to a single, unique resource among the
    entire pool of plugin instances; it would not be a great idea to have 1000
    different copies of a file output all competing to open a handle to the
    same output file, for instance. For this reason, many plugins are of type
    `PluginWithGlobal` and have an accompanying `PluginGlobal` object (see
    `pipeline_runner.go <https://github.com/mozilla-
    services/heka/tree/dev/pipeline/pipeline_runner.go>`_). Each
    `PluginWithGlobal` plugin type specified in your Heka configuration will
    cause `PoolSize` instances of the plugin itself to be created but only a
    single `PluginGlobal` instance to be shared between them.

Runner / Writer / BatchWriter
    A very common pattern, especially with output plugins, is for each plugin
    instance to process the contents of a message and then pass the results
    along to be written out to a shared resource stored in a `PluginGlobal`.
    To simplify the building of such plugins, Heka provides a built in
    `Runner` plugin that works alongside a `Writer` (or `BatchWriter`, if you
    want to write in batches rather than for every single message)
    implementation that you provide. (see `runner_plugin.go
    <https://github.com/mozilla-
    services/heka/tree/dev/pipeline/runner_plugin.go>`_) This will be covered
    in more detail below.

Plugin Configuration
====================

Heka uses JSON as its configuration file format (see: :ref:`configuration`),
and provides a simple mechanism through which plugins can integrate with the
system to initialize based on settings in the config file.

The minimal shared interface that a Heka plugin must implement in order to use
the config system is (surprise, surprise) `Plugin`, defined in
`pipeline_runner.go <https://github.com /mozilla-
services/heka/blob/dev/pipeline/pipeline_runner.go>`_::

    type Plugin interface {
            Init(config interface{}) error
    }

During Heka initialization a pool of instances is created of every plugin
listed in the configuration file. The JSON configuration for that plugin will
be parsed and the result will be passed in to the `Init` method of each
instance as the `config` argument, of type `interface{}`. By default the
underlying type will be `map[string]interface{}` (per Go's default JSON
parsing behavior), but there is a way for plugins to specify a custom struct
to be used instead (more on that below). The plugin instances can then use the
values extracted from the config file to perform any required initialization.

As an example, imagine you were writing a plugin that would only effect
messages that were from a specific set of hosts. The hosts coule be specified
in the config as a JSON array of hostnames. and your plugin definition and
`Init` method might look like this::

    type MyPlugin struct {
            hosts []string
    }

    func (self *MyPlugin) Init(config interface{}) error {
            conf := config.(*pipeline.PluginConfig) // PluginConfig == map[string]interface{}
            hostsConf, ok := conf['hosts']
            if !ok {
                    return errors.New("MyPlugin: No 'hosts' setting configured.")
            }
            hostsSeq, ok := hostsConf.([]interface)
            if !ok {
                    return errors.New("MyPlugin: 'hosts' setting not a sequence.")
            }
            self.hosts = make([]string, 0, 10)
            for _, host := range(hostsSeq) {
                    if hostStr, ok := host.(string); ok {
                            self.hosts = append(self.hosts, hostStr)
                    } else {
                            return fmt.Errorf("MyPlugin: hostname not a string: %+v", host)
                    }
            }
            return nil
    }

If your plugin is going to require a global object shared among all of the
plugin instances in the pool then instead of `Plugin` you should provide the
closely related `PluginWithGlobal` interface, also defined in
`pipeline_runner.go <https://github.com /mozilla-
services/heka/blob/dev/pipeline/pipeline_runner.go>`_.::

type PluginWithGlobal interface {
    Init(global PluginGlobal, config interface{}) error
    InitOnce(config interface{}) (global PluginGlobal, err error)
}

When Heka loads configuration for a `PluginWithGlobal` type from the config
file, it will first create an instance of the plugin and then call `InitOnce`,
passing in the loaded config data. `InitOnce` should perform any one-time-only
initialization (opening an outgoing network connection, for example) and then
create and return a custom `PluginGlobal` object containing any resources that
will need to be shared among the plugin pool. Then it will create the pool of
plugin instances, calling `Init` and passing in both the PluginGlobal *and*
the config object.

To further demonstrate consider an output plugin that will take a message,
serialize it to JSON, and then send it out over a UDP connection. The
initialization code might look like so::

    type UdpJsonOutput struct {
            global *UdpJsonOutputGlobal
    }

    type UdpJsonOutputGlobal struct {
            conn net.Conn
    }

    // provides pipeline.PluginGlobal interface
    func (self *UdpJsonOutputGlobal) Event(eventType string) {
            if eventType == pipeline.STOP {
                    self.conn.Close()
            }
    }

    func (self *UdpJsonOutput) InitOnce(config interface{}) (pipeline.PluginGlobal, error) {
            conf := config.(*pipeline.PluginConfig)
            addr, ok := conf["address"]
            if !ok {
                    return nil, errors.New("UdpJsonOutput: No UDP address")
            }
            addrStr, ok := addr.(string)
            if !ok {
                    return nil, errors.New("UdpJsonOutput: UDP address not a string")
            }
            udpAddr, err := net.ResolveUdpAddr("udp", addr)
            if err != nil {
                    return nil, fmt.Errorf("UdpJsonOutput error resolving UDP address %s: %s",
                            addrStr, err.Error())
            }
            udpConn, err := net.DialUDP("udp", nil, udpAddr)
            if err != nil {
                    return nil, fmt.Errorf("UdpJsonOutput error dialing UDP address %s: %s",
                            addrStr, err.Error())
            }
            return &UdpJsonOutputGlobal{udpConn}, nil
    }

    func (self *UdpJsonOutput) Init(global pipeline.PluginGlobal, config interface{}) error {
            self.global = global // UDP connection available as self.global.conn
            return nil
    }

Custom Plugin Config Structs
============================

In simple cases it might be sufficient to receive plugin configuration data as
a generic `map[string]interface{}` type, but if there are more than a couple
of config settings then validating and extracting the values quickly becomes
unwieldy. Heka supports a rudimentary plugin configuration schema system by
making use of the Go language's automatic parsing of JSON values into suitable
struct objects.

Plugins that wish to provide a custom configuration struct that will be
populated from the config file JSON should implement the `HasConfigStruct`
interface defined in the `config.go file <https://github.com /mozilla-
services/heka/blob/dev/pipeline/config.go>`_::

    type HasConfigStruct interface {
            ConfigStruct() interface{}
    }

Your code should define a struct that can hold the required config values, and
you should then implement a `ConfigStruct` method on your plugin which will
initialize one of these and return it. Heka's config loader will then use this
object as the value to be populated when Go's `json.Unmarshal` is called with
the JSON from the config file. Note that this also gives you a mechanism for
specifying default config values, by populating your config struct as desired
before returning it from the `ConfigStruct` method.

Revisiting our example above, let's say we wanted to have our `UdpJsonOutput`
plugin default to writing to my.example.com, port 44444, the initialization
code might look as follows::

    type UdpJsonOutput struct {
            global *UdpJsonOutputGlobal
    }

    type UdpJsonOutputGlobal struct {
            conn net.Conn
    }

    // provides pipeline.PluginGlobal interface
    func (self *UdpJsonOutputGlobal) Event(eventType string) {
            if eventType == pipeline.STOP {
                    self.conn.Close()
            }
    }

    type UdpJsonOutputConfig struct {
            Address string
    }

    // provides pipeline.HasConfigStruct interface
    func (self *UdpJsonOutput) ConfigStruct() interface{} {
            return &UdpJsonOutputConfig{"my.example.com:44444"}
    }

    func (self *UdpJsonOutput) InitOnce(config interface{}) (pipeline.PluginGlobal, error) {
            conf := config.(*UdpJsonOutputConfig)
            udpAddr, err := net.ResolveUdpAddr("udp", conf.Address)
            if err != nil {
                    return nil, fmt.Errorf("UdpJsonOutput error resolving UDP address %s: %s",
                            conf.Address, err.Error())
            }
            udpConn, err := net.DialUDP("udp", nil, udpAddr)
            if err != nil {
                    return nil, fmt.Errorf("UdpJsonOutput error dialing UDP address %s: %s",
                            conf.Address, err.Error())
            }
            return &UdpJsonOutputGlobal{udpConn}, nil
    }    

    func (self *UdpJsonOutput) Init(global pipeline.PluginGlobal, config interface{}) error {
            self.global = global // UDP connection available as self.global.conn
            return nil
    }

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

.. rubric:: Output plugin / WriteRunner / OutputWriter

.. graphviz:: writerunner.dot

Drilling down a bit more, it is probably easiest to start with looking at the
`OutputWriter` interface::

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
