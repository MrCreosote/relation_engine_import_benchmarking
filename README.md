# Relation engine import benchmarking

## Testing FastANI relations between 90k genomes as edges

Data source: http://enve-omics.ce.gatech.edu/data/fastani

FastANI matrix, NCBI_Prok-matrix.txt.gz

~1B edges, ~90k verts

### Check key = fn(name1, name2) isn't too large for Arango

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

### Cluster config

Loading into a collection with shards = 3, replication factor = 1, standard indexes (`_key` and `(_from, _to)`) on the CI cluster (3 nodes)

At the time of writing, the 3 nodes all write to the same drives I think (?).

* TODO retest when nodes write to separate drives.

### File set up

Since the matrix has no headers, make a headers file:

```
$ cat NCBI_Prok-matrix.txt.gz.headers.txt 
from to idscore
```

or for CSV:
```
root@3b864ab69fe5:/arangobenchmark# cat data/NCBI_Prok-matrix.txt.gz.headers.GCA.txt 
from,to,idscore
```

To reduce edge sizes, extract just the GCA ids from the names:

```
In [1]: import gzip

In [6]: with gzip.open('NCBI_Prok-matrix.txt.gz', 'rt') as infile, open('NCBI_Pr
   ...: ok-matrix.txt.gz.GCAonly.head100M.txt', 'wt') as outfile: 
   ...:     counter = 0 
   ...:     for line in infile: 
   ...:         name1, name2, score = line.split() 
   ...:         if '_GCA_' in name1 and '_GCA_' in name2: 
   ...:             id1 = name1.split('_GCA_')[1].split('.')[0] 
   ...:             id2 = name2.split('_GCA_')[1].split('.')[0] 
   ...:             outfile.write(f'GCA_{id1},GCA_{id2},{score}\n') 
   ...:             counter += 1 
   ...:         if counter >= 100000000: 
   ...:             break 
```

Check the ids didn't get munged weirdly:
```
In [9]: with open('NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt') as infile: 
   ...:     for line in infile: 
   ...:         name1, name2, score = line.split(',') 
   ...:         if (len(name1) != 13 or len(name2) != 13): 
   ...:             print(line) 
   ...:    

In [10]:  
```

Split the file into 10M line chunks:
```
$ head -10000000 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt > NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.txt 
$ tail +10000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.txt
$ tail +20000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.txt
$ tail +30000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.txt
$ tail +40000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.txt
$ tail +50000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.txt
$ tail +60000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.txt
$ tail +70000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.txt
$ tail +80000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.txt
$ tail +90000001 NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.txt
```

### Test run with GCA names, 100M edges

Loading 100M smaller (~180B as Arango docs) edges into Arango from a docker container on docker03:

```
root@3b864ab69fe5:/arangobenchmark# cat import.GCA.100M.sh 
#!/usr/bin/env sh

date
arango/3.9.1/bin/arangoimport \
    --file data/NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt \
    --headers-file data/NCBI_Prok-matrix.txt.gz.headers.GCA.txt \
    --type csv \
    --separator "," \
    --progress true \
    --server.endpoint tcp://10.58.1.211:8531 \
    --server.username gavin \
    --server.password $ARANGO_PWD_CI \
    --server.database gavin_test \
    --collection FastANI \
    --merge-attributes key=[from]_[to] \
    --translate "from=_from" \
    --translate "to=_to" \
    --translate "key=_key" \
    --from-collection-prefix node \
    --to-collection-prefix node \
    --threads 10
date
```
**Note:** This invocation causes all the logs from `arangoimport` to be lost if redirected to a
file

Example run:
```
root@3b864ab69fe5:/arangobenchmark# ./import.GCA.100M.sh > import_log.out 2> import_log.err &

root@3b864ab69fe5:/arangobenchmark# cat import_log.out 
Sat May 21 02:35:42 UTC 2022
Connected to ArangoDB 'http+tcp://10.58.1.211:8531, version: 3.9.0, database: 'gavin_test', username: 'gavin'
----------------------------------------
database:               gavin_test
collection:             FastANI
from collection prefix: node
to collection prefix:   node
create:                 no
create database:        no
source filename:        data/NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt
file type:              csv
quote:                  "
separator:              ,
headers file:           data/NCBI_Prok-matrix.txt.gz.headers.GCA.txt
threads:                10
on duplicate:           error
connect timeout:        5
request timeout:        1200
----------------------------------------
Starting CSV import...

created:          99999389
warnings/errors:  611
updated/replaced: 0
ignored:          0
lines read:       100000001
Sat May 21 03:07:52 UTC 2022
root@3b864ab69fe5:/arangobenchmark# ls -l import_log.*
-rw-r--r-- 1 root root   0 May 21 02:35 import_log.err
-rw-r--r-- 1 root root 905 May 21 03:07 import_log.out

```

