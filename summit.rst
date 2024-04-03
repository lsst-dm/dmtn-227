Deployment at Summit
====================
Set consdb to true in the phalanx environments values-summit.yaml.

Created values file in Phalanx 'values-summit.yaml <https://github.com/lsst-sqre/phalanx/blob/162905690a25309bc916f1304fc785c472f0e9ae/applications/consdb/values-usdf.yaml>'

Secret added  in summit  vault 'summit-lsp/consdb <https://vault.lsst.cloud/ui/vault/secrets/secret/kv/k8s_operator%2Fsummit-lsp.lsst.codes%2Fconsdb/details?version=1> 

Create Schema
-------------
For consitancy the schema file used at the base generated from Felis and post processed with KT's script was used.

With Summit VPN connected:

 psql  -h postgresdb01.cp.lsst.org -U oods butler 
 butler=> \i summit-lattis.sql
 \q



done.

