% Logging Plugin Development
% Greg Schueler
% June 5, 2013

## About Rundeck Logging

When Rundeck executes a Job or adhoc execution, it runs workflow steps across multiple nodes and channels all output from these steps into a log for the execution.  This log contains the output from each step, as well as metadata about the context that the event occurred in, such as the node name, date stamp, and contextual data about the step being executed.

This logging system consists of these components:

* *Streaming Log Writers* - One or more outputs for the log data
* *Streaming Log Reader* - One input for reading the log data

When an execution starts, it writes log events to all outputs until it is done.
When a user views the execution log in the Rundeck GUI, or accesses it via the API, the input component is used to read the log events for the specific Execution.

Rundeck provides a built-in Reader and Writer, by writing the log output to a formatted file on disk, stored in the `var/logs` directory.  This is the **Local File Log**.

In addition, in Rundeck 2.0+, each execution generates a *state* file, which contains information about how each step and node executed.  This file is also stored on disk.

However local storage of log files is not always ideal, such as when deploying of Rundeck in a cloud environment where local disk storage could be ephemeral, or when clustering multiple Rundeck servers.  In those cases, it would be useful to have a way to store the log file and state file somewhere else, and retrieve them when necessary.

Rundeck has another component:

* *Execution File Storage* - A way of storing and retrieving file data in an external system

Rundeck has a plugin mechanism for all three of these components, allowing the logging system and file storage system to be adapted to different needs.  

Log Events are written to all configured Writer plugins, as well as the **Local File Log** if not disabled:

![Writer plugins](../figures/log storage1.png)

Events are read from either a Reader plugin, or the **Local File Log**:

![Reader plugins](../figures/log storage2.png)

When the **Local File Log** is used, the logs can be asynchronously stored to a Storage plugin after they are completed. Later, the logs can be asynchronously retrieved via the Storage plugin to be used by the **Local File Log**:

![Storage plugins](../figures/log storage3.png)

Here are some examples of how it can be used:

* replace the **Local File Log** entirely with another Reader and set of Writers
* duplicate all log output somewhere, in addition to using the **Local File Log**
* Supplement the **Local File Log** with a secondary log file storage system, so that local files can be removed and restored as needed

## Changes since Rundeck 1.6

In Rundeck 1.6, there was an `LogFileStorage` service and plugin system.  In Rundeck 2.0+, this has been replaced by the `ExecutionFileStorage` service and plugin system.

Any plugins written for Rundeck 1.6 will *not* work in Rundeck 2.0, and need to be updated to use the new mechanism.

## Types of Logging Plugins

There are three types of plugins that can be created:

