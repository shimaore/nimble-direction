`update`
--------

Overwrite a document in a PouchDB database.

    update = (db,doc) ->
      db.get doc._id
      .catch (error) ->
        debug "#{error} (ignored)"
        {}
      .then ({_rev}) ->
        doc._rev = _rev if _rev?
        db.put doc
      .catch (error) ->
        debug "#{error}"
        throw error

    module.exports = update
    pkg = require './package.json'
    debug = (require 'debug') "#{pkg.name}:update"
