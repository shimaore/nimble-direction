    describe 'nimbre-direction', ->
      nimble = require '../index'

      prefix_admin = 'foo'
      prefix_source = 'bar'

      cfg = {prefix_admin,prefix_source}

      N = nimble cfg

      it 'should provide `replicate`', ->
        assert 'function' is typeof N.replicate
      it 'should provide `replicate_up`', ->
        assert 'function' is typeof N.replicate_up
      it 'should provide `push`', ->
        assert 'function' is typeof N.push
      it 'should provide `master_push`', ->
        assert 'function' is typeof N.master_push

      return

    assert = require 'assert'
