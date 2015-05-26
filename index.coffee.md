nimble-direction
================

This module extends the `cfg` object with CCNQ4-related tools.

    module.exports = (cfg) ->
      debug "Configuring #{pkg.name} version #{pkg.version}.", cfg

`prefix_admin` is a CouchDB prefix (a base URI with no database name in the path) with admin access to the local CouchDB instance.

      assert cfg.prefix_admin?, 'Missing prefix_admin'

`prefix_source` is a CouchDB prefix (a base URI with no database name in the path) for the central database (so that we can start replications from it).

      assert cfg.prefix_source?, 'Missing prefix_source'

The optional `prov_master_admin` (string or array) are URIs to the master provisioning database(s) with admin access.

`master_push`
---------------

Push a document on the master provisioning database.
Typically used to push a design document so that we can filter for replication.

      prov_masters = []
      masters = cfg.prov_master_admin
      if masters?
        if typeof masters is 'string'
          masters = [masters]
        prov_masters = masters.map (name) ->
          new PouchDB name

      cfg.master_push = (doc) ->
        debug "Updating master design document #{doc._id}"
        Promise.all prov_masters.map (master) ->
          update master, doc

`push`
------

Push a document on the local provisioning database.
Typically used to push a design document so that we can query.

      cfg.push = (doc) ->
        update cfg.prov, doc

`replicate`
-----------

`replicate(name,extensions_cb)` stores a replication document for the database `name` from `prefix_source` to `prefix_admin`.
Before submission, the replication document is passed to the (optional) `extensions_cb` callback.
TODO: allow Array for prefix_source so that we can replicate from a multi-master database.

      replicator = new PouchDB "#{cfg.prefix_admin}/_replicator"
      cfg.replicate = (name,extensions) ->
        id = "Replicate #{name} from master"
        debug "Going to start replication of #{name}."
        Promise.resolve()

Create the target database if it doesn't already exist.

        .then ->
          target = new PouchDB "#{cfg.prefix_admin}/#{name}"
          target.info()
        .catch (error) ->
          debug error
          debug "Unable to create (target) local #{name} database"
          throw error

Retrieve the previous revision of the replication document.

        .then ->
          replicator.get id
        .catch (error) ->
          debug error
          debug '(ignored)'
          {}

Update the replication document.

        .then (doc) ->
          source = url.parse cfg.prefix_source
          doc._id ?= id
          doc.source =
            url: url.format
              protocol: source.protocol
              host: source.host
              pathname: name

Even with CouchDB 1.6.1 we still have the issue with CouchDB not properly managing authorization headers when a username and password are provided in the original URI. So let's handle it ourselves.

          if source.auth?
            auth = (new Buffer source.auth).toString 'base64'
            doc.source.headers =
              Authorization: "Basic #{auth}"

          doc.target = name
          doc.continuous = true

Let the callback add any field they'd like.

          extensions? doc

Cleanup replication state, otherwise CouchDB won't let us update.

          delete doc._replication_state
          delete doc._replication_state_reason
          delete doc._replication_state_time
          delete doc._replication_id

Finally, update the document.

          debug "Updating '#{id}'.", doc
          replicator.put doc

        .catch (error) ->
          debug error
          if error.status? and error.status is 403
            debug "Replication already started"
            return
          debug "Replication from #{cfg.prefix_source}/#{name} failed."
          throw error

`users`
-------

A PouchDB instance to the local users database. (We keep the original name.)

      cfg.users = new PouchDB "#{cfg.prefix_admin}/_users"

`prov`
------

A PouchDB instance to the local provisioning database.

      cfg.prov = new PouchDB "#{cfg.prefix_admin}/provisioning"

      Promise.resolve true

Toolbox
=======

    PouchDB = require 'pouchdb'
    Promise = require 'bluebird'
    assert = require 'assert'
    url = require 'url'
    pkg = require './package.json'
    debug = (require 'debug') "#{pkg.name}:config"
    update = require './update'