* [StreamingLogWriter](#streaminglogwriter) - provides a stream-like mechanism for writing log events
* [StreamingLogReader](#streaminglogreader) - provides a stream-like mechanism for reading log events
* [ExecutionFileStorage](#executionfilestorage) - provides a way to both store and retrieve entire log files and execution state files

## Configuration

See the chapter [Plugins - Using Providers - Logging](../manual/plugins.html#logging).

## Logging Plugin Development

Rundeck supports two development modes for Logging plugins:

1. Java-based development deployed as a Jar file.
2. Groovy-based deployed as a single `.groovy` script.

Currently "script-based" plugins (shell scripts, that is) are not supported.

### Java Logging plugins

Java-based plugins can be developed just as any other Rundeck plugin, as described in the chapter [Plugin Development - Java Plugin Development](#java-plugin-development).

Your plugin class should implement the appropriate Java interface as described in the section for that plugin:

* [StreamingLogWriter](#streaminglogwriter)
* [StreamingLogReader](#streaminglogreader)
* [ExecutionFileStorage](#executionfilestorage)

To define configuration properties for your plugin, you use the same mechanisms as for Workflow Steps, described under the chapter [Workflow Step Plugin Development - Plugin Descriptions](workflow-step-plugin-development.html#plugin-descriptions).

The simplest way to do this is to use [Description Annotations](workflow-step-plugin-development.html#description-annotations). 

### Groovy Logging plugins

The Groovy plugin development method for Loggig Plugins is similar to [Notification Plugin Development - Groovy Plugins](notification-plugin-development.html#groovy-plugins).

Create a Groovy script, and define your plugin by calling the `rundeckPlugin` method, and pass it both the Class of the type of plugin, and a Closure used to build the plugin object.

    import com.dtolabs.rundeck.plugins.logging.StreamingLogWriterPlugin
    rundeckPlugin(StreamingLogWriterPlugin){
        //plugin definition goes here...
    }

## Example code

See the source directory `examples/example-groovy-log-plugins` for
examples of all three provider types written in Groovy.

* On github: [example-groovy-log-plugins](https://github.com/dtolabs/rundeck/tree/development/examples/example-groovy-log-plugins) 

See the source directory `examples/example-java-logging-plugins` for
Java examples.

* On github: [example-java-logging-plugins](https://github.com/dtolabs/rundeck/tree/development/examples/example-java-logging-plugins) 

## Execution Context Data

All three plugin types are given a Map of Execution "context data".  This is a dataset with information about the Execution that produced the log events.

This data map is the same as the "Job context variables" available when you execute a job or adhoc script, as described in the chapter [Job Workflows - Context Variables](../manual/job-workflows.html#context-variables).

Note that the Map keys will not start with `job.`, simply use the variable name, such as `execid`.

In addition, for ExecutionFileStorage plugins, another map entry named `filetype` will be specified, which indicates which type of file is being stored.  You need to use this filetype as part of the identifier for storing or retrieving the file.

## StreamingLogWriter

The `StreamingLogWriter` system receives log events from an execution and writes them somewhere.

The Java interface for these plugins is:
    
    com.dtolabs.rundeck.plugins.logging.StreamingLogWriterPlugin

### Java StreamingLogWriter

Create a Java class that implements the [StreamingLogWriterPlugin](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/plugins/logging/StreamingLogWriterPlugin.java) interface:

    /**
     * Plugin interface for streaming log writers
     */
    public interface StreamingLogWriterPlugin extends StreamingLogWriter {
        /**
         * Sets the execution context information for the log information being written, will be called prior to other
         * methods {@link #openStream()}
         *
         * @param context
         */
        public void initialize(Map<String, ? extends Object> context);
    }

This extends [StreamingLogWriter](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/core/logging/StreamingLogWriter.java):

    /**
     * writes log entries in a streaming manner
     */
    public interface StreamingLogWriter {
        /**
         * Open a stream, called before addEvent is called
         */
        void openStream() throws IOException;

        /**
         * Add a new event
         * @param event
         */
        void addEvent(LogEvent event);

        /**
         * Close the stream.
         */
        void close();
    }

The plugin is used in this manner:

1. When the plugin is instantiated, any configuration properties defined that have values to be resolved are set on the plugin instance
2. the `initialize` method is called with a map of [contextual data](#execution-context-data) about the execution
3. When an execution starts, the `openStream` method is called
4. For every log event that is emitted by the execution, each plugin's `addEvent` is called with the event
5. After execution finishes, `close` is called.

### Groovy StreamingLogWriter

Create a groovy script that calls the `rundeckPlugin` method and passes the `StreamingLogWriterPlugin` as the type of plugin:

    
    import com.dtolabs.rundeck.plugins.logging.StreamingLogWriterPlugin
    rundeckPlugin(StreamingLogWriterPlugin){
        //plugin definition
    }

To define metadata about your plugin, and configuration properties, see the [Notification Plugin Development - Groovy Notification Plugins - DSL - Definition](notification-plugin-development.html#definition) chapter.

Define these closures inside your definition:

`open`

    /**
     * The "open" closure is called to open the stream for writing events.
     * It is passed two map arguments, the execution data, and the plugin configuration data.
     *
     * It should return a Map containing the stream context, which will be passed back for later
     * calls to the "addEvent" closure.
     */
    open { Map execution, Map config ->
        //open a stream for writing
        //return a map containing any context you want to maintain
        [mycounter: ..., mystream: ... ]
    }

`addEvent`

    /**
     * "addEvent" closure is called to append a new event to the stream.  
     * It is passed the Map of stream context created in the "open" closure, and a LogEvent
     * 
     */
    addEvent { Map context, LogEvent event->
       // write the event to my stream
    }

`close`

    /**
     * "close" closure is called to end writing to the stream.
     *
     * In this example we don't declare any arguments, but an implicit 'context' variable is available with the stream
     * context data.
     */
    close { Map context ->
        // close my stream
    }


The plugin is used in this manner:

1. When an execution starts, the `open` closure is called with both the [contextual data](#execution-context-data) about the execution, and any configuration property values.  The returned context map is kept to pass to later calls
2. For every log event that is emitted by the execution, each plugin's `addEvent` is called with the event and the context map created by `open`
3. After execution finishes, `close` is called with the context map.

## StreamingLogReader

The `StreamingLogReader` system reads log events from somewhere for a specific execution and returns them in an iterator-like fashion.  Readers must also support an "offset", allowing the event stream to resume from some index within the stream.  These offset indices can be opaque to Rundeck (they could correspond to bytes, or event number, it is up to the plugin).  The plugin is expected to report an offset value when reading events, and be able to resume from a previously reported offset value. 

Additionally, these plugins should be able to report a `totalSize` (in an opaque manner), and a `lastModified` timestamp, indicating the last log event timestamp that was received.

The Java interface for these plugins is:
    
    com.dtolabs.rundeck.plugins.logging.StreamingLogReaderPlugin

### Java StreamingLogReader

Create a Java class that implements the [StreamingLogReaderPlugin](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/plugins/logging/StreamingLogReaderPlugin.java) interface:

    /**
     * Plugin interface for streaming log readers
     */
    public interface StreamingLogReaderPlugin extends StreamingLogReader {
        /**
         * Sets the execution context information for the log information being requested, will be called
         * prior to other methods {@link #openStream(Long)}, and must return true to indicate the stream is ready to be open, false otherwise.
         * @param context execution context data
         * @return true if the stream is ready to open
         */
        public boolean initialize(Map<String, ? extends Object> context);

    }

This extends [StreamingLogReader](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/core/logging/StreamingLogReader.java) interface:

    /**
     *  Reads log events in a streaming manner, and supports resuming from a specified offset.
     *
     *  @see LogEventIterator
     *  @see OffsetIterator
     *  @see Closeable
     *  @see CompletableIterator
     */
    public interface StreamingLogReader extends LogEventIterator, Closeable {
        /**
         * Read log entries starting at the specified offset
         *
         * @param offset
         *
         * @return
         */
        void openStream(Long offset) throws IOException;

        /**
         * Return the total size
         *
         * @return
         */
        long getTotalSize();

        /**
         * Return the last modification time of the log (e.g. last log entry time, or null if not modified)
         *
         * @return
         */
        Date getLastModified();
    }

Additional methods that must be implemented from super-interfaces:

    //from LogEventIterator
    LogEvent next();
    boolean hasNext();
    void remove(); //unused

    /**
     * Returns the current opaque offset within the underlying data stream
     *
     * @return
     */
    long getOffset(); //from OffsetIterator

    /**
     * Return true if the underlying source is completely exhausted, whether
     * or not there are any items to produce (may return false even if {@link Iterator#hasNext()} returns false).
     * @return true if underlying iteration source is exhasuted
     */
    boolean isComplete(); //from CompletableIterator

The plugin is used in this manner:

1. When the plugin is instantiated, any configuration properties defined that have values to be resolved are set on the plugin instance
2. the `initialize` method is called with a map of [contextual data](#execution-context-data) about the execution, if the method returns false, then clients are told that the log stream is pending.
2. the `getLastModified` method may be called, to determine if there are new events since a read sequence
3. The `openStream` method is called, possibly with an offset larger than 0, which indicates the event stream should begin at the specified offset
4. The `java.util.Iterator` methods will be called to iterate all available LogEvents
5. The `isComplete` method will be called to determine if the log output is complete, or may contain more entries later.
6. The `getOffset` method will be called to record the last offset read.
7. Finally, `close` is called.

Rundeck uses this interface to read the log events to display in the GUI, or send out via its API.  It uses the `offset`, as well as `lastModified`, to resume reading the log from a certain point, and to check whether there is more data since the last time it was read.

The implementation of the `isComplete` method is important, because it signals to Rundeck that all log events for the stream have been read and no more are expected to be available.  To be clear, this differs from the `java.util.Iterator#hasNext()` method, which returns true if any events are actually available.  `isComplete` should return false until no more events will ever be available.  

If you are developing a `StreamingLogWriter` in conjuction with a `StreamingLogReader`, keep in mind that the writer's `close` method will be called to indicate the end of the stream, which would be reflected on the reader side by `isComplete` returning true.

### Groovy StreamingLogReader

Create a groovy script that calls the `rundeckPlugin` method and passes the `StreamingLogReaderPlugin` as the type of plugin:

    
    import com.dtolabs.rundeck.plugins.logging.StreamingLogReaderPlugin
    rundeckPlugin(StreamingLogReaderPlugin){
        //plugin definition
    }

To define metadata about your plugin, and configuration properties, see the [Notification Plugin Development - Groovy Notification Plugins - DSL - Definition](notification-plugin-development.html#definition) chapter.

Define these closures inside your definition:

`info`

    /**
     * The 'info' closure is called to retrieve some metadata about the stream, 
     * such as whether it is available to read, totalSize of the content, and last
     *  modification time
     * 
     * It should return a Map containing these two entries:
     *  `ready` : a boolean indicating whether 'open' will work
     * `lastModified`: Long (unix epoch) or Date indicating last modification of the log
     * `totalSize`: Long indicating total size of the log, it doesn't have to indicate bytes,
     *     merely a measurement of total data size
     */
    info {Map execution, Map configuration->
      
        //return map containing metadata about the stream
        // it SHOULD contain these two elements:
        [
            lastModified: determineLastModified(),
            totalSize: determineDataSize(),
            ready: isReady()
        ]
    }

`open`

    /**
     * The `open` closure is called to begin reading events from the stream.
     * It is passed the execution data, the plugin configuration, and an offset.
     * It should return a Map containing any context to store between calls.
     */
    open { Map execution, Map configuration, long offset ->
        
        //return map of context data for your plugin to reuse later,
        [
            myfile: ...,
            mycounter:...
        ]
    }

`next`

    /**
     * Next is called to produce the next event, it should return a Map
     * containing: [event: (event data), offset: (next offset), complete: (true/false)].  
     * The event data can be a LogEvent, or a Map containing:
     * [
     * message: (String),
     * loglevel: (String or LogLevel),
     * datetime: (long or Date),
     * eventType: (String),
     * metadata: (Map),
     * ]
     * `complete` should be true if no more events will ever be available.
     */
    next { Map context->
        Map value=null
        boolean complete=false
        long offset=...
        try{
            value = readNextValue(...)
            complete = isComplete(..)
        }catch (SomeException e){
        }
        //event can be a Map, or a LogEvent
        return [event:event, offset:offset, complete:complete]
    }

`close`

    /**
     * Close is called to finish the read stream
     */
    close{ Map context->
        //perform any close action
    }

The plugin is used in this manner:

1. The `info` closure is called to determine the `lastModified` and `totalSize`.
1. The `open` closure is called with both the [contextual data](#execution-context-data) about the execution, and any configuration property values, and the expected read offset
2. The `next` closure is called repeatedly, until the result `event` entry is null, or `complete` is true.  The `offset` value is reported to the client.
3. The `close` closure is called with the context map.

## ExecutionFileStorage

The `ExecutionFileStorage` system is asked to store and retrieve entire log files and state files for a specific execution.

The Java interface for these plugins is:
    
    com.dtolabs.rundeck.plugins.logging.ExecutionFileStoragePlugin

Exection file storage allows Rundeck to store the files elsewhere, in case local file storage is not suitable for long-term retention. 

The ExecutionFileStorage service is used by two aspects of the Rundeck server currently.

1. Execution Log file - a data file containing all of the output for an execution
2. Execution state files - a data file containing all of the workflow step and node state information for an execution

### Storage behavior

If an ExecutionFileStoragePlugin is installed and configured to be enabled, Rundeck will use it in this way after an Execution completes:

Rundeck will place two *Storage Requests* in an asynchronous queue for that Execution to store the Log file and the State file.

When triggered, the *Storage Request* will use the configured ExecutionFileStorage plugin and invoke `store`:

* If it is unsuccessful, Rundeck may re-queue the request to retry it after a delay (configurable)

### Retrieval behavior

When a client requests a log stream to read via the **Local File Log**, or requests to read the **Execution Workflow State**, Rundeck determines if the file(s) are available locally.  If they are not available, it will start a *Retrieval Request* asynchronously for each missing file, and tell the client that the file is in a "pending" state.

The *Retrieval Request* will use the configured ExecutionFileStorage plugin, and invoke `retrieve`.

If successful, the client requests to read the **Local File Log** or **Execution Workflow State** should find the file available locally.  If unsuccessful, the result may be cached for a period of time to report to the client. After that time, a new *Retrieval Request* may be started if requested by a client.  After a certain number of attempts fail, further attempts will be disabled and return the cached status.  The retry delay and number of attempts can be configured.

### Execution File Availability

Your plugin will be asked if a file of a specific type is 'available', and should report back one of:

* `true` - the plugin can retrieve the file of the specified type
* `false` - the plugin cannot retrieve the file of the specified type

Only if `true` is reported will a *Retrieval Request* be created.

If there is an error discovering availability, your plugin should throw an Exception with the error message to report.

### Java ExecutionFileStorage

Create a Java class that implements the [ExecutionFileStoragePlugin](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/plugins/logging/ExecutionFileStoragePlugin.java) interface:

    /**
     * Plugin to implement {@link com.dtolabs.rundeck.core.logging.ExecutionFileStorage}
     */
    public interface ExecutionFileStoragePlugin extends ExecutionFileStorage {
        /**
         * Initializes the plugin with contextual data
         *
         * @param context
         */
        public void initialize(Map<String, ? extends Object> context);

        /**
         * Returns true if the file for the context and the given filetype is available, false otherwise
         *
         * @param filetype file type or extension of the file to check
         *
         * @return true if a file with the given filetype is available for the context
         *
         * @throws com.dtolabs.rundeck.core.logging.ExecutionFileStorageException
         *          if there is an error determining the availability
         */
        public boolean isAvailable(String filetype) throws ExecutionFileStorageException;
    }


This extends the the [ExecutionFileStorage](https://github.com/dtolabs/rundeck/tree/core/src/main/java/com/dtolabs/rundeck/core/logging/ExecutionFileStorage.java) interface:

    /**
     * Handles storage and retrieval of typed files for an execution, the filetype is specified in the {@link #store(String,
     * java.io.InputStream, long, java.util.Date)} and {@link #retrieve(String, java.io.OutputStream)} methods, and more
     * than one filetype may be stored or retrieved for the same execution.
     */
    public interface ExecutionFileStorage {
        /**
         * Stores a file of the given file type, read from the given stream
         *
         * @param filetype     filetype or extension of the file to store
         * @param stream       the input stream
         * @param length       the file length
         * @param lastModified the file modification time
         *
         * @return true if successful
         *
         * @throws java.io.IOException
         */
        boolean store(String filetype, InputStream stream, long length, Date lastModified) throws IOException,
                ExecutionFileStorageException;

        /**
         * Write a file of the given file type to the given stream
         *
         * @param filetype key to identify stored file
         * @param stream   the output stream
         *
         * @return true if successful
         *
         * @throws IOException
         */
        boolean retrieve(String filetype, OutputStream stream) throws IOException, ExecutionFileStorageException;
    }


The plugin is used in these two conditions:

* A log or state file needs to be stored via the plugin
* A log or state file needs to be retrieved via the plugin

1. When the plugin is instantiated, any configuration properties defined that have values to be resolved are set on the plugin instance
2. The `initialize` method is called with a map of [contextual data](#execution-context-data) about the execution.

When `retrieval` is needed:

1. The `isAvailable` method is called to determine if the plugin can retrieve the file, and the filetype is specified
2. If the method returns true, then `retrieve` method is called with the same filetype.

When `storage` is needed:

1. The `store` method is called with the filetype to store.

### Groovy ExecutionFileStorage

Create a groovy script that calls the `rundeckPlugin` method and passes the `ExecutionFileStoragePlugin` as the type of plugin:

    
    import com.dtolabs.rundeck.plugins.logging.ExecutionFileStoragePlugin
    rundeckPlugin(ExecutionFileStoragePlugin){
        //plugin definition
    }

To define metadata about your plugin, and configuration properties, see the [Notification Plugin Development - Groovy Notification Plugins - DSL - Definition](notification-plugin-development.html#definition) chapter.

Define these closures inside your definition:

`available`

    /**
     * Called to determine the file availability, return true to indicate it is available, 
     * false to indicate it is not available. An exception indicates an error.
     */
    available { String filetype, Map execution, Map configuration->
        //determine state
        return isAvailable()
    }

`store`

    /**
     * Called to store a log file, called with the execution data, configuration properties, and an InputStream.  Additionally `length` and `lastModified` properties are in the closure binding, providing the file length, and last modification Date.
     * Return true to indicate success.
     */
    store { String filetype, Map execution, Map configuration, InputStream source->
        //store output
        source.withReader { reader ->
            //...write somewhere
        }
        source.close()
        //return true if successful
        true
    }

`retrieve`

    /**
     * Called to retrieve a log file, called with the execution data, configuration properties, and an OutputStream.
     * Return true to indicate success.
     */
    retrieve {  String filetype, Map execution, Map configuration, OutputStream out->
        //get log file contents and write to output stream
        out << retrieveIt()
        //return true to indicate success
        true
    }

The plugin is used in this manner:

1. The `available` closure is called before retrieving the file, to determine if it is available, passing the filetype
1. The `store` closure is called when a file needs to be stored, with the filetype, the [contextual data](#execution-context-data), configuration Map, and InputStream which will produce the log data. Additionally `length` and `lastModified` properties are in the closure binding, providing the file length, and last modification Date.
2. The `retrieve` closure is called when a file needs to be retrieved, with the [contextual data](#execution-context-data), configuration Map, and OutputStream to write the log file content
