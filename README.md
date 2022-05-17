# Relation engine import benchmarking

## Testing FastANI relations between 90k genomes as edges

Data source: http://enve-omics.ce.gatech.edu/data/fastani

FastANI matrix, NCBI_Prok-matrix.txt.gz

~1B edges, ~90k verts

Max ArangoDB key length is [254 chars](https://github.com/arangodb/arangodb/issues/10754), so check max vert size:

```
$ ipython
Python 3.6.9 (default, Mar 15 2022, 13:55:28) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: maxlen = 0

In [2]: import gzip

# A few aborted attempts here

In [14]: matrix = gzip.open('NCBI_Prok-matrix.txt.gz', 'rt')

In [15]: for line in matrix: 
    ...:     name1, name2, percid = line.split(" ") 
    ...:     if len(name1) > maxlen: 
    ...:         print("1", name1, len(name1), maxlen) 
    ...:         maxlen = len(name1) 
    ...:     if len(name2) > maxlen:
    ...:         print("2", name2, len(name2), maxlen) 
    ...:         maxlen = len(name2) 
    ...:
1 Lactobacillus_plantarum_subsp__plantarum_ATCC_14917___JCM_1149___CGMCC_1_2437_GCA_000143745.LargeContigs.fna 108 105
1 Salmonella_enterica_subsp__enterica_serovar_Montevideo_str__USDA_ARS_USMARC_1903_NZ_CP007222.LargeContigs.fna 109 108
1 Salmonella_enterica_subsp__enterica_serovar_Typhimurium_str__LT2_4_delta_ramA__kan_GCA_000336195.LargeContigs.fna 113 109

In [16]:

```

So if we concatenate vert names to make edge keys it should be ok - 113 * 2 + 1 = 227

Loading into a collection with shards = 3, replication factor = 1, standard indexes (`_key` and `(_from, _to)`) on the CI cluster (3 nodes)

At the time of writing, the 3 nodes all write to the same drives I think (?).

* TODO retest when nodes write to separate drives.

Since the matrix has no headers, make a headers file:

```
$ cat NCBI_Prok-matrix.txt.gz.headers.txt 
from to idscore
```

A working incantation of `arangoimport`: 
```
$ ~/arango/3.9.1/bin/arangoimport
    --file NCBI_Prok-matrix.txt.gz
    --type csv
    --collection FastANI
    --merge-attributes key=[from]_[to]
    --headers-file NCBI_Prok-matrix.txt.gz.headers.txt
    --progress true
    --server.endpoint tcp://localhost:48000
    --server.username gavin
    --server.password $ARANGO_PWD_CI
    --server.database gavin_test
    --separator " "
    --translate "from=_from"
    --translate "to=_to"
    --translate "key=_key"
    --from-collection-prefix node
    --to-collection-prefix node
Connected to ArangoDB 'http+tcp://127.0.0.1:48000, version: 3.9.0, database: 'gavin_test',
username: 'gavin'
----------------------------------------
database:               gavin_test
collection:             FastANI
from collection prefix: node
to collection prefix:   node
create:                 no
create database:        no
source filename:        NCBI_Prok-matrix.txt.gz
file type:              csv
quote:                  "
separator:               
headers file:           NCBI_Prok-matrix.txt.gz.headers.txt
threads:                6
on duplicate:           error
connect timeout:        5
request timeout:        1200
----------------------------------------
Starting CSV import...
```

