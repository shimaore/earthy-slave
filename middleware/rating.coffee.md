    pkg = require '../package.json'
    @name = "#{pkg.name}:middleware:rating"
    debug = (require 'debug') @name
    seem = require 'seem'

    Rating = require 'entertaining-crib'
    PouchDB = require 'shimaore-pouchdb'

* cfg.rating (object, optional) parameters for the rating of calls
* cfg.rating.source (string) name of the cfg.rating.tables source. Default: `default`
* cfg.rating.tables (URI prefix) used to access the rating tables of the entertaining-crib module. Default: cfg.prefix_admin (from nimble-direction, i.e. env.NIMBLE_PREFIX_ADMIN)

    @server_pre = ->
      debug 'server_pre', @cfg
      @cfg.rating = new Rating
        source: @cfg.rating?.source ? 'default'
        rating_tables:
          if not @cfg.rating?.tables? or typeof @cfg.rating.tables is 'string'
            PouchDB.defaults prefix: @cfg.rating?.tables ? @cfg.prefix_admin
          else
            @cfg.rating?.tables
      debug 'server_pre: Ready'

    @include = seem ->

* session.rated.client (Rated object from entertaining-crib) rating object, client-side
* session.rated.carrier (Rated object from entertaining-crib) rating object, carrier-side

      stamp = new Date().toISOString()

      params =
          direction: @session.cdr_direction
          to: @session.ccnq_to_e164
          from: @session.ccnq_from_e164
          stamp: stamp
          client: @session.endpoint # from huge-play (egress-only since 15.x)
          carrier: @session.winner # from tough-rate

      @session.rated = yield @cfg.rating
        .rate params
        .catch (error) ->
          debug "rating_rate failed: #{error.stack ? error}"
          null

      @session.rated ?= {}
      @session.rated.params = params

      switch

This is the case e.g. for calls to voicemail.

        when not params.direction?
          debug 'Routing non-billable call: no direction provided'

This is the case e.g. for centrex-to-centrex (internal) calls.

        when params.direction is 'ingress' and not params.from? and not params.to?
          debug 'Routing non-billable call: no billable number on ingress'

System-wide configuration accepting non-billable calls.

- ingress

        when params.direction is 'ingress' and @cfg.route_non_billable_ingress_calls
          debug 'Routing non-billable ingress call: configuration allowed'

- both directions

        when @cfg.route_non_billable_calls
          debug 'Routing non-billable call: configuration allowed'

Reject non-billable (client-side) calls otherwise.

        when not @session.rated?.client?
          debug 'Unable to rate', @session.dialplan
          yield @respond '500 Unable to rate'
          return

Accept billable calls.

        else
          debug 'Routing'

The rating object was initialized, now apply the actual call billable (connected) duration.

      handle_report = (report) =>
        debug 'handle_report', report

        for own side, rated of @session.rated when side isnt 'params'
          debug 'computing', {side,rated}
          rated.compute Math.ceil parseInt(report.billable) / 1000
          @session.rated[side] = rated.toJSON()

        debug 'handle_report emit', @session.rated
        @call.emit 'rated', @session.rated

Handle both the case where the calls is over (sync)

      if @session.cdr_report?
        handle_report @session.cdr_report

or in-progress (async).

      else
        @call.once 'cdr_report'
        .then (report) -> handle_report report
        .catch (error) ->
          debug 'cdr_report', error.stack ? error.toString()
        return

      debug 'Ready'
