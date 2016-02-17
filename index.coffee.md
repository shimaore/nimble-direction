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

    module.exports = (cfg) ->

Configuration variables
-----------------------

      debug "Configuring #{pkg.name} version #{pkg.version}.", cfg

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
          new PouchDB name, skip_setup: true

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
        update cfg.prov, doc

`replicate`
-----------

`replicate(name,extensions_cb)`: replicate database `name` from `prefix_source` to `prefix_admin`.
Before submission, the replication document is passed to the (optional) `extensions_cb` callback.
TODO: allow Array for prefix_source so that we can replicate from a multi-master database.

      cfg.replicator ?= process.env.NIMBLE_REPLICATOR
      cfg.replicator ?= "#{cfg.prefix_admin}/_replicator"
      replicator = new PouchDB cfg.replicator, skip_setup: true

Here we have multiple solutions, so I'll test them:
- either delete any existing document with the same name (this should cancel the replication, based on the CouchDB docs), and recreate a new one;

      use_delete = true

- or use a different ID for documents that describes different replications.

      # use_delete = false

The one thing we know doesn't work is using the same document ID for documents that describe different replications (e.g. with different filters: experience shows the replicator doesn't notice and keep using the old filter).

      cfg.replicate = (name,extensions) ->
        unless cfg.prefix_source?
          debug "Warning: `replicate` called in standalone NIMBLE_MODE (ignored)"
          return

        source = url.parse cfg.prefix_source
        comment = "replication of #{name} from #{source.host}"
        debug "Going to start #{comment}."

I'm creating a `model` document.. just in case I'd have to revert to manually pushing to `/_replicate` because the replicator is too broken. :)

        model =
          comment: comment
          continuous: true
          target: name

Remove authorization from the source, because...

          source:
            url: url.format
              protocol: source.protocol
              host: source.host
              pathname: name

even with CouchDB 1.6.1 we still have the issue with CouchDB not properly managing authorization headers when a username and password are provided in the original URI that contains "special" characters (like `@` or space). So let's handle it ourselves.

        if source.auth?
          auth = (new Buffer source.auth).toString 'base64'
          debug "Encoded `#{source.auth}` of `#{cfg.prefix_source}` as `#{auth}`."
          model.source.headers =
            Authorization: "Basic #{auth}"

Let the callback add any field they'd like.

        extensions? model

Create a (somewhat) unique ID for the document.

        sum = crypto.createHash 'sha256'
        sum.update JSON.stringify model
        id = sum.digest 'hex'
        model.comment_id = id

When deleting, we can use the `comment` value since it doesn't have to be unique even if we change the record.
When creating documents with different IDs, well, use the computed ID.

        model._id = if use_delete then comment else id

Let's get started.

        Promise.resolve()

Create the target database if it doesn't already exist.

        .then ->
          target = new PouchDB "#{cfg.prefix_admin}/#{name}", skip_setup: false
          target.info()
        .catch (error) ->
          debug "info #{name}: #{error.stack ? error}"
          Promise.reject error

When using the deletion method, first delete the existing replication document.

        .then ->
          if use_delete
            replicator.get model._id
            .catch (error) -> {}
            .then ({_rev}) ->
              replicator.remove model._id, _rev if _rev?
        .catch (error) ->
          debug "remove #{model._id}: #{error.stack ? error}"
          Promise.reject error

Give CouchDB some time to breath.

        .delay 2000

Update the replication document.

        .then ->
          replicator.get model._id
        .catch (error) -> {}
        .then ({_rev}) ->
          doc = {}
          doc._rev = _rev if _rev?
          for own k,v of model
            doc[k] = v

          replicator.put doc

        .catch (error) ->
          debug "put #{model._id}: #{error.stack ? error}"
          if error.status? and error.status is 403
            debug "Replication already started"
            return
          debug "Replication from #{model.source} failed."
          Promise.reject error

`users`
-------

A PouchDB instance to the local users database.

      cfg.users ?= process.env.NIMBLE_USERS
      cfg.users ?= "#{cfg.prefix_admin}/_users"
      if typeof cfg.users is 'string'
        cfg.users = new PouchDB cfg.users, skip_setup: true

`prov`
------

A PouchDB instance to the local provisioning database.

      cfg.provisioning ?= process.env.NIMBLE_PROVISIONING
      cfg.provisioning ?= "#{cfg.prefix_admin}/provisioning"
      cfg.prov ?= new PouchDB cfg.provisioning, skip_setup: true

      Promise.resolve cfg

Toolbox
=======

    PouchDB = (require 'pouchdb').defaults
      ajax:
        forever: true
        timeout: 10000

    Promise = require 'bluebird'
    crypto = require 'crypto'
    assert = require 'assert'
    url = require 'url'
    pkg = require './package.json'
    debug = (require 'debug') "#{pkg.name}:config"
    update = require './update'
