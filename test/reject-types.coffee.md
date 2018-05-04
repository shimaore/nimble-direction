    describe 'reject-types', ->
      rfd = require '../reject-types'

      it 'should return an object', ->
        assert 'object' is typeof rfd
      it 'should return a string', ->
        assert 'string' is typeof rfd.validate_doc_update
      it 'should contain a function', ->
        assert 'function' is typeof eval rfd.validate_doc_update

      it 'should return a valid function', ->
        assert 'function' is typeof eval rfd.validate_doc_update

      it 'should return a function that validates a document', ->
        fun = eval rfd.validate_doc_update
        fun {type:'number',_id:'number:123'}

      it 'should return a function that validates a document', (done) ->
        fun = eval rfd.validate_doc_update
        try
          fun {type:'blob',_id:'blob:123'}
        catch error
          return done() if error.forbidden
        done new Error 'Missed'

    assert = require 'assert'
