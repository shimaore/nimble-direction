nimble-direction
================

This module extends the `cfg` object with CCNQ4-related tools.

Usage
-----

```coffeescript
Nimble = require 'nimble-direction'

cfg = require 'config.json'
Nimble cfg
.then ->
  run cfg
```

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    module.exports = (cfg) ->

Configuration variables
-----------------------

      debug "Configuring #{pkg.name} version #{pkg.version}."

### `prefix_admin` / `NIMBLE_PREFIX_ADMIN`  (required)

`.prefix_admin` (string) is a CouchDB prefix (a base URI with no database name in the path) with admin access to the _local_ CouchDB instance.
If it is not present in the configuration, the value of the `NIMBLE_PREFIX_ADMIN` environment variable is used.

      cfg.prefix_admin ?= process.env.NIMBLE_PREFIX_ADMIN

One of the environment variable or the `.prefix_admin` configuration item is required.

      assert cfg.prefix_admin?, 'Missing prefix_admin'

### `prefix_source` / `NIMBLE_PREFIX_SOURCE`  (optional)

`.prefix_source` (string) is a CouchDB prefix (a base URI with no database name in the path) for the _central_ (master) database (so that we can start replications from it).
If it is not present in the configuration, the value of the `NIMBLE_PREFIX_SOURCE` environment variable is used.

      cfg.prefix_source ?= process.env.NIMBLE_PREFIX_SOURCE

If neither is present, replication will not work.

### `prov_master_admin` / `NIMBLE_PROV_MASTER_ADMIN`  (optional)

`.prov_master_admin` (string or array) are URIs to the master provisioning database(s) with admin access.
If it is not present in the configuration, the value of the `NIMBLE_PROV_MASTER_ADMIN` environment variable is used.

      cfg.prov_master_admin ?= process.env.NIMBLE_PROV_MASTER_ADMIN

If neither is present, the `master_push` function will not work.

`master_push`
---------------

`master_push(doc)`: push a document on the master provisioning database.
Typically used to push a design document so that we can filter for replication.

      prov_masters = []
      masters = cfg.prov_master_admin
      if masters?
        if typeof masters is 'string'
          masters = [masters]
        prov_masters = masters.map (name) ->
          new CouchDB name

      cfg.master_push = (doc) ->
        debug "Updating master design document #{doc._id}"
        Promise.all prov_masters.map (master) ->
          update master, doc

`push`
------

Push a document on the local provisioning database.
Typically used to push a design document so that we can query.

      cfg.push = (doc) ->
        debug "Pushing #{doc._id}"
        update (new CouchDB cfg.provisioning), doc

`replicate`
-----------

`replicate(name,extensions_cb)`: replicate database `name` from `prefix_source` to `prefix_admin`.
Before submission, the replication document is passed to the (optional) `extensions_cb` callback.
TODO: allow Array for prefix_source so that we can replicate from a multi-master database.

The one thing we know doesn't work is using the same document ID for documents that describe different replications (e.g. with different filters: experience shows the replicator doesn't notice and keeps using the old filter).

      cfg.replicate = replicate = (name,extensions,again = 2, delay = 503) ->
        unless cfg.prefix_source?
          debug "Warning: `replicate` called in standalone NIMBLE_MODE (ignored)"
          return

        try
          await Replicator cfg.prefix_source, cfg.prefix_admin, name, extensions
          return true

        catch error
          debug "replicator #{name}: #{error.stack ? JSON.stringify error}"

          if again > 0
            await sleep delay+delay*0.5*Math.random()
            await replicate name, extensions, again-1, delay*1.5646
          else
            debug "replicator #{name}: Too many errors, giving up."
            return false

`prov`
------

A PouchDB instance to the local provisioning database.

      cfg.provisioning ?= process.env.NIMBLE_PROVISIONING
      cfg.provisioning ?= "#{cfg.prefix_admin}/provisioning"

      inject = (source) -> (db) ->
        {_id} = source
        doc = await db
          .get _id
          .catch -> {_id}
        for own k,v of source
          doc[k] = v
        await db.put doc
        return

      cfg.reject_tombstones = inject reject_tombstones
      cfg.reject_types = inject reject_types

      Promise.resolve cfg

Toolbox
=======

    CouchDB = require 'most-couchdb'

    reject_tombstones = require 'reject-tombstones'
    reject_types = require './reject-types'

    Replicator = require 'frantic-team'
    crypto = require 'crypto'
    assert = require 'assert'
    url = require 'url'
    pkg = require './package.json'
    debug = (require 'tangible') "#{pkg.name}:config"
    update = require './update'