### Import 100M GCA names in 10M batches recording time & disk space
```
root@194be4682dd0:/arangobenchmark# cat import.parameterized.GCAonly.sh 
#!/usr/bin/env sh

arango/3.9.1/bin/arangoimport \
    --file $INFILE \
    --headers-file data/NCBI_Prok-matrix.txt.gz.headers.GCA.txt \
    --type csv \
    --separator "," \
    --progress true \
    --server.endpoint tcp://10.58.1.211:8531 \
    --server.username gavin \
    --server.password $ARANGO_PWD_CI \
    --server.database gavin_test \
    --collection FastANI \
    --log.foreground-tty true \
    --merge-attributes key=[from]_[to] \
    --translate "from=_from" \
    --translate "to=_to" \
    --translate "key=_key" \
    --from-collection-prefix node \
    --to-collection-prefix node \
    --threads $THREADS
```

```
root@60d48dd5ffeb:/arangobenchmark# ipython
Python 3.9.5 (default, May  4 2021, 18:15:18) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.23.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import os
In [2]: import arango
In [3]: import subprocess
In [4]: import time

In [6]: files = !ls data/*GCAonly.head*-*

In [10]: def run_imports(files, threads):
    ...:     pwd = os.environ['ARANGO_PWD_CI']
    ...:     acli = arango.ArangoClient(hosts='http://10.58.1.211:8531')
    ...:     db = acli.db('gavin_test', username='gavin', password=pwd)
    ...:     col = db.collection('FastANI')
    ...:     ret = []
    ...:     for f in files:
    ...:         print("***" + f + "***")
    ...:         t1 = time.time()
    ...:         res = subprocess.run(
    ...:             './import.parameterized.GCAonly.sh',
    ...:             capture_output=True,
    ...:             env={
    ...:                 'ARANGO_PWD_CI': pwd,
    ...:                 'INFILE': f,
    ...:                 'THREADS': str(threads)
    ...:                 }
    ...:             )
    ...:         t = time.time() - t1
    ...:         if (res.returncode > 0):
    ...:             print("stdout")
    ...:             print(res.stdout)
    ...:             print("stderr")
    ...:             print(res.stderr)
    ...:         with open(f + ".out", 'wb') as logout:
    ...:             logout.write(res.stdout)
    ...:         stats = col.statistics()
    ...:         ret.append({
    ...:             'time': t,
    ...:             'disk': stats['documents_size'],
    ...:             'index': stats['indexes']['size']
    ...:             })
    ...:     return ret
    ...: 

In [11]: run_imports(files, 10)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.txt***
Out[11]: 
[{'time': 100.69085884094238, 'disk': 865253249, 'index': 1245316500},
 {'time': 93.10041427612305, 'disk': 1372181325, 'index': 1971201257},
 {'time': 96.4419322013855, 'disk': 2221664182, 'index': 2381061635},
 {'time': 95.80396676063538, 'disk': 2878582225, 'index': 2894953636},
 {'time': 98.68604755401611, 'disk': 3624084773, 'index': 3381993682},
 {'time': 97.59403681755066, 'disk': 4085427477, 'index': 4013655626},
 {'time': 99.40194582939148, 'disk': 4760989627, 'index': 4485422082},
 {'time': 100.70587682723999, 'disk': 5434036061, 'index': 4787867305},
 {'time': 100.35941863059998, 'disk': 6080331386, 'index': 5241744374},
 {'time': 100.36307787895203, 'disk': 6945681377, 'index': 5826990294}]

# C&P'd into another ipython instance:

In [8]: def print_res(data): 
   ...:     print('|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)
   ...: |Cumulative index size (B)|') 
   ...:     print('|---|---|---|---|') 
   ...:     docs = 10 
   ...:     cumtime = 0 
   ...:     for line in data: 
   ...:         cumtime += line['time'] 
   ...:         print(f"|{docs}|{cumtime}|{line['disk']}|{line['index']}|") 
   ...:         docs += 10 
   ...:                                                                         

In [9]: print_res(data)                                                         
|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|100.69085884094238|865253249|1245316500|
|20|193.79127311706543|1372181325|1971201257|
|30|290.2332053184509|2221664182|2381061635|
|40|386.0371720790863|2878582225|2894953636|
|50|484.7232196331024|3624084773|3381993682|
|60|582.3172564506531|4085427477|4013655626|
|70|681.7192022800446|4760989627|4485422082|
|80|782.4250791072845|5434036061|4787867305|
|90|882.7844977378845|6080331386|5241744374|
|100|983.1475756168365|6945681377|5826990294|
```

