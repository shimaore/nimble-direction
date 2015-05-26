    PouchDB = require 'pouchdb'
    memdown = require 'memdown'
    assert = require 'assert'

    describe 'update', ->
      update = require '../update'
      db = new PouchDB 'test', db: memdown

      before ->
        db.put
          _id:'foo'
          data: 3

      it 'should overwrite the document once', ->
        update db, _id:'foo', data:4
        .then ->
          db.get 'foo'
        .then (doc) ->
          assert doc.data is 4

      it 'should overwrite the document twice', ->
        update db, _id:'foo', data:5
        .then ->
          db.get 'foo'
        .then (doc) ->
          assert doc.data is 5

      it 'should create a new document', ->
        update db, _id:'bar', data:6
        .then ->
          db.get 'bar'
        .then (doc) ->
          assert doc.data is 6
