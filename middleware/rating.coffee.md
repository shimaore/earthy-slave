    pkg = require '../package.json'
    @name = "#{pkg.name}:middleware:rating"
    debug = (require 'debug') @name
    seem = require 'seem'
    assert = require 'assert'

    Rating = require 'entertaining-crib'
    PouchDB = require 'pouchdb'

* cfg.rating (object, optional) parameters for the rating of calls
* cfg.rating.source (string) name of the cfg.rating.tables source. Default: `default`
* cfg.rating.tables (URI prefix) used to access the rating tables of the entertaining-crib module. Default: cfg.prefix_admin (from nimble-direction, i.e. env.NIMBLE_PREFIX_ADMIN)

    @server_pre = ->
      @cfg.rating = new Rating
        source: @cfg.rating?.source ? 'default'
        rating_tables: @cfg.rating?.tables ? PouchDB.defaults prefix: @cfg.prefix_admin

    @include = seem ->

* session.rated.client (Rated object from entertaining-crib) rating object, client-side
* session.rated.carrier (Rated object from entertaining-crib) rating object, carrier-side

      stamp = new Date().toISOString()

      @session.rated = yield @cfg.rating.rate
        direction: @session.cdr_direction
        to: @session.ccnq_to_e164
        from: @session.ccnq_from_e164
        stamp: stamp
        client: @session.endpoint # from huge-play
        carrier: @session.winner # from tough-rate

      assert @session.rated?, 'Unable to rate'

The rating object was initialized, now apply the actual call billable (connected) duration.

      handle_report = (report) =>
        debug 'handle_report', report

        for own side, rated of @session.rated
          debug 'computing', {side,rated}
          rated.compute Math.ceil report.billable / 1000
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
