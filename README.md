<img align="right" width="200" title="logo: a beaver with an ax (because it's a logger, just like Paul Bunyan)" src="./assets/images/beaver_PNG57.png">

# Bunyan: An Elixir Logger

* easily extended with additional data sources and writers
* can be networked (with one or more nodes also collecting messages from
  other nodes)
* humane formatting of multi-line messages (including error_logger and
  SASL)
* supports per-source and per-writer configuration, and the ability to
  log to multiple files and devices
* works with logrotate (send it a `SIGHUP` and it will close and reopen
  the log file).

## Summary

### Summary of the Summary

~~~ elixir
{ :bunyan, ">= 0.0.0" }
~~~

Basic logging:

~~~ elixir
require Bunyan

Bunyan.info "message or function"
Bunyan.info "message or function", «extra»
~~~

Message can have embedded newlines, which will be honored. `«extra»` can
be any Elixir term: maps are encouraged, as they are formatted nicely.

### API Overview

#### Logging Functions

You must `require Bunyan` before using any of these four functions.

* `Bunyan.debug(msg_or_fun, extra \\ nil)`
* `Bunyan.info(msg_or_fun, extra \\ nil)`
* `Bunyan.warn(msg_or_fun, extra \\ nil)`
* `Bunyan.error(msg_or_fun, extra \\ nil)`

#### Runtime Configuration

*





## Architecture

<img title="architecture diagram showing sources, collector, and writers" src="./assets/images/overall_architecture.png">

Bunyan is designed to take logging input from a variety of sources. It
is distributed with two: an API which provides the functions you can
call in your application, and an Erlang error handler which intercepts,
reformats, and injects Erlang and OTP errors.

The sources send log messages to the collector. This in turn forwards
the messages to the various log writers. Two writers come as standard:
one writes to standard error (or a file on disk), and the other writes
to a remote instance of Bunyan.

You can configure multiple instances of each type of writer. This would
allow you to (for example) log everything to standard error, errors only
to a log file, and warnings and errors to two separate remote loggers.


## Log Levels

The log levels are `debug`, `info`, `warn`, and `error`, with `debug`
being the lowest and `error` the highest.

You can set log levels indpendently for each of the sources and each of
the writers. Only messages at or above the specified level will be
logged.

The level set on a source determines which messages are sent on to the
writters. The level set on a writer determines which messages get
written.

In addition, the API source has an additional option to set the compile
time log level. Calls to logging functions will not be compiled if
they are below this level.

## Configuration

Configuration can be specified in the regular `config.xxx` files. Much
of it can also be set at runtime using the `Bunyan.config` function.


The top level configuration looks like this:


~~~ elixir
[
  accept_remote_as:       GlobalLogger,

  runtime_log_level:      :debug,
  compile_time_log_level: :info,

  read_from: [
    «list of sources»
  ],
  write_to: [
    «list of writers»
  ]
]
~~~

* `accept_remote_as:`

   If specified, this is the name that remote loggers will use to
   connect to (and send messages via) this logger. If not set (the
   default), this logger cannot be connected to.

* `read_from:`

   A list of sources (see below).

* `write_to:`

   A list of writers (also below).


### Source Configuration

Each source configuration entry can be either a module name, or a tuple
containing a module name and a keyword list of options:

`Bunyan.Source.API` _or_ `{ Bunyan.Source.API, compile_time_log_level::info }`

Bunyan comes with two source modules. Each has their own configuration.
(You can add your own source module—see below for details)

#### Source: `Bunyan.Source.API`

Provides a programmatic API that lets applications create log messages
and configure the logger.

Options:

* `runtime_log_level:` _:debug_, _:info_, _:warn_, or _:error_

   Only calls to the corresponding API functions at or above this level
   will be passed to the collector.

   Defaults to `:debug` in development, `:info` otherwise.

* `compile_time_log_level:` _:debug_, _:info_, _:warn_, or _:error_

   Calls to the corresponding API functions below this this level
   will be ignored: no code will be generated for them. Because of this,
   you don't want to give function calls with side effects as parameters
   to logging functions.

   Defaults to `:debug` in development, `:info` otherwise.



#### Source: `Bunyan.Source.ErlangErrorLogger`

Handles messages and reports generated by the Erlang `error_logger`. It
also handles SASL and OTP messages that the error logger forwards.

* `runtime_log_level:` _:debug_, _:info_, _:warn_, or _:error_

   Only calls to the corresponding API functions at or above this level
   will be passed to the collector.

   Defaults to `:debug` in development, `:info` otherwise.


### Writer Configuration

Bunyan comes with two writers (but you can add your own—see below).

#### Writer: `Bunyan.Writer.Device`

Writes log messages to standard error after formatting them for human
consumption.

* `name:`

   The OTP name associated with this device. Set this if you want to run
   multiple device writers, as each must have a distinct name. You'll
   also need to specify this if you want to set device-specific options
   in your code (as the name is used to identify which device to
   update).

* `device:`

   This writer will send messages to the specified device. This is
   either the name of an IO handler (such as `:user` or
   `:standard_error`) or a string containing a file name. It would be
   produce to make this filename an absolue path.

   Defaults to `:user`

* `pid_file_name:`

   If the log device is a file on disk, and if this option is set to the
   name of a pid file, the operating system pid of the writer will
   be stored in the pid file. This allows utilities such as
   [logrotate][1] to send a USR1 signal to the writer, which will cause
   the writer to close and reopen the log file.

