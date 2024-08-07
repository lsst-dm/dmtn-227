Making the schema
================= 

Eventually Felis will do the complete creation of the schema directly. 
For now we are running it to produce a file. 
That file is slightly processed and then used to create the schema in the different databases.

Clone <git@github.com:lsst/felis.git> and install it with:

.. code-block::

  pip install .


Clone 'git@github.com:lsst/sdm_schemas.git'

The LATISS part of ConsDB schema is in 'yml/summit-latiss.yaml'.


To produce the DDL run Felis in "dry-run" mode as follows:


.. code-block::

  felis create-all --engine-url='postgresql://butler:nopass@postgresdb01:5432/butler' --dry-run yml/summit-latiss.yaml  >> tmp.sql

Felis puts quotes on names which make them case sensitive in postgres; we remove those with:

.. code-block::

   yml/post.sh  tmp.sql  >> summit-latiss.sql


Now we have the schema file we can use in any of the DBs.
Note that the ``cdb_latiss`` schema is dropped and recreated by this script.







