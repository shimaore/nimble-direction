replication filter
==================

Return a replication filter for the provisioning database.

The default (if no list of record types is provided) is to replicate all the known CCNQ4 provisioning records.

    default_replicate_types = [
      'carrier'
      'config'
      'domain'
      'emergency'
      'endpoint'
      'gateway'
      'host'
      'list'
      'location'
      'number'
      'number_domain'
      'ruleset'
    ]

    replication_filter = (replicate_types = default_replicate_types) ->

      main_code = (doc,req) ->

Use the `_id` to guess the doc type (this is basically the only piece of information we can reliably use).

        type_from_id = doc._id.split(':')[0]

        return type_from_id in replicate_types

The filter is a JavaScript closure.

      extra = "var replicate_types = #{JSON.stringify replicate_types};"
      p_fun extra, main_code

Toolbox
=======

    {p_fun} = require 'coffeescript-helpers'
    module.exports = replication_filter
