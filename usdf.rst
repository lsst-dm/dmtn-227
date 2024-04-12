Deployment at USDF
==================
First the values file in Phalanx was created namely `values-usdfprod.yaml <https://github.com/lsst-sqre/phalanx/blob/63377b6ae3d8661589ebf4c019232982785c8ec6/applications/consdb/values-usdfprod.yaml>`

Secret was created  in `the USDF vault <https://vault.slac.stanford.edu/ui/vault/secrets/secret/show/rubin/usdf-rsp/consdb>` with values from base.

Set ConsDB to true in the phalanx environments values-usdfprod.yaml.

Seems database schema exists already.

Also found the S3_ENDPOINT_URL was not being set so this was added to values and assigned in usdfprod and base.
