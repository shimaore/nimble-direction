`update`
--------

Overwrite a document in a PouchDB database.

    update = (db,doc) ->
      debug "Update #{doc._id}"
      db.get doc._id
      .catch (error) ->
        debug "#{doc._id}: #{error} (ignored)"
        {}
      .then ({_rev}) ->
        doc._rev = _rev if _rev?
        db.put doc
      .catch (error) ->
        debug "#{doc._id}: #{error}"
        Promise.reject error

    module.exports = update
    pkg = require './package.json'
    debug = (require 'debug') "#{pkg.name}:update"
