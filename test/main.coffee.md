    describe 'nimbre-direction', ->
      nimble = require '../index'

      prefix_admin = 'foo'
      prefix_source = 'bar'

      cfg = {prefix_admin,prefix_source}

      before ->
        nimble cfg

      it 'should provide `users`', ->
        assert 'object' is typeof cfg.users
      it 'should provide `prov`', ->
        assert 'object' is typeof cfg.prov
      it 'should provide `replicate`', ->
        assert 'function' is typeof cfg.replicate
      it 'should provide `push`', ->
        assert 'function' is typeof cfg.push
      it 'should provide `master_push`', ->
        assert 'function' is typeof cfg.master_push

      it 'should be idempotent', ->
        nimble cfg

TODO: actually test anything

    assert = require 'assert'
