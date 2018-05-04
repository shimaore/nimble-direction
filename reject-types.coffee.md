reject-types
============

Instead of using a (intrinsequally slow) replication filter, it is generally better to reject the unwanted documents in the receiving database.

The following is the known list of provisioning document types used by CCNQ4 voice servers that should be accepted.

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

    module.exports =
      _id: '_design/reject_types'
      validate_doc_update: """
        (function(doc) {
          var type_from_id = doc._id.split(':')[0];
          if( type_from_id.match(/^(?:#{default_replicate_types.join '|'})$/) ) {
            return;
          }
          throw {forbidden: 'Document type rejected'};
        })
      """
