    CouchDB = require 'most-couchdb/with-update'
    assert = require 'assert'

    describe 'update', ->
      db = new CouchDB 'http://admin:password@couchdb:5984/test-update'

      before ->
        try await db.create()
        await db.put
          _id:'foo'
          data: 3

      after ->
        try await db.destroy()

      it 'should overwrite the document once', ->
        db.update _id:'foo', data:4
        .then ->
          db.get 'foo'
        .then (doc) ->
          assert doc.data is 4

      it 'should overwrite the document twice', ->
        db.update _id:'foo', data:5
        .then ->
          db.get 'foo'
        .then (doc) ->
          assert doc.data is 5

      it 'should create a new document', ->
        db.update _id:'bar', data:6
        .then ->
          db.get 'bar'
        .then (doc) ->
          assert doc.data is 6

      return
