Deployment at USDF
==================
First the values file in Phalanx was created namely 'values-usdfprod.yaml <https://github.com/lsst-sqre/phalanx/blob/162905690a25309bc916f1304fc785c472f0e9ae/applications/consdb/values-usdf.yaml>'

'Secret was created  in the USDF vault <https://vault.slac.stanford.edu/ui/vault/secrets/secret/show/rubin/usdf-rsp/consdb>' with values from base.

Set ConsDB to true in the phalanx environments values-usdfprod.yaml.

Seems database schema exists already.

Also found the S3_ENDPOINT_URL was not being set so this was added to values and assigned in usdfprod and base.
