    describe 'replication_filter', ->
      rf = require '../replication_filter'
      it 'should return a string', ->
        assert 'string' is typeof rf()

    assert = require 'assert'
