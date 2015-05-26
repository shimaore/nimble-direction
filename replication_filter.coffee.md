replication filter
==================

Return a replication filter for the provisioning database.

The default (if no list of record types is provided) is to replicate all the known CCNQ4 provisioning records.

    default_replicate_types = [
      'carrier'
      'config'
      'destination'
      'domain'
      'emergency'
      'endpoint'
      'gateway'
      'host'
      'list'
      'location'
      'number'
      'ruleset'
    ]

    replication_filter = (replicate_types = default_replicate_types) ->

      main_code = (doc,req) ->

Do not attempt to replicate CouchDB-special documents (especially `_design` documents).

        if doc._id[0] is '_'
          return false

Use the `_id` to guess the doc type (this is basically the only piece of information we can reliably use).

        type_from_id = doc._id.split(':')[0]

If the document was deleted the `_id` is the only available piece of data.

        if doc._deleted? and doc._deleted and type_from_id in replicate_types
          return true

If the document still exists, make sure we're replicating a somewhat-consistent document.

        if not doc.type? or doc.type isnt type_from_id
          return false

We only replicate some documents types, not all of them.

        if doc.type not in replicate_types
          return false

If a `sip_domain_name` was provided in the query and the document contains a `sip_domain_name`, only replicate if they match.

        if req.query.sip_domain_name? and doc.sip_domain_name? and req.query.sip_domain_name isnt doc.sip_domain_name
          return false

        return true

The filter is a JavaScript closure.

      """
        (function () {
          var replicate_types = #{JSON.stringify replicate_types};
          #{p_fun main_code}
        })();

      """

Toolbox
=======

    {p_fun} = require 'coffeescript-helpers'
    module.exports = replication_filter
