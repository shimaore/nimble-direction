`update`
--------

Overwrite a document in a PouchDB database.

Note that `marked-summer/update-version` might be a better option.

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
        debug "#{doc._id}: #{error.stack ? JSON.stringify error}"
        Promise.reject new Error "update on #{doc._id} failed"

    module.exports = update
    pkg = require './package.json'
    debug = (require 'tangible') "#{pkg.name}:update"
