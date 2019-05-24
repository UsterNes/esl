Connection Listener (socket events handler)
===========================================

We use the same connection-listener for both client (FreeSwitch "inbound" socket) and server (FreeSwitch "outound" socket).
This is modelled after Node.js' http.js; the connection-listener is called either when FreeSwitch connects to our server, or when we connect to FreeSwitch from our client.

    class FreeSwitchParserError extends Error
      constructor: (args) ->
        super JSON.stringify args
        @args = args
        return

    connectionListener = (call) ->

The module provides statistics in the `stats` object if it is initialized. You may use it  to collect your own call-related statistics.

The parser will be the one receiving the actual data from the socket. We will process the parser's output below.

      parser = new FreeSwitchParser call.socket

Make the command responses somewhat unique. This is required since FreeSwitch doesn't provide us a way to match responses with requests.

      call.on 'CHANNEL_EXECUTE_COMPLETE', (res) ->
        event_uuid = res.body['Application-UUID']
        call.emit "CHANNEL_EXECUTE_COMPLETE #{event_uuid}", res

      call.on 'BACKGROUND_JOB', (res) ->
        job_uuid = res.body['Job-UUID']
        call.emit_later "BACKGROUND_JOB #{job_uuid}", {body:res.body._body}

The parser is responsible for de-framing messages coming from FreeSwitch and splitting it into headers and a body.
We then process those in order to generate higher-level events.

      parser.process = (headers,body) ->

Rewrite headers as needed to work around some weirdnesses in the protocol; and assign unified event IDs to the Event Socket's Content-Types.

        content_type = headers['Content-Type']
        if not content_type?
          if call.stats?
            call.stats.missing_content_type ?= 0
            call.stats.missing_content_type++
          call.emit 'error.missing-content-type', new FreeSwitchParserError {headers, body}
          return

Notice how all our (internal) event names are lower-cased; FreeSwitch always uses full-upper-case event names.

        switch content_type

auth/request
------------

FreeSwitch sends an authentication request when a client connect to the Event Socket.
Normally caught by the client code, there is no need for your code to monitor this event.

          when 'auth/request'
            event = 'freeswitch_auth_request'
            if call.stats?
              call.stats.auth_request ?= 0
              call.stats.auth_request++

command/reply
-------------

Commands trigger this type of event when they are submitted.
Normally caught by `send`, there is no need for your code to monitor this event.

          when 'command/reply'
            event = 'freeswitch_command_reply'

Apparently a bug in the response to `connect` causes FreeSwitch to send the headers in the body.

            if headers['Event-Name'] is 'CHANNEL_DATA'
              body = headers
              headers = {}
              for n in ['Content-Type','Reply-Text','Socket-Mode','Control']
                headers[n] = body[n]
                delete body[n]

            if call.stats?
              call.stats.command_reply ?= 0
              call.stats.command_reply++

text/event-json
---------------

A generic event with a JSON body. We map it to its own Event-Name.

          when 'text/event-json'
            if call.stats?
              call.stats.events ?= 0
              call.stats.events++

            try

Strip control characters that might be emitted by FreeSwitch.

              body = body.replace /[\x00-\x1F\x7F-\x9F]/g, ''

Parse the JSON body.

              body = JSON.parse(body)

In case of error report it as an error.

            catch exception
              trace 'Invalid JSON', body
              if call.stats?
                call.stats.json_parse_errors ?= 0
                call.stats.json_parse_errors++

              call.emit 'error.invalid-json', exception
              return

Otherwise trigger the proper event.

            event = body['Event-Name']

text/event-plain
----------------

Same as `text/event-json` except the body is encoded using plain text. Either way the module provides you with a parsed body (a hash/Object).

          when 'text/event-plain'
            body = parse_header_text(body)
            event = body['Event-Name']
            if call.stats?
              call.stats.events ?= 0
              call.stats.events++

log/data
--------

          when 'log/data'
            event = 'freeswitch_log_data'
            if call.stats?
              call.stats.log_data ?= 0
              call.stats.log_data++

text/disconnect-notice
----------------------

FreeSwitch's indication that it is disconnecting the socket.
You normally do not have to monitor this event; the `autocleanup` methods catches this event and emits either `freeswitch_disconnect` or `freeswitch_linger`, monitor those events instead.

          when 'text/disconnect-notice'
            event = 'freeswitch_disconnect_notice'
            if call.stats?
              call.stats.disconnect ?= 0
              call.stats.disconnect++

