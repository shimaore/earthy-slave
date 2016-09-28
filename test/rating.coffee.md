    chai = require 'chai'
    chai.should()
    seem = require 'seem'
    debug = (require 'debug') "#{(require '../package').name}:middleware:rating"
    PouchDB = (require 'pouchdb').defaults db: require 'memdown'
    describe 'rating', ->
      m = require '../middleware/rating'
      it 'should set `rated`', (done) ->
        p = seem ->
          trigger = null
          ctx =
            cfg:
              rating:
                source: 'local'
                tables: PouchDB
            session:
              cdr_direction:'egress'
              ccnq_to_e164: '18002288588'
              ccnq_from_e164: '33643482771'

Client-side data

              endpoint:
                _id: 'endpoint:something'
                rating:
                  '2016-01-01':
                    table: 'client+current'
                timezone: 'UTC'

Carrier-side data

              winner:
                rating:
                  '2016-01-01':
                    table: 'carrier+current'
                timezone: 'UTC'

            call:
              once: (event) ->
                debug 'once', event
                event.should.equal 'cdr_report'
                new Promise (resolve,reject) ->
                  debug 'Promise in action'
                  trigger = resolve
              emit: (event,rated) ->
                debug 'emit', event, rated
                event.should.equal 'rated'
                rated.should.have.property 'client'
                rated.client.should.have.property 'amount'
                rated.should.have.property 'carrier'
                rated.carrier.should.have.property 'amount'
                done()
          debug 'server_pre'
          m.server_pre.call ctx, ctx
          ctx.should.have.property 'cfg'
          ctx.cfg.should.have.property 'rating'

* cfg.rating.PouchDB ignore

          debug 'tables', ctx.cfg.rating.PouchDB
          db = new ctx.cfg.rating.PouchDB 'rates-client+current'
          yield db.put
            _id:'configuration'
            currency: 'EUR'
            divider: 1
            per: 60
          yield db.put
            _id:'prefix:1800'
            initial:
              cost: 0
              duration: 0
            subsequent:
              cost: 1
              duration: 60
          db.close()
          db = new ctx.cfg.rating.PouchDB 'rates-carrier+current'
          yield db.put
            _id:'configuration'
            currency: 'EUR'
            divider: 1
            per: 60
          yield db.put
            _id:'prefix:1800'
            initial:
              cost: 0
              duration: 0
            subsequent:
              cost: 0
              duration: 1
          db.close()

          debug 'include'
          yield m.include.call ctx, ctx
          debug 'include returned', ctx
          ctx.should.have.property 'session'
          ctx.session.should.have.property 'rated'
          ctx.session.rated.should.have.property 'client'
          ctx.session.rated.should.have.property 'carrier'
          debug 'trigger'
          trigger billable: 32262
        p().catch done
        null
