Deployment at Summit
====================
Set consdb to true in the phalanx environments values-summit.yaml.

Created values file in Phalanx `values-summit.yaml <https://github.com/lsst-sqre/phalanx/blob/63377b6ae3d8661589ebf4c019232982785c8ec6/applications/consdb/values-summit.yaml>`

Secret added  in summit  vault 'summit-lsp/consdb <https://vault.lsst.cloud/ui/vault/secrets/secret/kv/k8s_operator%2Fsummit-lsp.lsst.codes%2Fconsdb/details?version=1> 

Create Schema
-------------
For consistency the schema file used at the base generated from Felis and post-processed with the ``post.sh`` script was used.

With Summit VPN connected:

.. code-block::

  psql  -h postgresdb01.cp.lsst.org -U oods butler 
  butler=> \i summit-latiss.sql
  \q



done.

