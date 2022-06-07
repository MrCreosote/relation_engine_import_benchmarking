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

Time to load without chunking: 2654.2085466384888

#### Threads: 2

```
[{'time': 162.50271010398865, 'disk': 1089123555, 'index': 1547559666},
 {'time': 146.5888237953186, 'disk': 1310979922, 'index': 1822402821},
 {'time': 142.462890625, 'disk': 1722240686, 'index': 2374596699},
 {'time': 146.05258965492249, 'disk': 2032917869, 'index': 2908392605},
 {'time': 147.27964425086975, 'disk': 2424082853, 'index': 3432445955},
 {'time': 148.4281177520752, 'disk': 2762769794, 'index': 3817500475},
 {'time': 152.7095890045166, 'disk': 3125765439, 'index': 4126496961},
 {'time': 152.0172336101532, 'disk': 3487805249, 'index': 4586673462},
 {'time': 152.0630168914795, 'disk': 3870428919, 'index': 5165475988},
 {'time': 157.33430242538452, 'disk': 4145517491, 'index': 5381192671}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|162.50271010398865|1089123555|1547559666|
|20|309.09153389930725|1310979922|1822402821|
|30|451.55442452430725|1722240686|2374596699|
|40|597.6070141792297|2032917869|2908392605|
|50|744.8866584300995|2424082853|3432445955|
|60|893.3147761821747|2762769794|3817500475|
|70|1046.0243651866913|3125765439|4126496961|
|80|1198.0415987968445|3487805249|4586673462|
|90|1350.104615688324|3870428919|5165475988|
|100|1507.4389181137085|4145517491|5381192671|

Time to load without chunking: 1501.8327419757843

#### Threads: 5
```
[{'time': 82.61618256568909, 'disk': 1028416109, 'index': 1178578899},
 {'time': 93.2343819141388, 'disk': 1412548745, 'index': 1919106436},
 {'time': 100.10973262786865, 'disk': 1750961281, 'index': 2472092288},
 {'time': 103.07891869544983, 'disk': 2130080357, 'index': 3009628048},
 {'time': 105.06236481666565, 'disk': 2563519424, 'index': 3472262181},
 {'time': 109.6224672794342, 'disk': 2892115664, 'index': 3947697516},
 {'time': 115.8645966053009, 'disk': 3243101852, 'index': 4263697280},
 {'time': 114.3697292804718, 'disk': 3625831454, 'index': 4652460549},
 {'time': 110.75835919380188, 'disk': 4106390054, 'index': 5211345262},
 {'time': 115.44159722328186, 'disk': 4362265517, 'index': 5514485229}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|82.61618256568909|1028416109|1178578899|
|20|175.85056447982788|1412548745|1919106436|
|30|275.96029710769653|1750961281|2472092288|
|40|379.03921580314636|2130080357|3009628048|
|50|484.101580619812|2563519424|3472262181|
|60|593.7240478992462|2892115664|3947697516|
|70|709.5886445045471|3243101852|4263697280|
|80|823.9583737850189|3625831454|4652460549|
|90|934.7167329788208|4106390054|5211345262|
|100|1050.1583302021027|4362265517|5514485229|

Time to load without chunking: 1053.9380702972412

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

Time to load without chunking: 984.0401871204376

#### Threads: 20
```
[{'time': 83.00524592399597, 'disk': 1088634412, 'index': 2368193569},
 {'time': 102.83153247833252, 'disk': 1742261372, 'index': 2383289307},
 {'time': 118.12259697914124, 'disk': 2296869017, 'index': 2552836869},
 {'time': 98.42180371284485, 'disk': 2212252861, 'index': 3245254862},
 {'time': 100.04490566253662, 'disk': 2648500076, 'index': 3910376757},
 {'time': 101.33223032951355, 'disk': 3109872678, 'index': 4295152194},
 {'time': 105.38703346252441, 'disk': 3660888596, 'index': 4736229435},
 {'time': 103.67567420005798, 'disk': 3932108838, 'index': 5251204295},
 {'time': 100.8618516921997, 'disk': 4231695081, 'index': 5873872341},
 {'time': 104.34968733787537, 'disk': 4642203679, 'index': 6358290429}]
 ```

 |Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|83.00524592399597|1088634412|2368193569|
|20|185.8367784023285|1742261372|2383289307|
|30|303.9593753814697|2296869017|2552836869|
|40|402.3811790943146|2212252861|3245254862|
|50|502.4260847568512|2648500076|3910376757|
|60|603.7583150863647|3109872678|4295152194|
|70|709.1453485488892|3660888596|4736229435|
|80|812.8210227489471|3932108838|5251204295|
|90|913.6828744411469|4231695081|5873872341|
|100|1018.0325617790222|4642203679|6358290429|

Time to load without chunking: 983.1698892116547

#### Graphs

![Load time vs. threads for 100M edges](images/loadtime_vs_threads_100Mdocs.png)
![Load time vs. db size for 10M edges](images/time_to_load_10Medges_vs_db_size.png)
![Data size vs. threads for 100M edges](images/datasize_vs_threads_100Mdocs.png)
![Index size vs. threads for 100M edges](images/indexsize_vs_threads_100Mdocs.png)

### Importing ~1B edges

Make the full data file
```
In [1]: import gzip

In [2]: with gzip.open('data/NCBI_Prok-matrix.txt.gz', 'rt') as infile, gzip.open('data/NCBI_Prok-matrix.txt.gz.GCAonly.txt', 'wt') as outfile:
   ...:     for line in infile:
   ...:         name1, name2, score = line.split()
   ...:         if '_GCA_' in name1 and '_GCA_' in name2:
   ...:             id1 = name1.split('_GCA_')[1].split('.')[0]
   ...:             id2 = name2.split('_GCA_')[1].split('.')[0]
   ...:             outfile.write(f'GCA_{id1},GCA_{id2},{score},GCA_{id1}_GCA{id2}\n')
```

oops
```
root@f94963f88a5a:/arangobenchmark/data# mv NCBI_Prok-matrix.txt.gz.GCAonly.txt NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz
```

```
root@f94963f88a5a:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz | wc -l
934312527
root@f94963f88a5a:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz | wc -l
790191759
```
`head` and `tail` look correct

Using the same procedure as above for running imports:

```
In [6]: files = ['data/NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz']

In [7]: ret = run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz***

In [8]: ret
Out[8]: [{'time': 9511.930794477463, 'disk': 30064289593, 'index': 34790415400}]
```

```
root@49a6ab15f017:/arangobenchmark/data# tail -5 NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz.out 
created:          790182532
warnings/errors:  9227
updated/replaced: 0
ignored:          0
lines read:       790191760
```

Errors are all conflicts:
```
root@f94963f88a5a:/arangobenchmark/data# grep WARNING NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz.out | grep -v "unique constraint"
root@f94963f88a5a:/arangobenchmark/data# 
```

### Import 10M edges into a collection with 780M edges

```
root@a85c5bf36a43:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz | head -780000000 | gzip > NCBI_Prok-matrix.txt.gz.GCAonly.head780M.txt.gz
```

`head` and `tail` look fine

```
root@a85c5bf36a43:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz | tail +780000001 | head -10000000 > NCBI_Prok-matrix.txt.gz.GCAonly.head780-790M.txt
```

Split looks good
```
root@a85c5bf36a43:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head780M.txt.gz | tail -2
GCA_001980835,GCA_000297635,74.699,GCA_001980835_GCA000297635
GCA_001980835,GCA_001831025,74.6598,GCA_001980835_GCA001831025
root@a85c5bf36a43:/arangobenchmark/data# head -2 NCBI_Prok-matrix.txt.gz.GCAonly.head780-790M.txt 
GCA_001980835,GCA_001552215,74.6325,GCA_001980835_GCA001552215
GCA_001980835,GCA_001829475,74.6063,GCA_001980835_GCA001829475
root@a85c5bf36a43:/arangobenchmark/data# gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz | grep -n -C 3 GCA_001980835_GCA001552215
779999998-GCA_001980835,GCA_001788395,74.7086,GCA_001980835_GCA001788395
779999999-GCA_001980835,GCA_000297635,74.699,GCA_001980835_GCA000297635
780000000-GCA_001980835,GCA_001831025,74.6598,GCA_001980835_GCA001831025
780000001:GCA_001980835,GCA_001552215,74.6325,GCA_001980835_GCA001552215
780000002-GCA_001980835,GCA_001829475,74.6063,GCA_001980835_GCA001829475
780000003-GCA_001980845,GCA_001528085,85.223,GCA_001980845_GCA001528085
780000004-GCA_001980845,GCA_000756845,84.7426,GCA_001980845_GCA000756845
```

Using the same procedures as above to load the edges:
```
In [6]: files = ['data/NCBI_Prok-matrix.txt.gz.GCAonly.head780M.txt.gz', 'data/NCBI_Prok-matrix.txt.gz.GCAonly.head780-790M.txt']

In [7]: ret = run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head780M.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head780-790M.txt***

In [8]: ret
Out[8]: 
[{'time': 9322.453085422516, 'disk': 29807987933, 'index': 34455606868},
 {'time': 126.4388222694397, 'disk': 30138533830, 'index': 34939498403}]
```

```
root@a85c5bf36a43:/arangobenchmark# tail -4 data/NCBI_Prok-matrix.txt.gz.GCAonly.head780M.txt.gz.out 
warnings/errors:  9116
updated/replaced: 0
ignored:          0
lines read:       780000001
root@a85c5bf36a43:/arangobenchmark# tail -4 data/NCBI_Prok-matrix.txt.gz.GCAonly.head780-790M.txt.out 
warnings/errors:  110
updated/replaced: 0
ignored:          0
lines read:       10000001
```

### Interlude: rebuild docker image

Lots of stuff from the original image is going to be missing...

```
$ docker pull python:3.10.4-bullseye
Digest: sha256:86862fd2ad17902cc3a95b7effd257dfd043151f05d280170bdd6ff34f7bc78b
$ docker run -it python:3.10.4-bullseye bash
root@1698400eea2b:/# pip install ipython python-arango
root@1698400eea2b:/# exit
$ docker ps -a | grep bullseye
1698400eea2b        python:3.10.4-bullseye                                                                    "bash"                   3 minutes ago       Exited (127) 3 minutes ago                                   thirsty_robinson
$ docker commit 1698400eea2b gavins_ipython_image_dont_delete
```

Assume docker commits after adding stuff from there on out

Leaving out obvious dir creation, cds, etc

```
$ !502
export ARANGO_PWD_CI=$(head -c -1 ~/.arango3.5gavin)
$ docker run -it -e "ARANGO_PWD_CI=$ARANGO_PWD_CI" gavins_ipython_image_dont_delete bash
root@8fe86466b937:/arangobenchmark/arango/3.9.1# wget https://download.arangodb.com/arangodb39/Community/Linux/arangodb3-client-linux-3.9.1.tar.gz
root@8fe86466b937:/arangobenchmark/arango/3.9.1# tar -xf arangodb3-client-linux-3.9.1.tar.gz 
root@8fe86466b937:/arangobenchmark/arango/3.9.1# mv arangodb3-client-linux-3.9.1/* .
root@8fe86466b937:/arangobenchmark/arango/3.9.1# rmdir arangodb3-client-linux-3.9.1
root@8fe86466b937:/arangobenchmark/arango/3.9.1# rm arangodb3-client-linux-3.9.1.tar.gz 
root@8fe86466b937:/arangobenchmark# apt update
root@8fe86466b937:/arangobenchmark# apt install nano
```

In another terminal:
```
~/relationengine/testdata$ docker cp NCBI_Prok-matrix.txt.gz 8fe86466b937:/
```

Back to the original terminal:
```
root@8fe86466b937:/arangobenchmark/data# mv /NCBI_Prok-matrix.txt.gz .
```

### Imports with full strain names

Prep file

```
root@cf6859c67159:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import gzip

In [2]: with gzip.open('data/NCBI_Prok-matrix.txt.gz', 'rt') as infile, gzip.open('data/NCBI_Prok-matrix.txt.key.gz', 'wt') as outfile:
   ...:     for line in infile:
   ...:         id1, id2, score = line.split()
   ...:         outfile.write(f'{id1},{id2},{score},{id1}_{id2}\n')
   ...: 
```

Headers file:
```
root@cf6859c67159:/arangobenchmark/data# cat NCBI_Prok-matrix.txt.key.headers.txt 
_from,_to,idscore,_key
```

Shell script:
```
root@cf6859c67159:/arangobenchmark# cat import.parameterized.sh 
#!/usr/bin/env sh

arango/3.9.1/bin/arangoimport \
    --file $INFILE \
    --headers-file data/NCBI_Prok-matrix.txt.key.headers.txt \
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

ipython code:
```
root@903bb9f59479:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import os
In [2]: import subprocess
In [3]: import arango
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
   ...:             './import.parameterized.sh',
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

In [6]: files = !ls data/NCBI_Prok-matrix.txt.key.gz

In [8]: ret = run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.key.gz***

In [9]: ret
Out[9]: [{'time': 16552.933314561844, 'disk': 66566784024, 'index': 68504968850}]

root@903bb9f59479:/arangobenchmark# tail -5 data/NCBI_Prok-matrix.txt.key.gz.out 
created:          934312527
warnings/errors:  0
updated/replaced: 0
ignored:          0
lines read:       934312528
```

### Interlude 2: recreate docker image again to reduce size

... to enable persistence via `docker save` and `docker load`.

* Install all needed tools & save
* Mount large data files vs keeping them in the container

```
$ docker run -it -v ~/relationengine/testdata:/arangobenchmark/data python:3.10.4-bullseye bash
root@5a2934920add:/arangobenchmark# pip install ipython python-arango
root@5a2934920add:/arangobenchmark# apt update
root@5a2934920add:/arangobenchmark# apt install nano
root@5a2934920add:/arangobenchmark# rm -rf /var/lib/apt/lists/*
root@5a2934920add:/arangobenchmark/arango# wget https://download.arangodb.com/arangodb39/Community/Linux/arangodb3-client-linux-3.9.1.tar.gz
root@5a2934920add:/arangobenchmark/arango# tar -xf arangodb3-client-linux-3.9.1.tar.gz 
root@5a2934920add:/arangobenchmark/arango# mv arangodb3-client-linux-3.9.1 3.9.1
root@5a2934920add:/arangobenchmark/arango# rm arangodb3-client-linux-3.9.1.tar.gz
$ docker commit 5a2934920add gavins_ipython_image_dont_delete
sha256:d308fe326266a6c7756f76e7444bd25faf73a4aa3b38a3220623f78f95ccac63
```

Also commit after adding scripts, ipython functions, etc, not shown. *Don't* commit after adding
large data.

To run:
```
$ !490
export ARANGO_PWD_CI=$(head -c -1 ~/.arango3.5gavin)
$ docker run -it -e "ARANGO_PWD_CI=$ARANGO_PWD_CI" -v ~/relationengine/testdata:/arangobenchmark/data gavins_ipython_image_dont_delete bash
```

Headers file (note mounted now)
```
root@c1a775a9517a:/arangobenchmark/data# cat NCBI_Prok-matrix.txt.key.headers.txt 
_from,_to,idscore,_key
```

Shell script (same as prior listing)
```
root@c1a775a9517a:/arangobenchmark# cat import.parameterized.sh 
#!/usr/bin/env sh

arango/3.9.1/bin/arangoimport \
    --file $INFILE \
    --headers-file data/NCBI_Prok-matrix.txt.key.headers.txt \
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

root@c1a775a9517a:/arangobenchmark# chmod a+x import.parameterized.sh 
```

ipython commands:
```
root@c1a775a9517a:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import os

In [2]: import subprocess

In [3]: import arango

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
   ...:             './import.parameterized.sh',
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

In [6]: files = !ls data/NCBI_Prok-matrix.txt.gz.GCAonly.head*-*M.key.txt.gz

In [8]: ret = run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head10-20M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head20-30M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-40M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head40-50M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head50-60M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-70M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head70-80M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head80-90M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head90-100M.key.txt.gz***

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

In [10]: ret
Out[10]: 
[{'time': 86.81573224067688, 'disk': 949347892, 'index': 1167781464},
 {'time': 101.16269755363464, 'disk': 1352009429, 'index': 1792281278},
 {'time': 106.16566228866577, 'disk': 1754879599, 'index': 2360931803},
 {'time': 109.93988513946533, 'disk': 2102285551, 'index': 2735284083},
 {'time': 113.78973937034607, 'disk': 2387373687, 'index': 3199688078},
 {'time': 114.59047746658325, 'disk': 2893977967, 'index': 3682897303},
 {'time': 113.77107119560242, 'disk': 3273406214, 'index': 4041828143},
 {'time': 123.94383144378662, 'disk': 3596679860, 'index': 4450177119},
 {'time': 116.43767976760864, 'disk': 3952700237, 'index': 4931605652},
 {'time': 122.76644206047058, 'disk': 4408017533, 'index': 5446271927}]

In [11]: print_res(ret)
|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|86.81573224067688|949347892|1167781464|
|20|187.97842979431152|1352009429|1792281278|
|30|294.1440920829773|1754879599|2360931803|
|40|404.0839772224426|2102285551|2735284083|
|50|517.8737165927887|2387373687|3199688078|
|60|632.464194059372|2893977967|3682897303|
|70|746.2352652549744|3273406214|4041828143|
|80|870.179096698761|3596679860|4450177119|
|90|986.6167764663696|3952700237|4931605652|
|100|1109.3832185268402|4408017533|5446271927|
```

Save the image
```
$ docker commit c1a775a9517a        gavins_ipython_image_dont_delete
sha256:18cf889418dcffd5264c2b6967105ea75299fc6f583ad67157ceb8c8ce84aa21
~/relationengine$ docker save gavins_ipython_image_dont_delete | gzip > docker_images/gavins_python_image_dont_delete.tar.gz
```

### Test with 2x replication, write concern = 1 or 2

#### Replication 1, write concern = 1
(from run above)

```
In [10]: ret
Out[10]: 
[{'time': 86.81573224067688, 'disk': 949347892, 'index': 1167781464},
 {'time': 101.16269755363464, 'disk': 1352009429, 'index': 1792281278},
 {'time': 106.16566228866577, 'disk': 1754879599, 'index': 2360931803},
 {'time': 109.93988513946533, 'disk': 2102285551, 'index': 2735284083},
 {'time': 113.78973937034607, 'disk': 2387373687, 'index': 3199688078},
 {'time': 114.59047746658325, 'disk': 2893977967, 'index': 3682897303},
 {'time': 113.77107119560242, 'disk': 3273406214, 'index': 4041828143},
 {'time': 123.94383144378662, 'disk': 3596679860, 'index': 4450177119},
 {'time': 116.43767976760864, 'disk': 3952700237, 'index': 4931605652},
 {'time': 122.76644206047058, 'disk': 4408017533, 'index': 5446271927}]
 ```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|86.81573224067688|949347892|1167781464|
|20|187.97842979431152|1352009429|1792281278|
|30|294.1440920829773|1754879599|2360931803|
|40|404.0839772224426|2102285551|2735284083|
|50|517.8737165927887|2387373687|3199688078|
|60|632.464194059372|2893977967|3682897303|
|70|746.2352652549744|3273406214|4041828143|
|80|870.179096698761|3596679860|4450177119|
|90|986.6167764663696|3952700237|4931605652|
|100|1109.3832185268402|4408017533|5446271927|

#### Replication 2, write concern = 1

```
In [9]: ret
Out[9]: 
[{'time': 187.63442873954773, 'disk': 827101724, 'index': 981185771},
 {'time': 215.33341908454895, 'disk': 1085746407, 'index': 1475026269},
 {'time': 218.79501628875732, 'disk': 1503111098, 'index': 2018335206},
 {'time': 222.90910935401917, 'disk': 1870773853, 'index': 2379575805},
 {'time': 228.29527640342712, 'disk': 2221447246, 'index': 2757545106},
 {'time': 233.11459732055664, 'disk': 2685796111, 'index': 3232485568},
 {'time': 233.15758442878723, 'disk': 3031608167, 'index': 3777998414},
 {'time': 235.30174851417542, 'disk': 3515258033, 'index': 4203683176},
 {'time': 240.15702867507935, 'disk': 3723252997, 'index': 4698533660},
 {'time': 231.10766696929932, 'disk': 4072413844, 'index': 5129758976}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|187.63442873954773|827101724|981185771|
|20|402.9678478240967|1085746407|1475026269|
|30|621.762864112854|1503111098|2018335206|
|40|844.6719734668732|1870773853|2379575805|
|50|1072.9672498703003|2221447246|2757545106|
|60|1306.081847190857|2685796111|3232485568|
|70|1539.2394316196442|3031608167|3777998414|
|80|1774.5411801338196|3515258033|4203683176|
|90|2014.698208808899|3723252997|4698533660|
|100|2245.8058757781982|4072413844|5129758976|

#### Replication 2, write concern = 2

```
In [14]: ret
Out[14]: 
[{'time': 202.84811568260193, 'disk': 791070725, 'index': 1036492066},
 {'time': 219.01715087890625, 'disk': 1050555342, 'index': 1312254228},
 {'time': 221.07375836372375, 'disk': 1511362413, 'index': 1864033328},
 {'time': 226.06171917915344, 'disk': 1847668738, 'index': 2317275687},
 {'time': 232.40011930465698, 'disk': 2159771737, 'index': 2693763483},
 {'time': 232.5830192565918, 'disk': 2517824849, 'index': 3224745190},
 {'time': 225.28864288330078, 'disk': 2898331481, 'index': 3717340008},
 {'time': 231.02699995040894, 'disk': 3203535008, 'index': 4083733890},
 {'time': 233.80236911773682, 'disk': 3686826765, 'index': 4567129658},
 {'time': 232.3505344390869, 'disk': 3909801576, 'index': 4994828412}]
```

|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|202.84811568260193|791070725|1036492066|
|20|421.8652665615082|1050555342|1312254228|
|30|642.9390249252319|1511362413|1864033328|
|40|869.0007441043854|1847668738|2317275687|
|50|1101.4008634090424|2159771737|2693763483|
|60|1333.9838826656342|2517824849|3224745190|
|70|1559.272525548935|2898331481|3717340008|
|80|1790.2995254993439|3203535008|4083733890|
|90|2024.1018946170807|3686826765|4567129658|
|100|2256.4524290561676|3909801576|4994828412|
