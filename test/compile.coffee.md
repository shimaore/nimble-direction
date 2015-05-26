    describe 'The modules', ->
      it 'should compile', ->
        require '../index'
        require '../replication_filter'
        require '../replication_filter_doc'
        require '../update'
