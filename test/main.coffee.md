    describe 'nimbre-direction', ->
      nimble = require '../index'

      prefix_admin = 'foo'
      prefix_source = 'bar'

      cfg = {prefix_admin,prefix_source}

      before ->
        nimble cfg

      it 'should provide `replicate`', ->
        assert 'function' is typeof cfg.replicate
      it 'should provide `replicate_up`', ->
        assert 'function' is typeof cfg.replicate_up
      it 'should provide `push`', ->
        assert 'function' is typeof cfg.push
      it 'should provide `master_push`', ->
        assert 'function' is typeof cfg.master_push

      it 'should be idempotent', ->
        nimble cfg

TODO: actually test anything

    assert = require 'assert'
