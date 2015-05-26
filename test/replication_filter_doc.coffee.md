    describe 'replication_filter_doc', ->
      rfd = require '../replication_filter_doc'
      pkg =
        name: 'foo'
        version: '1.3.2'
      it 'should return an object', ->
        assert 'object' is typeof rfd pkg

    assert = require 'assert'
