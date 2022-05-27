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

Due to [this bug](https://github.com/arangodb/arangodb/issues/16337), we need to explicitly
add a `_key` to the file rather than using `arangoimport --merge-attributes`:

```
$ ipython
Python 3.6.9 (default, Mar 15 2022, 13:55:28) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: def add_key(infile, outfile): 
   ...:     with open(infile) as inf, open(outfile, 'w') as outf: 
   ...:         for line in inf: 
   ...:             n1, n2, score = line.split(',') 
   ...:             outf.write(line.strip() + f',{n1}_{n2}\n') 
   ...:                                                                         

In [2]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.key.txt')            

In [3]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.key.txt')           

In [4]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.key.txt')           

In [5]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.key.txt')           

In [6]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.key.txt')           

In [7]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.key.txt')           

In [8]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.key.txt')           

In [9]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.txt', 
   ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.key.txt')           

In [10]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.txt', 
    ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.key.txt')          

In [11]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.txt', 
    ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.key.txt')         

In [12]: add_key('NCBI_Prok-matrix.txt.gz.GCAonly.head100M.txt', 
    ...:         'NCBI_Prok-matrix.txt.gz.GCAonly.head100M.key.txt') 
```

New headers file:
```
root@194be4682dd0:/arangobenchmark# cat data/NCBI_Prok-matrix.txt.gz.headers.GCA.key.txt 
_from,_to,idscore,_key
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
    --headers-file data/NCBI_Prok-matrix.txt.gz.headers.GCA.key.txt \
    --type csv \
    --separator "," \
    --progress true \
    --server.endpoint tcp://10.58.1.211:8531 \
    --server.username gavin \
    --server.password $ARANGO_PWD_CI \
    --server.database gavin_test \
    --collection FastANI \
    --log.foreground-tty true \
    --from-collection-prefix node \
    --to-collection-prefix node \
    --threads $THREADS
```

```
root@194be4682dd0:/arangobenchmark# ipython
Python 3.9.5 (default, May  4 2021, 18:15:18) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.23.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import os
In [2]: import arango
In [3]: import subprocess
In [4]: import time

In [5]: def run_imports(files, threads):
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

In [6]: files = !ls data/*GCAonly.head*-*.key.txt

In [7]: ret = run_imports(files, 10)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.key.txt***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.key.txt***

In [8]: ret
Out[8]: 
[{'time': 82.16389203071594, 'disk': 852623552, 'index': 1215503619},
 {'time': 91.71568083763123, 'disk': 1293238119, 'index': 1880682559},
 {'time': 92.58307576179504, 'disk': 1798231480, 'index': 2536598084},
 {'time': 98.12529063224792, 'disk': 2233294282, 'index': 3133633538},
 {'time': 101.78906798362732, 'disk': 2600135670, 'index': 3587923848},
 {'time': 102.4697265625, 'disk': 2845676673, 'index': 3908386172},
 {'time': 128.71135234832764, 'disk': 3267892116, 'index': 4348325344},
 {'time': 100.68136262893677, 'disk': 3932783598, 'index': 4941000120},
 {'time': 101.79515099525452, 'disk': 4335324319, 'index': 5246917869},
 {'time': 109.03617691993713, 'disk': 4741192441, 'index': 5793021687}]

In [9]: def print_res(data):
   ...:     print('|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|')
   ...:     print('|---|---|---|---|')
   ...:     docs = 10
   ...:     cumtime = 0
   ...:     for line in data:
   ...:         cumtime += line['time']
   ...:         print(f"|{docs}|{cumtime}|{line['disk']}|{line['index']}|")
   ...:         docs += 10
   ...: 

In [10]: print_res(ret)
|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|82.16389203071594|852623552|1215503619|
|20|173.87957286834717|1293238119|1880682559|
|30|266.4626486301422|1798231480|2536598084|
|40|364.58793926239014|2233294282|3133633538|
|50|466.37700724601746|2600135670|3587923848|
|60|568.8467338085175|2845676673|3908386172|
|70|697.5580861568451|3267892116|4348325344|
|80|798.2394487857819|3932783598|4941000120|
|90|900.0345997810364|4335324319|5246917869|
|100|1009.0707767009735|4741192441|5793021687|
```

#### Threads: 1
```
[{'time': 256.25492453575134, 'disk': 828417841, 'index': 1145348890},
 {'time': 257.696035861969, 'disk': 1180248939, 'index': 1807144473},
 {'time': 264.3402614593506, 'disk': 1549957700, 'index': 2317748837},
 {'time': 260.30645632743835, 'disk': 1984488947, 'index': 2792429148},
 {'time': 263.5722382068634, 'disk': 2342460871, 'index': 3223781087},
 {'time': 268.8849856853485, 'disk': 2483976751, 'index': 3530840184},
 {'time': 265.08757162094116, 'disk': 2908427483, 'index': 4141378629},
 {'time': 268.65593338012695, 'disk': 3280267354, 'index': 4313201240},
 {'time': 268.33949518203735, 'disk': 3709683286, 'index': 4865901742},
 {'time': 269.58896684646606, 'disk': 4131381842, 'index': 5366974372}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|256.25492453575134|828417841|1145348890|
|20|513.9509603977203|1180248939|1807144473|
|30|778.2912218570709|1549957700|2317748837|
|40|1038.5976781845093|1984488947|2792429148|
|50|1302.1699163913727|2342460871|3223781087|
|60|1571.0549020767212|2483976751|3530840184|
|70|1836.1424736976624|2908427483|4141378629|
|80|2104.7984070777893|3280267354|4313201240|
|90|2373.1379022598267|3709683286|4865901742|
|100|2642.7268691062927|4131381842|5366974372|


#### Threads: 10
```
[{'time': 82.16389203071594, 'disk': 852623552, 'index': 1215503619},
 {'time': 91.71568083763123, 'disk': 1293238119, 'index': 1880682559},
 {'time': 92.58307576179504, 'disk': 1798231480, 'index': 2536598084},
 {'time': 98.12529063224792, 'disk': 2233294282, 'index': 3133633538},
 {'time': 101.78906798362732, 'disk': 2600135670, 'index': 3587923848},
 {'time': 102.4697265625, 'disk': 2845676673, 'index': 3908386172},
 {'time': 128.71135234832764, 'disk': 3267892116, 'index': 4348325344},
 {'time': 100.68136262893677, 'disk': 3932783598, 'index': 4941000120},
 {'time': 101.79515099525452, 'disk': 4335324319, 'index': 5246917869},
 {'time': 109.03617691993713, 'disk': 4741192441, 'index': 5793021687}]
 ```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|82.16389203071594|852623552|1215503619|
|20|173.87957286834717|1293238119|1880682559|
|30|266.4626486301422|1798231480|2536598084|
|40|364.58793926239014|2233294282|3133633538|
|50|466.37700724601746|2600135670|3587923848|
|60|568.8467338085175|2845676673|3908386172|
|70|697.5580861568451|3267892116|4348325344|
|80|798.2394487857819|3932783598|4941000120|
|90|900.0345997810364|4335324319|5246917869|
|100|1009.0707767009735|4741192441|5793021687|

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
