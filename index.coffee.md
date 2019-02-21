nimble-direction
================

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    module.exports = (cfg) ->

Configuration variables
-----------------------

### `prefix_admin` / `NIMBLE_PREFIX_ADMIN`  (required)

`cfg.prefix_admin` (string,required) is a CouchDB prefix (a base URI with no database name in the path) with admin access to the _local_ CouchDB instance. Default: the value of the `NIMBLE_PREFIX_ADMIN` environment variable.

      {prefix_admin} = cfg
      prefix_admin ?= process.env.NIMBLE_PREFIX_ADMIN

One of the environment variable or the `.prefix_admin` configuration item is required.

      assert prefix_admin?, 'Missing prefix_admin'

### `prefix_source` / `NIMBLE_PREFIX_SOURCE`  (optional)

`cfg.prefix_source` (string) is a CouchDB prefix (a base URI with no database name in the path) for the _central_ (master) database (so that we can start replications from it). Default value: the value of the `NIMBLE_PREFIX_SOURCE` environment variable.

      {prefix_source} = cfg
      prefix_source ?= process.env.NIMBLE_PREFIX_SOURCE

If neither is present, replication will not work.

### `prefix_upload` / `NIMBLE_PREFIX_UPLOAD`  (optional)

      {prefix_upload} = cfg
      prefix_upload ?= process.env.NIMBLE_PREFIX_UPLOAD

### `prov_master_admin` / `NIMBLE_PROV_MASTER_ADMIN`  (optional)

`cfg.prov_master_admin` (string or array) are URIs to the master provisioning database(s) with admin access.
If it is not present in the configuration, the value of the `NIMBLE_PROV_MASTER_ADMIN` environment variable is used.

      {prov_master_admin} = cfg
      prov_master_admin ?= process.env.NIMBLE_PROV_MASTER_ADMIN

If neither is present, the `master_push` function will not work.

`master_push`
---------------

`master_push(doc)`: push a document on the master provisioning database.
Typically used to push a design document so that we can filter for replication.

      prov_masters = []
      masters = prov_master_admin
      if masters?
        if typeof masters is 'string'
          masters = [masters]
        prov_masters = masters.map (name) ->
          new CouchDB name

      master_push = (doc) ->
        debug "Updating master design document #{doc._id}"
        Promise.all prov_masters.map (master) ->
          master.update doc

`push`
------

Push a document on the local provisioning database.
Typically used to push a design document so that we can query.

      push = (doc) ->
        debug "Pushing #{doc._id}"
        (new CouchDB provisioning).update doc

`replicate`
-----------

`replicate(name,extensions_cb)`: replicate database `name` from `prefix_source` to `prefix_admin`.
Before submission, the replication document is passed to the (optional) `extensions_cb` callback.
TODO: allow Array for prefix_source so that we can replicate from a multi-master database.

The one thing we know doesn't work is using the same document ID for documents that describe different replications (e.g. with different filters: experience shows the replicator doesn't notice and keeps using the old filter).

cfg.prefix_source (string, URI) — prefix used to build URI of source CouchDB databases when replicating downstream (e.g. from a master provisioning server)

      replicate = (name,extensions,again = 2, delay = 503) ->
        unless prefix_source?
          debug "Warning: `replicate` is missing `prefix_source (ignored)"
          return

        try
          await Replicator prefix_source, prefix_admin, name, extensions
          return true

        catch error
          debug "replicate #{name}: #{error.stack ? JSON.stringify error}"

          if again > 0
            await sleep delay+delay*0.5*Math.random()
            await replicate name, extensions, again-1, delay*1.5646
          else
            debug "replicate #{name}: Too many errors, giving up."
            return false

cfg.prefix_upload (string, URI) — prefix used to build URI of target CouchDB databases when replicating upstream (e.g. towards a CDR aggregation server)

      replicate_up = (name,group_name) ->
        unless prefix_upload?
          debug "Warning: `replicate_up` is missing `prefix_upload` (ignored)"
          return

        try
          await Replicator prefix_admin, prefix_upload, name, null, group_name
          return true

        catch error
          debug "replicate_up #{name}: #{error.stack ? JSON.stringify error}"
          return false

cfg.provisioning (string,URI) — URL to the local provisioning URI. Default: the value of the `NIMBLE_PROVISIONING` environment variable, otherwise a database called `provisioning` in the `cfg.prefix_admin` server.

      {provisioning} = cfg
      provisioning ?= process.env.NIMBLE_PROVISIONING
      provisioning ?= "#{prefix_admin}/provisioning"

      inject = (source) -> (db) ->
        {_id} = source
        doc = await db
          .get _id
          .catch -> {_id}
        for own k,v of source
          doc[k] = v
        await db.put doc
        return

      {
      reject_tombstones: inject reject_tombstones
      reject_types: inject reject_types
      prefix_admin
      prefix_source
      prefix_upload
      prov_master_admin
      master_push
      push
      replicate
      replicate_up
      provisioning
      }

Toolbox
=======

    CouchDB = require 'most-couchdb/with-update'

    reject_tombstones = require 'reject-tombstones'
    reject_types = require './reject-types'

    Replicator = require 'frantic-team'
    assert = require 'assert'
    debug = (require 'tangible') 'nimble-direction'
