Replication document with a filter
==================================

This is a replication document meant to be inserted in a master provisioning database to help with replication to a local provisioning database.
The proper way to retrieve the name of the filter is to use the `replication_filter` field.

    replication_filter_doc = (pkg,replicate_types) ->
      id = "#{pkg.name}-#{pkg.version}-source"

      _id: "_design/#{id}"
      language: 'javascript'
      replication_filter: "#{id}/replication"
      package: pkg.name
      version: pkg.version

      filters:
        replication: replication_filter replicate_types

Toolbox
=======

    module.exports = replication_filter_doc
    replication_filter = require './replication_filter'