#### Threads: 10
```
[{'time': 100.69085884094238, 'disk': 865253249, 'index': 1245316500},
 {'time': 93.10041427612305, 'disk': 1372181325, 'index': 1971201257},
 {'time': 96.4419322013855, 'disk': 2221664182, 'index': 2381061635},
 {'time': 95.80396676063538, 'disk': 2878582225, 'index': 2894953636},
 {'time': 98.68604755401611, 'disk': 3624084773, 'index': 3381993682},
 {'time': 97.59403681755066, 'disk': 4085427477, 'index': 4013655626},
 {'time': 99.40194582939148, 'disk': 4760989627, 'index': 4485422082},
 {'time': 100.70587682723999, 'disk': 5434036061, 'index': 4787867305},
 {'time': 100.35941863059998, 'disk': 6080331386, 'index': 5241744374},
 {'time': 100.36307787895203, 'disk': 6945681377, 'index': 5826990294}]
 ```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|100.69085884094238|865253249|1245316500|
|20|193.79127311706543|1372181325|1971201257|
|30|290.2332053184509|2221664182|2381061635|
|40|386.0371720790863|2878582225|2894953636|
|50|484.7232196331024|3624084773|3381993682|
|60|582.3172564506531|4085427477|4013655626|
|70|681.7192022800446|4760989627|4485422082|
|80|782.4250791072845|5434036061|4787867305|
|90|882.7844977378845|6080331386|5241744374|
|100|983.1475756168365|6945681377|5826990294|

### Imports with full strain names

* TODO clean this up, right now it's out of date

Make 10M and 100M line files for smaller imports:
```
root@bcd06c631a4b:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.head.10Mlines.txt.gz | wc -l
10000000
root@bcd06c631a4b:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.head.100Mlines.txt.gz | wc -l
100000000
```

Run script:
```
root@bcd06c631a4b:/arangobenchmark# cat import.sh 
#!/usr/bin/env sh

date
arango/3.9.1/bin/arangoimport \
    --file data/NCBI_Prok-matrix.txt.gz.head.100Mlines.txt.gz \
    --headers-file data/NCBI_Prok-matrix.txt.gz.headers.txt \
    --type csv \
    --separator " " \
    --progress true \
    --server.endpoint tcp://10.58.1.211:8531 \
    --server.username gavin \
    --server.password $ARANGO_PWD_CI \
    --server.database gavin_test \
    --collection FastANI \
    --merge-attributes key=[from]_[to] \
    --translate "from=_from" \
    --translate "to=_to" \
    --translate "key=_key" \
    --from-collection-prefix node \
    --to-collection-prefix node \
    --threads 10
date
```

Example import run on docker03 in a docker container:
```
root@bcd06c631a4b:/arangobenchmark# cat import_log 
Wed May 18 03:27:32 UTC 2022
Connected to ArangoDB 'http+tcp://10.58.1.211:8531, version: 3.9.0, database: 'gavin_test', username: 'gavin'
----------------------------------------
database:               gavin_test
collection:             FastANI
from collection prefix: node
to collection prefix:   node
create:                 no
create database:        no
source filename:        data/NCBI_Prok-matrix.txt.gz.head.100Mlines.txt.gz
file type:              csv
quote:                  "
separator:               
headers file:           data/NCBI_Prok-matrix.txt.gz.headers.txt
threads:                10
on duplicate:           error
connect timeout:        5
request timeout:        1200
----------------------------------------
Starting CSV import...

created:          73106151
warnings/errors:  221527
updated/replaced: 0
ignored:          0
lines read:       73692769
Wed May 18 04:44:47 UTC 2022
```