* `runtime_log_level:` _:debug_, _:info_, _:warn_, or _:error_

   Only calls to the corresponding API functions at or above this level
   will be written to standard error.

   Defaults to `:debug` in development, `:info` otherwise.

* `main_format_string:`

   The format used for the first line of a log message (see Formats
   below)

   Defaults to `"$time [$level] $message_first_line"`

* `additional_format_string:`

   The format used on the remaining lines of the log message.

   Defaults to `"$message_rest\n$extra"`.

* `level_colors: %{ ... }`

   Specifies the colors to be used when displaying the `$level` field.
   This is a map of one or more entries where the keys are the log level
   and the value is a string used to prefix that level. You can use
   `IO.ANSI` to generate these strings.

   The defaults are:

   ~~~ elixir
   level_colors:   %{
     @debug => faint(),
     @info  => green(),
     @warn  => yellow(),
     @error => light_red() <> bright()
   }
   ~~~

* `message_colors:`

  The colors used for the message text at various log levels.

  ~~~ elixir
  message_colors: %{
    @debug => faint(),
    @info  => reset(),
    @warn  => yellow(),
    @error => light_red() <> bright()
  }
  ~~~

* `timestamp_color:`

  The attributes used to display the `$time`, `$date`, and `$datetime`
  fields. Defaults to `faint()`.

* `extra_color:`

  The attributes used to display the `$extra` field. Defaults to `italic()<>faint()`.

* `use_ansi_color?:`

  If falsy, the various color attributes will be ignored, and the log
  messages will not be colored.

  Defaults to `true` if writing to a console, `false` otherwise.

#### Writer:  Bunyan.Writers.Remote

Used to forward log messages to another instance of Bunyan.

* `runtime_log_level:` _:debug_, _:info_, _:warn_, or _:error_

   Only log messages at or above this level will be forwarded to the
   remote logger.

   Defaults to `:debug` in development, `:error` otherwise.

* `send_to:`

  The name of the logger to send the log messages to. This name must
  have been given as an `accept_remote_as` option to that logger.

* `retry_backoff_factor:`

  If this logged cannot find the remote logger, it initiates a series of
  retries, with each retry waiting `retry_backoff_factor` times longer
  than the previous. The first retry is after one second.

  The default is `3`, which means retries will take place at 1s, 3s, 9s,
  27s, and so on.

* `max_retry_backoff:`

  The maximum time (in seconds) that the exponential backoff will wait
  between retries. Defaults to 300s.

  Once the backoff period reaches this value, the writer will try one
  last time to contact the remote logger. If this fails, the writer will
  generate an `error` level local log message and exit.

### Log Message Format Specifications

The `Device` writer tries to create nicely formatted output. For
example, it will try to indent multi-line messages so the start of the
text of the message lines up, and it recognizes things such as maps when
laying out nontext data.

What it writes is under your control. You specify this using format
strings. Each message is potentially formatted using two formats. The
first of these, the `main_format_string` is used to write the first line
of the message. Typically this string will include some kind of time
stamp and a log level, as well as the first line of the actual log
message.

The `additional_format_string` is used to format the rest of the log
message (if any). The output generated under the control of this format
will automatically be indented to line up with the start of the message
in the first line.

Newlines in the message or in the format string will automatically cause
the message to be split and indented.

A format string consists of regular text and field names. The regular
text is simply copied into the resulting message. The contents of the
corresponding fields are substituted for the field names.

* `$date`

   The date the log message was sent (yy-mm-dd).

* `$time`

   The time the log message was sent (hh:mm:ss.mmm). because we'll all big
   girls and boys, this time will be in UTC.

* `$datetime`

   `"$date $time"`

* `$message`

   The whole log message. If the message contains newlines, it will be
   split, with the second and subsequent lines appearing beneath the
   first and left-aligned with it.

* `$msg_first_line`

   Just the first line of the message

* `$msg_rest`

   Lines 2... of the message

* `$level`

   The log level as a single character (D, I, W, or E)

* `$node`

   The node that generated the message

* `$pid`

   The pid that generated the message

* `$extra`

   For messages generated via the API, this will be the contents of the
   second parameter, formatted nicely.

   For reports coming from the Erlang error logger, this will be the
   raw content of the report.

The configuration options

~~~ elixir
[
  name:                   MyLogger,
  accept_remote_as:       GlobalLogger,

  runtime_log_level:      :debug,
  compile_time_log_level: :info,

  read_from:              [
    Bunyan.Source.Api,
    Bunyan.Source.ErlangErrorLogger,
  ],
  write_to:               [
    {
      Bunyan.Writer.Device, [
        main_format_string:        "$time [$level] $message_first_line",
        additional_format_string:  "$message_rest\n$extra",

        level_colors:   %{
          @debug => faint(),
          @info  => green(),
          @warn  => yellow(),
          @error => light_red() <> bright()
        },
        message_colors: %{
          @debug => faint(),
          @info  => reset(),
          @warn  => yellow(),
          @error => light_red() <> bright()
        },
        timestamp_color: faint(),
        extra_color:     italic() <> faint(),

        use_ansi_color?: true

      ]
    },
    #{ Bunyan.Writers.Remote, [ send_to: YourLogger, min_log_level: :warn ] },
  ]
]
~~~

## Why Another Logger?

I needed a distributed logger as part of the Toyland project, and
couldn't find what I needed. I also wanted something more decoupled than
the available options.

[1]: https://linux.die.net/man/8/logrotate