api/response
------------

Triggered when an `api` message returns. Due to the inability to map those responses to requests, you might want to use `queue_api` instead of `api` for concurrent usage.
You normally do not have to monitor this event, the `api` methods catches it.

          when 'api/response'
            event = 'freeswitch_api_response'
            if call.stats?
              call.stats.api_responses ?= 0
              call.stats.api_responses++

          when 'text/rude-rejection'
            event = 'freeswitch_rude_rejection'
            if call.stats?
              call.stats.rude_rejections ?= 0
              call.stats.rude_rejections++

Others?
-------

          else

Ideally other content-types should be individually specified. In any case we provide a fallback mechanism.

            trace 'Unhandled Content-Type', content_type
            event = "freeswitch_#{content_type.replace /[^a-z]/, '_'}"
            call.emit 'error.unhandled-content-type', new FreeSwitchParserError {content_type}
            if call.stats?
              call.stats.unhandled ?= 0
              call.stats.unhandled++

Event content
-------------

The messages sent at the server- or client-level only contain the headers and the body, possibly modified by the above code.

        msg = {headers,body}

        call.emit event, msg
        return

Get things started
------------------

Get things started: notify the application that the connection is established and that we are ready to send commands to FreeSwitch.

      call.emit 'freeswitch_connect'
      return

Server
======

The server is used when FreeSwitch needs to be able to initiate a connection to us so that we can handle an existing call.


We inherit from the `Server` class of Node.js' `net` module. This way any method from `Server` may be re-used (although in most cases only `listen` is used).

    net = require 'net'

    class FreeSwitchServer extends net.Server
      constructor: (requestListener) ->
        super()

        @on 'connection', (socket) ->

For every new connection to our server we get a new `Socket` object, which we wrap inside our `FreeSwitchResponse` object. This becomes the `call` object used throughout the application.

          call = new FreeSwitchResponse socket

The `freeswitch_connect` event is triggered by our `connectionListener` once the parser is set up and ready.

          call.once 'freeswitch_connect', ->

The request-listener is called within the context of the `FreeSwitchResponse` object.

            try
              requestListener.call call

All errors are reported on `FreeSwitchResponse`.

            catch exception
              call.emit 'error.listener', exception

The connection-listener is called last to set the parser up and trigger the request-listener.

          connectionListener call
          return

        return

The `server` we export is only slightly more complex. It sets up a filter so that the application only gets its own events, and sets up automatic cleanup which will be used before disconnecting the socket.
The call handler will receive a `FreeSwitchResponse` object, `options` are optional (and currently unused).

    exports.server = (options = {}, handler, report) ->
      if typeof options is 'function'
        [options,handler,report] = [{},options,handler]

      report ?= options.report
      report ?= (error) ->
        debug "Server: #{error}"

      options.all_events ?= true
      options.my_events ?= true

      assert.ok handler?, "server handler is required"
      assert.strictEqual typeof handler, 'function', "server handler must be a function"

      server = new FreeSwitchServer ->

Here starts our default request-listener.

        try
          Unique_ID = 'Unique-ID'

Confirm connection with FreeSwitch.

          @connect()
          .then (res) ->
            @data = res.body
            @uuid = @data[Unique_ID]

Restricting events using `filter` is required so that `event_json` will only obtain our events.

            @filter Unique_ID, @uuid if options.my_events
          .then ->
            @auto_cleanup()
          .then ->
            if options.all_events
              @event_json 'ALL'
            else
              @event_json 'CHANNEL_EXECUTE_COMPLETE', 'BACKGROUND_JOB'
          .then -> handler.apply this, arguments
          .catch -> report.apply this, arguments

        catch exception
          report exception

        return

      debug "Ready to start #{pkg.name} #{pkg.version} server."
      return server

Client
======

Client mode is used to place new calls or take over existing calls.

We inherit from the `Socket` class of Node.js' `net` module. This way any method from `Socket` may be re-used (although in most cases only `connect` is used).

    class FreeSwitchClient extends net.Socket
      constructor: ->
        super()

Contrarily to the server which will handle multiple socket connections over its lifetime, a client only handles one socket, so only one `FreeSwitchResponse` object is needed as well.

        @call = call = new FreeSwitchResponse this

Parsing of incoming messages is handled by the connection-listener.

        @once 'connect', ->
          connectionListener call
        return

