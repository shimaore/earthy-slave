    describe 'Modules', ->
      list = [
          'middleware/rating.coffee.md'
        ]

      unit = (m) ->
        it "should load #{m}", ->
          ctx =
            cfg:
              sip_profiles:{}
              prefix_admin: ''
            session:{}
            call:
              once: -> Promise.resolve null
              emit: ->
            req:
              variable: -> null
          M = require "../#{m}"
          M.server_pre?.call ctx, ctx
          # M.include.call ctx, ctx

      for m in list
        unit m
