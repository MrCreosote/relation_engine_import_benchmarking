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

Loading into a collection with shards = 3, replication factor = 1, standard indexes
(`_key` and `(_from, _to)`) on the CI cluster (3 nodes)

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
**Notes:**
* This invocation causes all the logs from `arangoimport` to be lost if redirected to a file
* When this was run the 3 CI db servers all wrote to the same NFS drive, which has since been
  fixed

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

In [8]: ret = run_imports(files, 1)
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

In [9]: ret
Out[9]: 
[{'time': 269.27555799484253, 'disk': 939411901, 'index': 1014359016},
 {'time': 260.0713520050049, 'disk': 1349245807, 'index': 1611683301},
 {'time': 262.94750022888184, 'disk': 1631814164, 'index': 2282676808},
 {'time': 263.59321808815, 'disk': 1834457317, 'index': 2832430175},
 {'time': 270.49464535713196, 'disk': 2301898073, 'index': 3287005881},
 {'time': 265.1101965904236, 'disk': 2715813051, 'index': 3667140778},
 {'time': 271.66647124290466, 'disk': 3083087581, 'index': 4052590502},
 {'time': 273.58852100372314, 'disk': 3442726185, 'index': 4623478467},
 {'time': 268.69639348983765, 'disk': 3761069220, 'index': 4958904221},
 {'time': 268.5925540924072, 'disk': 4175970449, 'index': 5417221812}]


In [11]: def print_res(data):
    ...:     print('|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B
    ...: )|Cumulative index size (B)|')
    ...:     print('|---|---|---|---|')
    ...:     docs = 10
    ...:     cumtime = 0
    ...:     for line in data:
    ...:         cumtime += line['time']
    ...:         print(f"|{docs}|{cumtime}|{line['disk']}|{line['index']}|")
    ...:         docs += 10
    ...: 
   ...: 
    ...: 

In [12]: print_res(ret)
|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|269.27555799484253|939411901|1014359016|
|20|529.3469099998474|1349245807|1611683301|
|30|792.2944102287292|1631814164|2282676808|
|40|1055.8876283168793|1834457317|2832430175|
|50|1326.3822736740112|2301898073|3287005881|
|60|1591.4924702644348|2715813051|3667140778|
|70|1863.1589415073395|3083087581|4052590502|
|80|2136.7474625110626|3442726185|4623478467|
|90|2405.4438560009003|3761069220|4958904221|
|100|2674.0364100933075|4175970449|5417221812|

```

#### Threads: 1
```
[{'time': 269.27555799484253, 'disk': 939411901, 'index': 1014359016},
 {'time': 260.0713520050049, 'disk': 1349245807, 'index': 1611683301},
 {'time': 262.94750022888184, 'disk': 1631814164, 'index': 2282676808},
 {'time': 263.59321808815, 'disk': 1834457317, 'index': 2832430175},
 {'time': 270.49464535713196, 'disk': 2301898073, 'index': 3287005881},
 {'time': 265.1101965904236, 'disk': 2715813051, 'index': 3667140778},
 {'time': 271.66647124290466, 'disk': 3083087581, 'index': 4052590502},
 {'time': 273.58852100372314, 'disk': 3442726185, 'index': 4623478467},
 {'time': 268.69639348983765, 'disk': 3761069220, 'index': 4958904221},
 {'time': 268.5925540924072, 'disk': 4175970449, 'index': 5417221812}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|269.27555799484253|939411901|1014359016|
|20|529.3469099998474|1349245807|1611683301|
|30|792.2944102287292|1631814164|2282676808|
|40|1055.8876283168793|1834457317|2832430175|
|50|1326.3822736740112|2301898073|3287005881|
|60|1591.4924702644348|2715813051|3667140778|
|70|1863.1589415073395|3083087581|4052590502|
|80|2136.7474625110626|3442726185|4623478467|
|90|2405.4438560009003|3761069220|4958904221|
|100|2674.0364100933075|4175970449|5417221812|


#### Threads: 10
```
[{'time': 87.05017137527466, 'disk': 2236442662, 'index': 1473379920},
 {'time': 93.59703421592712, 'disk': 2676066966, 'index': 2070743359},
 {'time': 99.17555260658264, 'disk': 1740117248, 'index': 2849776607},
 {'time': 99.67172241210938, 'disk': 2187804601, 'index': 3387705579},
 {'time': 103.1408953666687, 'disk': 2506598304, 'index': 3751101951},
 {'time': 104.7887213230133, 'disk': 3040381345, 'index': 4364063801},
 {'time': 103.23674011230469, 'disk': 3542169062, 'index': 4600724608},
 {'time': 104.69807362556458, 'disk': 3755623946, 'index': 4941651533},
 {'time': 105.69861245155334, 'disk': 4116761506, 'index': 5409674133},
 {'time': 108.95327091217041, 'disk': 4513367122, 'index': 5841878226}]

 ```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|87.05017137527466|2236442662|1473379920|
|20|180.64720559120178|2676066966|2070743359|
|30|279.8227581977844|1740117248|2849776607|
|40|379.4944806098938|2187804601|3387705579|
|50|482.6353759765625|2506598304|3751101951|
|60|587.4240972995758|3040381345|4364063801|
|70|690.6608374118805|3542169062|4600724608|
|80|795.3589110374451|3755623946|4941651533|
|90|901.0575234889984|4116761506|5409674133|
|100|1010.0107944011688|4513367122|5841878226|

### Imports with full strain names

* TODO redo this, right now it's out of date

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