The `client` function we provide wraps `FreeSwitchClient` in order to provide some defaults.
The `handler` will be called in the context of the `FreeSwitchResponse`; the `options` are optional, but may include a `password`.

    exports.default_password = 'ClueCon'

    exports.client = (options = {}, handler, report) ->
      if typeof options is 'function'
        [options,handler,report] = [{},options,handler]

      report ?= options.report
      report ?= (error) ->
        debug "Client report error: #{error}"

If neither `options` not `password` is provided, the default password is assumed.

      options.password ?= exports.default_password

      assert.ok handler?, "client handler is required"
      assert.strictEqual typeof handler, 'function', "client handler must be a function"

      client = new FreeSwitchClient()

Normally when the client connects, FreeSwitch will first send us an authentication request. We use it to trigger the remainder of the stack.

      client.call.onceAsync 'freeswitch_auth_request'
      .then ->
        @auth options.password
      .then -> @auto_cleanup()
      .then -> @event_json 'CHANNEL_EXECUTE_COMPLETE', 'BACKGROUND_JOB'
      .then -> handler.apply this, arguments
      .catch -> report.apply this, arguments

      debug "Ready to start #{pkg.name} #{pkg.version} client."
      return client

    exports.reconnect = (connect_options, options, handler, report) ->
      connect_options ?=
        host: '127.0.0.1'
        port: 8021
      {notify} = connect_options
      client = null
      running = true
      reconnect = (retry,attempt = 0) ->
        if not running
          debug "reconnect attempt ##{attempt}: stopping client to ", connect_options
          return
        debug "reconnect attempt ##{attempt} (retry is #{retry}ms): (re)connecting client to ", connect_options
        client?.destroy()

        client = exports.client options, handler, report
        client.on 'error', (error) ->
          if retry < 5000
            retry = (retry * 1200) // 1000 if error.code is 'ECONNREFUSED'
          debug "reconnect attempt ##{attempt}: client received `error` event: #{error.code} — #{error}. (Reconnecting in #{retry}ms.)"
          notify? 'reconnecting', retry
          setTimeout (-> reconnect retry, attempt+1), retry
          return
        client.on 'end', ->
          debug "reconnect attempt ##{attempt}: client received `end` event (remote end sent a FIN packet). (Reconnecting in #{retry}ms.)"
          notify? 'reconnecting', retry
          setTimeout (-> reconnect retry, attempt+1), retry
          return
        client.on 'close', (had_error) ->
          debug "reconnect attempt ##{attempt}: client received `close` event (due to error: #{had_error}). (Ignored.)"
          notify? 'close', had_error
          return

        client.connect connect_options
        ->
          debug "reconnect attempt ##{attempt}: end requested by application."
          notify? 'end'
          running = false
          client?.end()

      reconnect 200

createClient
============

    {EventEmitter2} = require 'eventemitter2'

Options are socket.connect options plus `password`.

    class Wrapper extends EventEmitter2
      constructor: (options) ->
        super()
        self = this
        notify = (event,args...) ->
          if event is 'error'
            if self.listeners('error').length > 0
              self.emit 'error', error
          else
            self.emit event, args...
          return

        handler = ->
          self.client = this
          self.emit 'connect', this
        report = (error) ->
          notify 'error', error

        options.notify = notify
        @end = exports.reconnect options, options, handler, report
        return

    for name in '''
      write send api bgapi event_json nixevent noevents filter filter_delete sendevent auth connect linger exit log nolog
      sengmsg_uuid sendmsg execute_uuid command_uuid hangup_uuid unicast_uuid
      execute command hangup unicast
    '''.split /\s+/
      do (name) ->
        Wrapper::[name] = (args...) ->
          trace "Wrapper::#{name}", args
          @client[name] args...

    exports.createClient = (options) ->
      new Wrapper options

Please note that the client is not started with `event_json ALL` since by default this would mean obtaining all events from FreeSwitch. Instead, we only monitor the events we need to be notified for (commands and `bgapi` responses).
You must manually run `@event_json` and an optional `@filter` command.

Toolbox
-------

    assert = require 'assert'

    FreeSwitchParser = require './parser'
    FreeSwitchResponse = require './response'
    {parse_header_text} = FreeSwitchParser

    pkg = require '../package.json'
    debug = (require 'debug') 'esl:main'
    trace = (require 'debug') 'esl:main:trace'
