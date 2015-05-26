    describe 'replication_filter', ->
      rf = require '../replication_filter'
      it 'should return a string', ->
        assert 'string' is typeof rf()

      it 'should return a valid function', ->
        eval rf()

      it 'should return a function that validates a document', ->
        filter = eval rf()
        assert true is filter {type:'number',_id:'number:123'}, {query:{}}

    assert = require 'assert'
