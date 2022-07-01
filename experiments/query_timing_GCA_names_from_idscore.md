# Test query timing for ID score cutoff on specific genomes with different collection sizes

[Test data setup for 10M and 100M GCA names](../create_test_data.md#100m-gca-id-edges-split-into-10m-chunks)  
[Test data setup for 800M GCA names](../create_test_data.md#780m-cga-edges--10m-gca-edges)  
[Test data setup for 1M GCA names](../create_test_data.md#1m-gca-edges)

[Environment setup](../environment_setup.md#set-up-for-query-timings)

## Test that the data is unsorted

The plan is to randomly choose IDs from the first 1M lines of the data and then
run queries against 1M, 10M, 100M, and 800M item Arango collections. If the data is ordered,
then the first 1M lines of the data will be first in the index and so that might have an effect
on the search time. If the data is unordered then it shouldn't.

```
root@28c427a095af:/arangobenchmark/data# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import gzip

In [3]: from_ = []

In [5]: with gzip.open('NCBI_Prok-matrix.txt.gz.GCAonly.txt.gz', 'rt') as infile
   ...: :
   ...:     for l in infile:
   ...:         f, t, s, k = l.split(',')
   ...:         from_.append(f)
   ...: 

In [6]: import random

In [7]: 
   ...: def stochastic_sortedness(list_, num_samples):
   ...:     ''' Based on https://stackoverflow.com/a/16994740/643675 '''
   ...:     if len(list_) < 2:
   ...:         return 0
   ...:     # a 2 item list works, but is a pretty dumb thing to pass to this fu
   ...: nction *shrug*
   ...:     maxindex = len(list_) - 1
   ...:     score = 0
   ...:     count = 0
   ...:     while count < num_samples:
   ...:         i1 = random.randint(0, maxindex)
   ...:         i2 = random.randint(0, maxindex)
   ...:         if i1 != i2:
   ...:             score += 0 if list_[min(i1, i2)] <= list_[max(i1, i2)] else
   ...: 1
   ...:             count += 1
   ...:     return score / num_samples
   ...: 

In [8]: len(from_)
Out[8]: 790191759

In [10]: stochastic_sortedness(from_, 100000)
Out[10]: 0.49895

In [11]: stochastic_sortedness(from_[0:100000000], 100000)
Out[11]: 0.50089

In [12]: stochastic_sortedness(from_[0:10000000], 100000)
Out[12]: 0.49483

In [13]: stochastic_sortedness(from_[0:1000000], 100000)
Out[13]: 0.4892
```

## Find genomes to test

Looking for genomes with outdegree of various orders of magnitude   
```
root@fec1b4bce30a:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import json

In [2]: import gzip

In [3]: from collections import defaultdict

In [4]: with open('data/frequencies.json') as infile:
   ...:     freqs_800M = json.loads(infile.read())
   ...: 

In [5]: len(freqs_800M)
Out[5]: 81013

In [6]: def from_freqs(filename):
   ...:     with gzip.open(filename, 'rt') as infile:
   ...:         freqs = defaultdict(int)
   ...:         for line in infile:
   ...:             name1, name2, score, key = line.split(',')
   ...:             freqs[name1.strip()] += 1
   ...:     return freqs
   ...: 

In [7]: freqs_100M = from_freqs('data/NCBI_Prok-matrix.txt.gz.GCAonly.head100M.key.txt.gz')

In [8]: len(freqs_100M)
Out[8]: 78056

In [9]: sum(freqs_100M.values())
Out[9]: 100000000

In [10]: freqs_10M = from_freqs('data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-10M.key.txt.gz')

In [11]: len(freqs_10M)
Out[11]: 73757

In [12]: sum(freqs_10M.values())
Out[12]: 10000000

In [13]: freqs_1M = from_freqs('data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-1M.key.txt.gz')

In [14]: len(freqs_1M)
Out[14]: 36564

In [15]: sum(freqs_1M.values())
Out[15]: 1000000

In [16]: freqs_all = {k: (
    ...:     freqs_1M[k],
    ...:     freqs_10M[k],
    ...:     freqs_100M[k],
    ...:     freqs_800M[k]) for k in freqs_1M}

In [17]: len(freqs_all)
Out[17]: 36564

In [18]: sorted(freqs_all.items(), key=lambda x: x[1][3])[0]
Out[18]: ('GCA_000714715', (1, 1, 1, 1))

In [19]: sorted(freqs_all.items(), key=lambda x: x[1][3])[-1]
Out[19]: ('GCA_000817745', (189, 1157, 9672, 75746))

In [20]: freqs_all_sorted = sorted(freqs_all.items(), key=lambda x: x[1][3])

In [21]: freqs_all_sorted[1000]
Out[21]: ('GCA_001869205', (1, 1, 18, 186))

In [22]: freqs_all_sorted[10000]
Out[22]: ('GCA_001493775', (7, 35, 214, 1695))

In [23]: freqs_all_sorted[20000]
Out[23]: ('GCA_001382175', (32, 172, 1729, 13598))

In [24]: freqs_all_sorted[15000]
Out[24]: ('GCA_900105565', (18, 87, 847, 6846))

In [25]: freqs_all_sorted[17000]
Out[25]: ('GCA_001322565', (32, 168, 1666, 13090))

In [26]: freqs_all_sorted[16000]
Out[26]: ('GCA_001899065', (27, 154, 1331, 10735))

In [27]: freqs_all_sorted[5000]
Out[27]: ('GCA_000525795', (4, 24, 155, 1224))

In [28]: freqs_all_sorted[4000]
Out[28]: ('GCA_001562915', (1, 13, 144, 1151))

In [29]: freqs_all_sorted[3000]
Out[29]: ('GCA_000274685', (1, 8, 71, 570))

In [30]: freqs_all_sorted[3500]
Out[30]: ('GCA_001517115', (1, 12, 107, 908))

In [31]: freqs_all_sorted[3700]
Out[31]: ('GCA_001043795', (3, 15, 128, 1054))

In [34]: targets = [freqs_all_sorted[0][0], freqs_all_sorted[3700][0], freqs_all_sorted[16000][0], freqs_all_sorted[-1][0]]

In [35]: targets
Out[35]: ['GCA_000714715', 'GCA_001043795', 'GCA_001899065', 'GCA_000817745']

In [37]: list(map(lambda x: freqs_all[x], targets))
Out[37]: 
[(1, 1, 1, 1),
 (3, 15, 128, 1054),
 (27, 154, 1331, 10735),
 (189, 1157, 9672, 75746)]
```

Outdegrees for each genome in the various collections:
|Genome ID|1M|10M|100M|800M|
|--|--|--|--|--|
|GCA_000714715|1|1|1|1|
|GCA_001043795|3|15|128|1054|
|GCA_001899065|27|154|1331|10735|
|GCA_000817745|189|1157|9672|75746|

## Calculating stats and distributions on genomes to test

```
root@3cae5fd1a4e7:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import arango

In [2]: import os

In [3]: import statistics

In [4]: pwd = os.environ['ARANGO_PWD_CI']

In [5]: acli = arango.ArangoClient(hosts='http://10.58.1.211:8531')

In [6]: db = acli.db('gavin_test', username='gavin', password=pwd)

In [7]: aql = db.aql

In [8]: def query_genome_idscore(colname, genome_id, minpercent):
   ...:     return list(aql.execute(
   ...:         f'''
   ...:         FOR doc IN {colname}
   ...:             FILTER doc._from == "node/{genome_id}"
   ...:             FILTER doc.idscore > {minpercent}
   ...:             RETURN doc
   ...:         '''))
   ...: 

In [9]: targets = ['GCA_000714715', 'GCA_001043795', 'GCA_001899065', 'GCA_000817745']

In [10]: res = [query_genome_idscore('FastANI_800M_idscore_index', g, 0) for g in targets]

In [11]: idscores_each = [[d['idscore'] for d in docs] for docs in res]

In [12]: [len(idscores_each[x]) for x in range(len(idscores_each))]
Out[12]: [1, 1054, 10735, 75742]

In [13]: [statistics.mean(idscores_each[x]) for x in range(len(idscores_each))]
Out[13]: [100, 78.1700944971537, 75.34356068933396, 75.09656010535767]

In [14]: [statistics.median(idscores_each[x]) for x in range(len(idscores_each))]
Out[14]: [100, 77.05845, 75.1737, 75.0807]

In [15]: [min(idscores_each[x]) for x in range(len(idscores_each))]
Out[15]: [100, 75.688, 74.5356, 74.4053]

In [16]: [max(idscores_each[x]) for x in range(len(idscores_each))]
Out[16]: [100, 100, 100, 99.9443]

In [17]: [statistics.pstdev(idscores_each[x]) for x in range(1, 4)]
Out[17]: [4.0429053351381015, 0.6267747861299644, 0.22337950665706552]

In [18]: import numpy as np
      
In [19]: res = np.histogram(idscores_each[1], bins=30, range=(70,100))

In [20]: for i in res[0]:
    ...:     print(i)
    ...: 
0
0
0
0
0
4
460
350
176
0
0
0
0
0
0
1
0
0
1
32
1
1
0
0
0
0
0
4
20
4

In [21]: res = np.histogram(idscores_each[2], bins=30, range=(70,100))

In [22]: for i in res[0]:
    ...:     print(i)
    ...: 
0
0
0
0
2698
6464
1414
137
15
5
0
0
1
0
0
0
0
0
0
0
0
0
0
0
0
0
0
0
0
1

In [23]: res = np.histogram(idscores_each[3], bins=30, range=(70,100))

In [24]: for i in res[0]:
    ...:     print(i)
    ...: 
0
0
0
0
21745
53852
133
9
0
0
0
0
0
0
0
0
0
0
1
0
0
0
0
0
0
0
0
0
1
1

```

## Figure out how %timeit calculates its results

```
root@fec1b4bce30a:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import arango

In [2]: import os

In [3]: import statistics

In [4]: pwd = os.environ['ARANGO_PWD_CI']

In [5]: acli = arango.ArangoClient(hosts='http://10.58.1.211:8531')

In [6]: db = acli.db('gavin_test', username='gavin', password=pwd)

In [7]: aql = db.aql

In [8]: targets = ['GCA_000714715', 'GCA_001043795', 'GCA_001899065', 'GCA_000817745']

In [9]: colnames = ['FastANI_800M', 'FastANI_800M_idscore_index', 'FastANI_100M', 'FastANI_100M_idscore_index', 'FastANI_10M', 'FastANI_10M_idscore_index', 'FastANI_1M', 'FastANI_1M_idscore_index']

In [10]: def query_genome_idscore(colname, genome_id, minpercent):
    ...:     return list(aql.execute(
    ...:         f'''
    ...:         FOR doc IN {colname}
    ...:             FILTER doc._from == "node/{genome_id}"
    ...:             FILTER doc.idscore > {minpercent}
    ...:             RETURN doc
    ...:         '''))
    ...: 

In [11]: for c in colnames:
    ...:     res = ['foo']
    ...:     print(c)
    ...:     tires = %timeit -o res[0] = query_genome_idscore(c, targets[0], 0)
    ...:     print(len(tires.all_runs), statistics.mean(tires.all_runs) / tires.loops, statistics.pstdev(tires.all_runs) / tires.loops)
    ...:     print('result count:', len(res), '\n')
    ...: 
FastANI_800M
7.44 ms ± 296 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.007442028578370809 0.00029627987014653175
result count: 1 

FastANI_800M_idscore_index
7.29 ms ± 180 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.007291783359167831 0.00017982523595152594
result count: 1 

FastANI_100M
7.4 ms ± 123 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.0074014086941523215 0.0001225137992045393
result count: 1 

FastANI_100M_idscore_index
7.34 ms ± 481 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.007335876861054983 0.00048128667622457854
result count: 1 

FastANI_10M
7.78 ms ± 219 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.007784079354522484 0.00021933045948754646
result count: 1 

FastANI_10M_idscore_index
7.54 ms ± 254 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.007535800522725496 0.0002535797981421664
result count: 1 

FastANI_1M
6.82 ms ± 307 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.006820344987458416 0.0003072175292368485
result count: 1 

FastANI_1M_idscore_index
7.55 ms ± 201 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
7 0.0075514899965907845 0.00020117101549935773
result count: 1 
```

## Calculate query times for targets at various %ID cutoffs and collection sizes

No ID cutoff for the first target needed, only 1 result
```
root@3cae5fd1a4e7:/arangobenchmark# ipython
Python 3.10.4 (main, May 28 2022, 13:14:58) [GCC 10.2.1 20210110]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.4.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import arango

In [2]: import os

In [3]: import statistics

In [4]: from collections import defaultdict

In [5]: pwd = os.environ['ARANGO_PWD_CI']

In [6]: acli = arango.ArangoClient(hosts='http://10.58.1.211:8531')

In [7]: db = acli.db('gavin_test', username='gavin', password=pwd)

In [8]: aql = db.aql

In [9]: def query_genome_idscore(colname, genome_id, minpercent):
   ...:     return list(aql.execute(
   ...:         f'''
   ...:         FOR doc IN {colname}
   ...:             FILTER doc._from == "node/{genome_id}"
   ...:             FILTER doc.idscore > {minpercent}
   ...:             RETURN doc
   ...:         '''))
   ...: 

In [10]: def run_queries(target, idscores):
    ...:     res = defaultdict(dict)
    ...:     for c in colnames:
    ...:         for idscore in idscores:
    ...:             print(c, idscore)
    ...:             holder = ['foo']
    ...:             tires = %timeit -o holder[0] = query_genome_idscore(c, target, idscore)
    ...:             res[c][idscore] = (statistics.mean(tires.all_runs) / tires.loops, statistics.pstdev(tires.all_runs) / tires.loops, len(holder[0]))
    ...:     return res
    ...: 

In [11]: targets = ['GCA_000714715', 'GCA_001043795', 'GCA_001899065', 'GCA_000817745']

In [12]: colnames = ['FastANI_800M', 'FastANI_800M_idscore_index', 'FastANI_100M', 'FastANI_100M_idscore_index', 'FastANI_10M', 'FastANI_10M_idscore_index', 'F
    ...: astANI_1M', 'FastANI_1M_idscore_index']

In [13]: res = run_queries(targets[0], [0])
FastANI_800M 0
7.65 ms ± 177 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 0
7.88 ms ± 226 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 0
7.59 ms ± 299 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 0
7.96 ms ± 287 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 0
7.78 ms ± 204 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 0
7.77 ms ± 231 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 0
7.55 ms ± 206 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 0
7.9 ms ± 270 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)

In [14]: res
Out[14]: 
defaultdict(dict,
            {'FastANI_800M': {0: (0.0076479921862483025,
               0.00017719417889011814,
               1)},
             'FastANI_800M_idscore_index': {0: (0.007876891875639558,
               0.000225777670430686,
               1)},
             'FastANI_100M': {0: (0.007585750492289663,
               0.00029884375407547006,
               1)},
             'FastANI_100M_idscore_index': {0: (0.00795514882542193,
               0.0002869983254324934,
               1)},
             'FastANI_10M': {0: (0.007784612947808844,
               0.0002041133132579265,
               1)},
             'FastANI_10M_idscore_index': {0: (0.007765462869512183,
               0.00023078830513654434,
               1)},
             'FastANI_1M': {0: (0.007548005862959794,
               0.0002057486010154933,
               1)},
             'FastANI_1M_idscore_index': {0: (0.00790126448231084,
               0.00026980648326645673,
               1)}})

In [15]: def reformat_results(res):
    ...:     ret = {}
    ...:     for colname in res:
    ...:         idx = 'Index' if 'idscore_index' in colname else 'No index'
    ...:         dbsize = int(colname.split('M')[0][8:])
    ...:         if dbsize not in ret:
    ...:             ret[dbsize] = {}
    ...:         ret[dbsize][idx] = res[colname]
    ...:     return ret
    ...: 

In [16]: res2 = reformat_results(res)

In [17]: res2
Out[17]: 
{800: {'No index': {0: (0.0076479921862483025, 0.00017719417889011814, 1)},
  'Index': {0: (0.007876891875639558, 0.000225777670430686, 1)}},
 100: {'No index': {0: (0.007585750492289663, 0.00029884375407547006, 1)},
  'Index': {0: (0.00795514882542193, 0.0002869983254324934, 1)}},
 10: {'No index': {0: (0.007784612947808844, 0.0002041133132579265, 1)},
  'Index': {0: (0.007765462869512183, 0.00023078830513654434, 1)}},
 1: {'No index': {0: (0.007548005862959794, 0.0002057486010154933, 1)},
  'Index': {0: (0.00790126448231084, 0.00026980648326645673, 1)}}}

In [26]: def print_0_percid_data(target, data):
    ...:     sizes = sorted(data.keys())
    ...:     print(target)
    ...:     print('|DB size|Compound Index|Default Index|Count|')
    ...:     print('|--|--|--|--|')
    ...:     for size in sizes:
    ...:         print(f'|{size}|{data[size]["Index"][0][0]}|{data[size]["No index"][0][0]}|{data[size]["Index"][0][2]}|')
    ...: 

In [27]: print_0_percid_data(targets[0], res2)
GCA_000714715
|DB size|Compound Index|Default Index|Count|
|--|--|--|--|
|1|0.00790126448231084|0.007548005862959794|1|
|10|0.007765462869512183|0.007784612947808844|1|
|100|0.00795514882542193|0.007585750492289663|1|
|800|0.007876891875639558|0.0076479921862483025|1|
```

GCA_000714715, %ID cutoff = 0
|DB size|Compound Index|Default Index|Count|
|--|--|--|--|
|1|0.00790126448231084|0.007548005862959794|1|
|10|0.007765462869512183|0.007784612947808844|1|
|100|0.00795514882542193|0.007585750492289663|1|
|800|0.007876891875639558|0.0076479921862483025|1|

```
In [28]: res = run_queries(targets[1], [70, 75, 76, 76.5, 77, 77.5, 78, 79, 100])
FastANI_800M 70
28.5 ms ± 3.56 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 75
27 ms ± 686 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 76
24.9 ms ± 1.35 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 76.5
22.4 ms ± 619 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 77
17.9 ms ± 539 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M 77.5
15.3 ms ± 384 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M 78
14.3 ms ± 313 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M 79
12.5 ms ± 327 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M 100
12 ms ± 145 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 70
29.7 ms ± 1.1 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 75
29.1 ms ± 1.18 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 76
28.4 ms ± 1.07 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 76.5
21.2 ms ± 972 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 77
17.4 ms ± 960 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 77.5
13.1 ms ± 544 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 78
12.1 ms ± 268 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 79
9.33 ms ± 234 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 100
7.91 ms ± 217 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 70
10 ms ± 176 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 75
10.2 ms ± 336 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 76
9.96 ms ± 254 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 76.5
9.97 ms ± 180 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 77
7.8 ms ± 454 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 77.5
8.56 ms ± 218 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 78
8.52 ms ± 161 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 79
8.49 ms ± 167 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 100
7.74 ms ± 186 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 70
9.8 ms ± 384 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 75
9.93 ms ± 242 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 76
10 ms ± 327 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 76.5
9.35 ms ± 486 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 77
8.77 ms ± 391 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 77.5
8.18 ms ± 336 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 78
7.85 ms ± 447 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 79
7.66 ms ± 157 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 100
7.53 ms ± 398 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 70
8.04 ms ± 317 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 75
7.22 ms ± 140 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 76
7.46 ms ± 419 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 76.5
8.02 ms ± 340 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 77
8.02 ms ± 247 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 77.5
7.96 ms ± 194 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 78
8.12 ms ± 350 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 79
7.81 ms ± 449 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 100
7.73 ms ± 269 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 70
8.31 ms ± 235 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 75
8.4 ms ± 369 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 76
8.42 ms ± 176 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 76.5
8.43 ms ± 410 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 77
8.13 ms ± 248 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 77.5
8.02 ms ± 242 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 78
7.8 ms ± 202 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 79
7.8 ms ± 241 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 100
7.61 ms ± 224 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 70
7.35 ms ± 257 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 75
7.45 ms ± 204 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 76
7.65 ms ± 206 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 76.5
7.4 ms ± 303 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 77
7.23 ms ± 124 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 77.5
7.47 ms ± 156 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 78
7.55 ms ± 112 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 79
7.3 ms ± 183 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 100
7.54 ms ± 297 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 70
7.64 ms ± 196 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 75
7.57 ms ± 163 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 76
7.56 ms ± 144 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 76.5
7.59 ms ± 259 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 77
7.26 ms ± 245 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 77.5
7.74 ms ± 302 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 78
7.42 ms ± 199 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 79
7.55 ms ± 239 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 100
7.48 ms ± 117 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)

In [29]: res2 = reformat_results(res)

In [30]: res
Out[30]: 
defaultdict(dict,
            {'FastANI_800M': {70: (0.028493346513382027,
               0.003557600128486706,
               1054),
              75: (0.026977396982588935, 0.0006864039211334353, 1054),
              76: (0.02487557285598346, 0.0013462340968643616, 1050),
              76.5: (0.022364463124956404, 0.0006185980448433306, 986),
              77: (0.01787294867714601, 0.0005388582978902705, 590),
              77.5: (0.015337911140439767, 0.0003835373308526435, 329),
              78: (0.014300070971782718, 0.00031291711810474, 240),
              79: (0.012529546873910087, 0.0003269927546216843, 64),
              100: (0.01203559348226658, 0.0001448220888627054, 0)},
             'FastANI_800M_idscore_index': {70: (0.02970689120037215,
               0.001096742590370909,
               1054),
              75: (0.02914605591712253, 0.001176741921586655, 1054),
              76: (0.02840630336265479, 0.001065119594937484, 1050),
              76.5: (0.021173634606280498, 0.0009716046601165061, 986),
              77: (0.01735409886177097, 0.0009599443502263486, 590),
              77.5: (0.013087128465995192, 0.0005436785108917917, 329),
              78: (0.012076096493484718, 0.0002683994997422064, 240),
              79: (0.009333554677931327, 0.0002341985010720023, 64),
              100: (0.007913854884515915, 0.00021666838770041468, 0)},
             'FastANI_100M': {70: (0.010013244921075446,
               0.0001764110239399362,
               128),
              75: (0.010155446200764605, 0.0003360788990035585, 128),
              76: (0.009958488683083227, 0.0002536822135181791, 128),
              76.5: (0.009968961138012154, 0.0001797973823596132, 119),
              77: (0.007802146435050028, 0.00045404572144724, 67),
              77.5: (0.008562839648553304, 0.00021799177136637947, 33),
              78: (0.008522078070257393, 0.0001607263392836673, 23),
              79: (0.008493307820920434, 0.00016658054323853054, 6),
              100: (0.00773597971403173, 0.00018611503950054697, 0)},
             'FastANI_100M_idscore_index': {70: (0.009804060428536364,
               0.00038446509828390875,
               128),
              75: (0.009932666611192482, 0.00024151859619545593, 128),
              76: (0.010006559283605642, 0.00032723211704371847, 128),
              76.5: (0.009351666628250054, 0.00048615873021256337, 119),
              77: (0.00876571058029575, 0.0003911189096031593, 67),
              77.5: (0.008184758604371121, 0.0003360898368287055, 33),
              78: (0.007853612736133593, 0.0004471635691679575, 23),
              79: (0.007664346237267766, 0.00015709078440174666, 6),
              100: (0.007527071768417954, 0.00039844052469151285, 0)},
             'FastANI_10M': {70: (0.008041770490152495,
               0.00031698074773842317,
               15),
              75: (0.0072174773125776226, 0.00013972553089447342, 15),
              76: (0.00745961337616401, 0.0004194008486447274, 15),
              76.5: (0.008022567416940418, 0.00033967950817123096, 14),
              77: (0.00801963801628777, 0.00024676797752452547, 9),
              77.5: (0.007957289974604334, 0.00019374517367913975, 4),
              78: (0.008121860810954656, 0.00034988860680218204, 1),
              79: (0.007809665611545954, 0.00044912731632021634, 0),
              100: (0.007728229372629097, 0.000268642950174923, 0)},
             'FastANI_10M_idscore_index': {70: (0.00830915835686028,
               0.00023492409318655758,
               15),
              75: (0.008397924781643919, 0.0003686478491755673, 15),
              76: (0.008417240057938865, 0.00017610013524301627, 15),
              76.5: (0.008427468491718174, 0.00040971289037902524, 14),
              77: (0.008133986459246704, 0.00024825076484186995, 9),
              77.5: (0.008017260426921503, 0.00024235725827028982, 4),
              78: (0.007804803276168448, 0.0002022518603534982, 1),
              79: (0.0077954007924667425, 0.00024081366730153476, 0),
              100: (0.0076052043799843105, 0.00022426132570790809, 0)},
             'FastANI_1M': {70: (0.007354256454855204,
               0.0002565781724687829,
               3),
              75: (0.007449815931863018, 0.0002035363791686122, 3),
              76: (0.007650200128555298, 0.00020565761553168399, 3),
              76.5: (0.007397548030795795, 0.00030326009351314186, 3),
              77: (0.007233466113518391, 0.00012416804973618421, 0),
              77.5: (0.0074686315815363615, 0.00015598785532553662, 0),
              78: (0.0075541199211563385, 0.00011214290614529302, 0),
              79: (0.007304560683135476, 0.0001827480692183891, 0),
              100: (0.007537868476605841, 0.00029727576206591396, 0)},
             'FastANI_1M_idscore_index': {70: (0.007642260768583843,
               0.000195874743873633,
               3),
              75: (0.007568490607663989, 0.00016326846325433455, 3),
              76: (0.007555569442255157, 0.00014361202797880674, 3),
              76.5: (0.007588767239025661, 0.000259293488587246, 3),
              77: (0.007260604251974395, 0.00024506818565012936, 0),
              77.5: (0.0077381157036870716, 0.00030247916611908824, 0),
              78: (0.00741843394802085, 0.0001985791540793954, 0),
              79: (0.007549205729737878, 0.0002387371231758983, 0),
              100: (0.0074818041002643965, 0.00011714915531600665, 0)}})

In [31]: res2
Out[31]: 
{800: {'No index': {70: (0.028493346513382027, 0.003557600128486706, 1054),
   75: (0.026977396982588935, 0.0006864039211334353, 1054),
   76: (0.02487557285598346, 0.0013462340968643616, 1050),
   76.5: (0.022364463124956404, 0.0006185980448433306, 986),
   77: (0.01787294867714601, 0.0005388582978902705, 590),
   77.5: (0.015337911140439767, 0.0003835373308526435, 329),
   78: (0.014300070971782718, 0.00031291711810474, 240),
   79: (0.012529546873910087, 0.0003269927546216843, 64),
   100: (0.01203559348226658, 0.0001448220888627054, 0)},
  'Index': {70: (0.02970689120037215, 0.001096742590370909, 1054),
   75: (0.02914605591712253, 0.001176741921586655, 1054),
   76: (0.02840630336265479, 0.001065119594937484, 1050),
   76.5: (0.021173634606280498, 0.0009716046601165061, 986),
   77: (0.01735409886177097, 0.0009599443502263486, 590),
   77.5: (0.013087128465995192, 0.0005436785108917917, 329),
   78: (0.012076096493484718, 0.0002683994997422064, 240),
   79: (0.009333554677931327, 0.0002341985010720023, 64),
   100: (0.007913854884515915, 0.00021666838770041468, 0)}},
 100: {'No index': {70: (0.010013244921075446, 0.0001764110239399362, 128),
   75: (0.010155446200764605, 0.0003360788990035585, 128),
   76: (0.009958488683083227, 0.0002536822135181791, 128),
   76.5: (0.009968961138012154, 0.0001797973823596132, 119),
   77: (0.007802146435050028, 0.00045404572144724, 67),
   77.5: (0.008562839648553304, 0.00021799177136637947, 33),
   78: (0.008522078070257393, 0.0001607263392836673, 23),
   79: (0.008493307820920434, 0.00016658054323853054, 6),
   100: (0.00773597971403173, 0.00018611503950054697, 0)},
  'Index': {70: (0.009804060428536364, 0.00038446509828390875, 128),
   75: (0.009932666611192482, 0.00024151859619545593, 128),
   76: (0.010006559283605642, 0.00032723211704371847, 128),
   76.5: (0.009351666628250054, 0.00048615873021256337, 119),
   77: (0.00876571058029575, 0.0003911189096031593, 67),
   77.5: (0.008184758604371121, 0.0003360898368287055, 33),
   78: (0.007853612736133593, 0.0004471635691679575, 23),
   79: (0.007664346237267766, 0.00015709078440174666, 6),
   100: (0.007527071768417954, 0.00039844052469151285, 0)}},
 10: {'No index': {70: (0.008041770490152495, 0.00031698074773842317, 15),
   75: (0.0072174773125776226, 0.00013972553089447342, 15),
   76: (0.00745961337616401, 0.0004194008486447274, 15),
   76.5: (0.008022567416940418, 0.00033967950817123096, 14),
   77: (0.00801963801628777, 0.00024676797752452547, 9),
   77.5: (0.007957289974604334, 0.00019374517367913975, 4),
   78: (0.008121860810954656, 0.00034988860680218204, 1),
   79: (0.007809665611545954, 0.00044912731632021634, 0),
   100: (0.007728229372629097, 0.000268642950174923, 0)},
  'Index': {70: (0.00830915835686028, 0.00023492409318655758, 15),
   75: (0.008397924781643919, 0.0003686478491755673, 15),
   76: (0.008417240057938865, 0.00017610013524301627, 15),
   76.5: (0.008427468491718174, 0.00040971289037902524, 14),
   77: (0.008133986459246704, 0.00024825076484186995, 9),
   77.5: (0.008017260426921503, 0.00024235725827028982, 4),
   78: (0.007804803276168448, 0.0002022518603534982, 1),
   79: (0.0077954007924667425, 0.00024081366730153476, 0),
   100: (0.0076052043799843105, 0.00022426132570790809, 0)}},
 1: {'No index': {70: (0.007354256454855204, 0.0002565781724687829, 3),
   75: (0.007449815931863018, 0.0002035363791686122, 3),
   76: (0.007650200128555298, 0.00020565761553168399, 3),
   76.5: (0.007397548030795795, 0.00030326009351314186, 3),
   77: (0.007233466113518391, 0.00012416804973618421, 0),
   77.5: (0.0074686315815363615, 0.00015598785532553662, 0),
   78: (0.0075541199211563385, 0.00011214290614529302, 0),
   79: (0.007304560683135476, 0.0001827480692183891, 0),
   100: (0.007537868476605841, 0.00029727576206591396, 0)},
  'Index': {70: (0.007642260768583843, 0.000195874743873633, 3),
   75: (0.007568490607663989, 0.00016326846325433455, 3),
   76: (0.007555569442255157, 0.00014361202797880674, 3),
   76.5: (0.007588767239025661, 0.000259293488587246, 3),
   77: (0.007260604251974395, 0.00024506818565012936, 0),
   77.5: (0.0077381157036870716, 0.00030247916611908824, 0),
   78: (0.00741843394802085, 0.0001985791540793954, 0),
   79: (0.007549205729737878, 0.0002387371231758983, 0),
   100: (0.0074818041002643965, 0.00011714915531600665, 0)}}}

In [48]: def print_percid_data(target, data):
    ...:     print(target)
    ...:     sizes = sorted(data.keys())
    ...:     headers = ['%ID']
    ...:     for size in sizes:
    ...:         headers.extend([f'{size}M edges, default index', f'{size}M edges, compound index', f'{size}M edges, count'])
    ...:     print(f'|{"|".join(headers)}|')
    ...:     print(f'|{"|".join(["--"] * len(headers))}|')
    ...:     perc_ids = sorted(data[sizes[0]]['Index'].keys())
    ...:     for pid in perc_ids:
    ...:         line = [str(pid)]
    ...:         for size in sizes:
    ...:            line.extend([str(data[size]['No index'][pid][0]), str(data[size]['Index'][pid][0]), str(data[size]['Index'][pid][2])])
    ...:         print(f'|{"|".join(line)}|')
    ...: 

In [49]: print_percid_data(targets[1], res2)
GCA_001043795
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.007354256454855204|0.007642260768583843|3|0.008041770490152495|0.00830915835686028|15|0.010013244921075446|0.009804060428536364|128|0.028493346513382027|0.02970689120037215|1054|
|75|0.007449815931863018|0.007568490607663989|3|0.0072174773125776226|0.008397924781643919|15|0.010155446200764605|0.009932666611192482|128|0.026977396982588935|0.02914605591712253|1054|
|76|0.007650200128555298|0.007555569442255157|3|0.00745961337616401|0.008417240057938865|15|0.009958488683083227|0.010006559283605642|128|0.02487557285598346|0.02840630336265479|1050|
|76.5|0.007397548030795795|0.007588767239025661|3|0.008022567416940418|0.008427468491718174|14|0.009968961138012154|0.009351666628250054|119|0.022364463124956404|0.021173634606280498|986|
|77|0.007233466113518391|0.007260604251974395|0|0.00801963801628777|0.008133986459246704|9|0.007802146435050028|0.00876571058029575|67|0.01787294867714601|0.01735409886177097|590|
|77.5|0.0074686315815363615|0.0077381157036870716|0|0.007957289974604334|0.008017260426921503|4|0.008562839648553304|0.008184758604371121|33|0.015337911140439767|0.013087128465995192|329|
|78|0.0075541199211563385|0.00741843394802085|0|0.008121860810954656|0.007804803276168448|1|0.008522078070257393|0.007853612736133593|23|0.014300070971782718|0.012076096493484718|240|
|79|0.007304560683135476|0.007549205729737878|0|0.007809665611545954|0.0077954007924667425|0|0.008493307820920434|0.007664346237267766|6|0.012529546873910087|0.009333554677931327|64|
|100|0.007537868476605841|0.0074818041002643965|0|0.007728229372629097|0.0076052043799843105|0|0.00773597971403173|0.007527071768417954|0|0.01203559348226658|0.007913854884515915|0|

```

GCA_001043795
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.007354256454855204|0.007642260768583843|3|0.008041770490152495|0.00830915835686028|15|0.010013244921075446|0.009804060428536364|128|0.028493346513382027|0.02970689120037215|1054|
|75|0.007449815931863018|0.007568490607663989|3|0.0072174773125776226|0.008397924781643919|15|0.010155446200764605|0.009932666611192482|128|0.026977396982588935|0.02914605591712253|1054|
|76|0.007650200128555298|0.007555569442255157|3|0.00745961337616401|0.008417240057938865|15|0.009958488683083227|0.010006559283605642|128|0.02487557285598346|0.02840630336265479|1050|
|76.5|0.007397548030795795|0.007588767239025661|3|0.008022567416940418|0.008427468491718174|14|0.009968961138012154|0.009351666628250054|119|0.022364463124956404|0.021173634606280498|986|
|77|0.007233466113518391|0.007260604251974395|0|0.00801963801628777|0.008133986459246704|9|0.007802146435050028|0.00876571058029575|67|0.01787294867714601|0.01735409886177097|590|
|77.5|0.0074686315815363615|0.0077381157036870716|0|0.007957289974604334|0.008017260426921503|4|0.008562839648553304|0.008184758604371121|33|0.015337911140439767|0.013087128465995192|329|
|78|0.0075541199211563385|0.00741843394802085|0|0.008121860810954656|0.007804803276168448|1|0.008522078070257393|0.007853612736133593|23|0.014300070971782718|0.012076096493484718|240|
|79|0.007304560683135476|0.007549205729737878|0|0.007809665611545954|0.0077954007924667425|0|0.008493307820920434|0.007664346237267766|6|0.012529546873910087|0.009333554677931327|64|
|100|0.007537868476605841|0.0074818041002643965|0|0.007728229372629097|0.0076052043799843105|0|0.00773597971403173|0.007527071768417954|0|0.01203559348226658|0.007913854884515915|0|

```
In [50]: res = run_queries(targets[2], [70, 74, 74.5, 75, 75.5, 76, 76.5, 77, 100])
FastANI_800M 70
218 ms ± 40 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 74
185 ms ± 9.54 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 74.5
173 ms ± 5.41 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 75
149 ms ± 7.03 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 75.5
78.1 ms ± 2 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 76
62.4 ms ± 2.1 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 76.5
40.5 ms ± 2.23 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 77
35.6 ms ± 986 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 100
30 ms ± 1.4 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 70
204 ms ± 8.68 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M_idscore_index 74
208 ms ± 5.24 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 74.5
201 ms ± 4.72 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 75
155 ms ± 4.75 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 75.5
54.2 ms ± 1.34 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 76
35.1 ms ± 1.98 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 76.5
17 ms ± 818 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 77
9.46 ms ± 481 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 100
7.53 ms ± 195 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 70
30.8 ms ± 1.03 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 74
29.4 ms ± 1.23 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 74.5
28.7 ms ± 1.23 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 75
22 ms ± 1.51 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 75.5
15.4 ms ± 726 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 76
14.3 ms ± 324 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 76.5
12.7 ms ± 336 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 77
11.3 ms ± 329 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 100
10.8 ms ± 549 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 70
29.7 ms ± 1.43 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 74
29.6 ms ± 1.37 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 74.5
30.6 ms ± 931 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 75
21.8 ms ± 598 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 75.5
11.6 ms ± 1.03 ms per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 76
9.98 ms ± 340 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 76.5
8.12 ms ± 283 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 77
7.92 ms ± 585 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 100
7.38 ms ± 324 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 70
10.1 ms ± 236 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 74
9.97 ms ± 322 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 74.5
10.3 ms ± 299 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 75
9.78 ms ± 445 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 75.5
8.61 ms ± 245 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 76
8.48 ms ± 216 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 76.5
8.14 ms ± 269 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 77
8.59 ms ± 343 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 100
8.39 ms ± 241 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 70
10.7 ms ± 205 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 74
9.59 ms ± 264 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 74.5
10.2 ms ± 276 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 75
8.93 ms ± 506 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 75.5
8.18 ms ± 294 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 76
7.89 ms ± 262 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 76.5
7.74 ms ± 189 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 77
7.58 ms ± 160 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 100
7.34 ms ± 254 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 70
7.62 ms ± 179 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 74
7.69 ms ± 200 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 74.5
7.77 ms ± 204 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 75
8.09 ms ± 323 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 75.5
7.32 ms ± 348 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 76
7.37 ms ± 520 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 76.5
7.7 ms ± 191 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 77
7.29 ms ± 320 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 100
7.91 ms ± 245 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 70
8.42 ms ± 232 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 74
8.44 ms ± 898 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 74.5
8.19 ms ± 232 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 75
7.76 ms ± 484 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 75.5
7.45 ms ± 259 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 76
7.14 ms ± 300 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 76.5
7.15 ms ± 259 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 77
7.41 ms ± 242 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 100
7.7 ms ± 177 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)

In [51]: res2 = reformat_results(res)

In [52]: res
Out[52]: 
defaultdict(dict,
            {'FastANI_800M': {70: (0.21840401605835982,
               0.04000038769226781,
               10735),
              74: (0.18485029001853295, 0.009544601872591428, 10735),
              74.5: (0.1725548486358353, 0.005414843955214917, 10735),
              75: (0.1493768601545266, 0.007034164297643157, 8037),
              75.5: (0.07807520856814724, 0.0019966173233603945, 2550),
              76: (0.062413491095815385, 0.002099247427628218, 1572),
              76.5: (0.040494116076401306, 0.0022347257862605076, 641),
              77: (0.03557166285546763, 0.0009862789204698914, 159),
              100: (0.030020569930119173, 0.0013973833900229953, 0)},
             'FastANI_800M_idscore_index': {70: (0.2038467884329813,
               0.00868344905249401,
               10735),
              74: (0.20816588113084436, 0.00524373731668442, 10735),
              74.5: (0.20116468280819907, 0.004718178708645463, 10735),
              75: (0.15512518288035476, 0.004751516988243567, 8037),
              75.5: (0.054178431976054396, 0.0013390151329499985, 2550),
              76: (0.03513734703883529, 0.001980986447927007, 1572),
              76.5: (0.01699614233470389, 0.0008179592093664889, 641),
              77: (0.00946076458852206, 0.0004806730284120001, 159),
              100: (0.007527031796053052, 0.00019451475120622802, 0)},
             'FastANI_100M': {70: (0.030769232580704346,
               0.0010293019361201244,
               1331),
              74: (0.02940999511629343, 0.0012258040823628752, 1331),
              74.5: (0.02873086802927511, 0.0012293897107329428, 1331),
              75: (0.022026362682559662, 0.001507762740790803, 975),
              75.5: (0.01535222260680582, 0.000726037806848094, 299),
              76: (0.014296830746212177, 0.0003239170145842167, 187),
              76.5: (0.012679125726489084, 0.0003363272192671734, 70),
              77: (0.011294680667508925, 0.00032933398513560337, 21),
              100: (0.010840717313278998, 0.0005490978467431186, 0)},
             'FastANI_100M_idscore_index': {70: (0.02970072044325726,
               0.0014346041103173441,
               1331),
              74: (0.02962130821709122, 0.0013682209773694834, 1331),
              74.5: (0.030562174785882235, 0.0009310151397849516, 1331),
              75: (0.021789178664663008, 0.0005984918912868486, 975),
              75.5: (0.01163339254978512, 0.001025788906989133, 299),
              76: (0.009984190352261066, 0.00034015152098682894, 187),
              76.5: (0.008119713383327638, 0.0002831277440770741, 70),
              77: (0.007917768540126937, 0.0005852702827153576, 21),
              100: (0.007384324346535972, 0.00032448917634502936, 0)},
             'FastANI_10M': {70: (0.010106780610180326,
               0.00023591482599429254,
               154),
              74: (0.009966125313990883, 0.00032221750299054794, 154),
              74.5: (0.010333836367353797, 0.00029924198082146273, 154),
              75: (0.009778665893578103, 0.00044522207381436093, 108),
              75.5: (0.00860786525266511, 0.00024500505999567766, 35),
              76: (0.008481095183108535, 0.00021557128302250275, 18),
              76.5: (0.008140744646745069, 0.00026872115111345425, 4),
              77: (0.008592831447188344, 0.0003431547485048257, 1),
              100: (0.008385563410286394, 0.0002408513810481565, 0)},
             'FastANI_10M_idscore_index': {70: (0.010675037420753921,
               0.00020502706222500395,
               154),
              74: (0.009587977419474295, 0.0002639285983700296, 154),
              74.5: (0.010215386486213123, 0.00027582628380707784, 154),
              75: (0.008926672342100313, 0.0005055397535534163, 108),
              75.5: (0.008179001160232085, 0.0002937097820312462, 35),
              76: (0.007891874365242463, 0.0002620171959068225, 18),
              76.5: (0.007743764208363635, 0.00018938500147244687, 4),
              77: (0.0075756694842129945, 0.00015988340701374965, 1),
              100: (0.007336162715884192, 0.0002540098852550406, 0)},
             'FastANI_1M': {70: (0.007618863402998873,
               0.00017916536343574605,
               27),
              74: (0.007693638651232634, 0.00019994960008331032, 27),
              74.5: (0.007769025432478104, 0.00020435233784188347, 27),
              75: (0.008089844434122953, 0.0003227445566563552, 20),
              75.5: (0.007320615163605128, 0.00034832398257845154, 3),
              76: (0.007374742017792804, 0.0005203445367233759, 1),
              76.5: (0.007697477778419852, 0.00019106992018241945, 0),
              77: (0.007285316796707255, 0.0003198002012292695, 0),
              100: (0.00791028003075293, 0.00024540651254574634, 0)},
             'FastANI_1M_idscore_index': {70: (0.008423557752477271,
               0.00023232981668811157,
               27),
              74: (0.008437648378312589, 0.0008981298978543305, 27),
              74.5: (0.00818960582172232, 0.00023197693180722643, 27),
              75: (0.007764529158760395, 0.00048449975051893183, 20),
              75.5: (0.00745002989258085, 0.0002593630205823063, 3),
              76: (0.007143870562847171, 0.00030027272474971253, 1),
              76.5: (0.0071458235822085825, 0.00025890946423907, 0),
              77: (0.007409968925639987, 0.0002420398161406251, 0),
              100: (0.007700743280085069, 0.00017681101204734232, 0)}})

In [53]: res2
Out[53]: 
{800: {'No index': {70: (0.21840401605835982, 0.04000038769226781, 10735),
   74: (0.18485029001853295, 0.009544601872591428, 10735),
   74.5: (0.1725548486358353, 0.005414843955214917, 10735),
   75: (0.1493768601545266, 0.007034164297643157, 8037),
   75.5: (0.07807520856814724, 0.0019966173233603945, 2550),
   76: (0.062413491095815385, 0.002099247427628218, 1572),
   76.5: (0.040494116076401306, 0.0022347257862605076, 641),
   77: (0.03557166285546763, 0.0009862789204698914, 159),
   100: (0.030020569930119173, 0.0013973833900229953, 0)},
  'Index': {70: (0.2038467884329813, 0.00868344905249401, 10735),
   74: (0.20816588113084436, 0.00524373731668442, 10735),
   74.5: (0.20116468280819907, 0.004718178708645463, 10735),
   75: (0.15512518288035476, 0.004751516988243567, 8037),
   75.5: (0.054178431976054396, 0.0013390151329499985, 2550),
   76: (0.03513734703883529, 0.001980986447927007, 1572),
   76.5: (0.01699614233470389, 0.0008179592093664889, 641),
   77: (0.00946076458852206, 0.0004806730284120001, 159),
   100: (0.007527031796053052, 0.00019451475120622802, 0)}},
 100: {'No index': {70: (0.030769232580704346, 0.0010293019361201244, 1331),
   74: (0.02940999511629343, 0.0012258040823628752, 1331),
   74.5: (0.02873086802927511, 0.0012293897107329428, 1331),
   75: (0.022026362682559662, 0.001507762740790803, 975),
   75.5: (0.01535222260680582, 0.000726037806848094, 299),
   76: (0.014296830746212177, 0.0003239170145842167, 187),
   76.5: (0.012679125726489084, 0.0003363272192671734, 70),
   77: (0.011294680667508925, 0.00032933398513560337, 21),
   100: (0.010840717313278998, 0.0005490978467431186, 0)},
  'Index': {70: (0.02970072044325726, 0.0014346041103173441, 1331),
   74: (0.02962130821709122, 0.0013682209773694834, 1331),
   74.5: (0.030562174785882235, 0.0009310151397849516, 1331),
   75: (0.021789178664663008, 0.0005984918912868486, 975),
   75.5: (0.01163339254978512, 0.001025788906989133, 299),
   76: (0.009984190352261066, 0.00034015152098682894, 187),
   76.5: (0.008119713383327638, 0.0002831277440770741, 70),
   77: (0.007917768540126937, 0.0005852702827153576, 21),
   100: (0.007384324346535972, 0.00032448917634502936, 0)}},
 10: {'No index': {70: (0.010106780610180326, 0.00023591482599429254, 154),
   74: (0.009966125313990883, 0.00032221750299054794, 154),
   74.5: (0.010333836367353797, 0.00029924198082146273, 154),
   75: (0.009778665893578103, 0.00044522207381436093, 108),
   75.5: (0.00860786525266511, 0.00024500505999567766, 35),
   76: (0.008481095183108535, 0.00021557128302250275, 18),
   76.5: (0.008140744646745069, 0.00026872115111345425, 4),
   77: (0.008592831447188344, 0.0003431547485048257, 1),
   100: (0.008385563410286394, 0.0002408513810481565, 0)},
  'Index': {70: (0.010675037420753921, 0.00020502706222500395, 154),
   74: (0.009587977419474295, 0.0002639285983700296, 154),
   74.5: (0.010215386486213123, 0.00027582628380707784, 154),
   75: (0.008926672342100313, 0.0005055397535534163, 108),
   75.5: (0.008179001160232085, 0.0002937097820312462, 35),
   76: (0.007891874365242463, 0.0002620171959068225, 18),
   76.5: (0.007743764208363635, 0.00018938500147244687, 4),
   77: (0.0075756694842129945, 0.00015988340701374965, 1),
   100: (0.007336162715884192, 0.0002540098852550406, 0)}},
 1: {'No index': {70: (0.007618863402998873, 0.00017916536343574605, 27),
   74: (0.007693638651232634, 0.00019994960008331032, 27),
   74.5: (0.007769025432478104, 0.00020435233784188347, 27),
   75: (0.008089844434122953, 0.0003227445566563552, 20),
   75.5: (0.007320615163605128, 0.00034832398257845154, 3),
   76: (0.007374742017792804, 0.0005203445367233759, 1),
   76.5: (0.007697477778419852, 0.00019106992018241945, 0),
   77: (0.007285316796707255, 0.0003198002012292695, 0),
   100: (0.00791028003075293, 0.00024540651254574634, 0)},
  'Index': {70: (0.008423557752477271, 0.00023232981668811157, 27),
   74: (0.008437648378312589, 0.0008981298978543305, 27),
   74.5: (0.00818960582172232, 0.00023197693180722643, 27),
   75: (0.007764529158760395, 0.00048449975051893183, 20),
   75.5: (0.00745002989258085, 0.0002593630205823063, 3),
   76: (0.007143870562847171, 0.00030027272474971253, 1),
   76.5: (0.0071458235822085825, 0.00025890946423907, 0),
   77: (0.007409968925639987, 0.0002420398161406251, 0),
   100: (0.007700743280085069, 0.00017681101204734232, 0)}}}

In [54]: print_percid_data(targets[2], res2)
GCA_001899065
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.007618863402998873|0.008423557752477271|27|0.010106780610180326|0.010675037420753921|154|0.030769232580704346|0.02970072044325726|1331|0.21840401605835982|0.2038467884329813|10735|
|74|0.007693638651232634|0.008437648378312589|27|0.009966125313990883|0.009587977419474295|154|0.02940999511629343|0.02962130821709122|1331|0.18485029001853295|0.20816588113084436|10735|
|74.5|0.007769025432478104|0.00818960582172232|27|0.010333836367353797|0.010215386486213123|154|0.02873086802927511|0.030562174785882235|1331|0.1725548486358353|0.20116468280819907|10735|
|75|0.008089844434122953|0.007764529158760395|20|0.009778665893578103|0.008926672342100313|108|0.022026362682559662|0.021789178664663008|975|0.1493768601545266|0.15512518288035476|8037|
|75.5|0.007320615163605128|0.00745002989258085|3|0.00860786525266511|0.008179001160232085|35|0.01535222260680582|0.01163339254978512|299|0.07807520856814724|0.054178431976054396|2550|
|76|0.007374742017792804|0.007143870562847171|1|0.008481095183108535|0.007891874365242463|18|0.014296830746212177|0.009984190352261066|187|0.062413491095815385|0.03513734703883529|1572|
|76.5|0.007697477778419852|0.0071458235822085825|0|0.008140744646745069|0.007743764208363635|4|0.012679125726489084|0.008119713383327638|70|0.040494116076401306|0.01699614233470389|641|
|77|0.007285316796707255|0.007409968925639987|0|0.008592831447188344|0.0075756694842129945|1|0.011294680667508925|0.007917768540126937|21|0.03557166285546763|0.00946076458852206|159|
|100|0.00791028003075293|0.007700743280085069|0|0.008385563410286394|0.007336162715884192|0|0.010840717313278998|0.007384324346535972|0|0.030020569930119173|0.007527031796053052|0|
```

GCA_001899065
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.007618863402998873|0.008423557752477271|27|0.010106780610180326|0.010675037420753921|154|0.030769232580704346|0.02970072044325726|1331|0.21840401605835982|0.2038467884329813|10735|
|74|0.007693638651232634|0.008437648378312589|27|0.009966125313990883|0.009587977419474295|154|0.02940999511629343|0.02962130821709122|1331|0.18485029001853295|0.20816588113084436|10735|
|74.5|0.007769025432478104|0.00818960582172232|27|0.010333836367353797|0.010215386486213123|154|0.02873086802927511|0.030562174785882235|1331|0.1725548486358353|0.20116468280819907|10735|
|75|0.008089844434122953|0.007764529158760395|20|0.009778665893578103|0.008926672342100313|108|0.022026362682559662|0.021789178664663008|975|0.1493768601545266|0.15512518288035476|8037|
|75.5|0.007320615163605128|0.00745002989258085|3|0.00860786525266511|0.008179001160232085|35|0.01535222260680582|0.01163339254978512|299|0.07807520856814724|0.054178431976054396|2550|
|76|0.007374742017792804|0.007143870562847171|1|0.008481095183108535|0.007891874365242463|18|0.014296830746212177|0.009984190352261066|187|0.062413491095815385|0.03513734703883529|1572|
|76.5|0.007697477778419852|0.0071458235822085825|0|0.008140744646745069|0.007743764208363635|4|0.012679125726489084|0.008119713383327638|70|0.040494116076401306|0.01699614233470389|641|
|77|0.007285316796707255|0.007409968925639987|0|0.008592831447188344|0.0075756694842129945|1|0.011294680667508925|0.007917768540126937|21|0.03557166285546763|0.00946076458852206|159|
|100|0.00791028003075293|0.007700743280085069|0|0.008385563410286394|0.007336162715884192|0|0.010840717313278998|0.007384324346535972|0|0.030020569930119173|0.007527031796053052|0|

```
In [55]: res = run_queries(targets[3], [70, 74, 74.5, 75, 75.5, 76, 100])
FastANI_800M 70
1.32 s ± 111 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 74
1.24 s ± 80.2 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 74.5
1.21 s ± 53.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 75
867 ms ± 27.7 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 75.5
203 ms ± 5.2 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M 76
160 ms ± 16.4 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M 100
152 ms ± 10.6 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 70
1.23 s ± 91.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M_idscore_index 74
1.23 s ± 26.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M_idscore_index 74.5
1.23 s ± 50.1 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M_idscore_index 75
887 ms ± 27.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_800M_idscore_index 75.5
39.5 ms ± 3.89 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_800M_idscore_index 76
9.82 ms ± 888 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_800M_idscore_index 100
7.48 ms ± 216 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M 70
165 ms ± 6.97 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_100M 74
169 ms ± 3.94 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 74.5
170 ms ± 9.12 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 75
128 ms ± 5.09 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 75.5
34.8 ms ± 1.43 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 76
27.2 ms ± 1.75 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M 100
27.9 ms ± 1.68 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 70
191 ms ± 18.5 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
FastANI_100M_idscore_index 74
178 ms ± 6.29 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 74.5
185 ms ± 10.8 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 75
131 ms ± 6.35 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_100M_idscore_index 75.5
12.3 ms ± 480 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 76
8.16 ms ± 163 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_100M_idscore_index 100
7.46 ms ± 186 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 70
28.7 ms ± 2.86 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M 74
26 ms ± 1.28 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M 74.5
27 ms ± 1.37 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M 75
19.1 ms ± 300 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 75.5
10.6 ms ± 430 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 76
10.7 ms ± 216 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M 100
9.78 ms ± 377 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 70
25.4 ms ± 1.16 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M_idscore_index 74
26.8 ms ± 1.65 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M_idscore_index 74.5
28.8 ms ± 1.04 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M_idscore_index 75
19.3 ms ± 985 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
FastANI_10M_idscore_index 75.5
7.97 ms ± 504 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 76
6.79 ms ± 328 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_10M_idscore_index 100
7.6 ms ± 151 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 70
10.6 ms ± 385 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 74
10.1 ms ± 716 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 74.5
9.95 ms ± 331 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 75
9.11 ms ± 416 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 75.5
7.84 ms ± 325 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 76
8.04 ms ± 130 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M 100
8.09 ms ± 262 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 70
10.4 ms ± 430 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 74
10.5 ms ± 322 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 74.5
11.1 ms ± 303 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 75
9.99 ms ± 382 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 75.5
7.34 ms ± 429 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 76
7.22 ms ± 175 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
FastANI_1M_idscore_index 100
7 ms ± 292 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)

In [56]: res2 = reformat_results(res)

In [57]: res
Out[57]: 
defaultdict(dict,
            {'FastANI_800M': {70: (1.3164149029180408,
               0.11074052687025229,
               75742),
              74: (1.2424138292138065, 0.08024230638097421, 75742),
              74.5: (1.2119144221235598, 0.05331521831726635, 75738),
              75: (0.8671864989612784, 0.02772441064827615, 53987),
              75.5: (0.20257084563906705, 0.0051997674440188765, 1996),
              76: (0.15969007474237257, 0.01642741074180528, 144),
              100: (0.1522687048651278, 0.010615731248378928, 0)},
             'FastANI_800M_idscore_index': {70: (1.2273640327953868,
               0.09180896265995622,
               75742),
              74: (1.230183769017458, 0.02675073963847515, 75742),
              74.5: (1.2303273579371827, 0.05014704005621964, 75738),
              75: (0.8868207177147269, 0.027592522152504243, 53987),
              75.5: (0.03947870755302055, 0.003892399120297308, 1996),
              76: (0.009820263417703765, 0.0008879753474790217, 144),
              100: (0.007477043235142316, 0.00021646996621326267, 0)},
             'FastANI_100M': {70: (0.16505602427891322,
               0.006970730979256991,
               9672),
              74: (0.16914036948499936, 0.0039399438925346135, 9672),
              74.5: (0.17011757313406892, 0.00911711125536625, 9671),
              75: (0.1275880230324609, 0.005088002896611981, 6878),
              75.5: (0.034764595742204356, 0.0014261061041255898, 269),
              76: (0.027173222481672255, 0.0017463624129536084, 25),
              100: (0.027918045701725142, 0.001680294726735482, 0)},
             'FastANI_100M_idscore_index': {70: (0.19066272529640368,
               0.018540303130775312,
               9672),
              74: (0.17769963731989263, 0.006291966445413841, 9672),
              74.5: (0.18456687917932868, 0.010848827790178298, 9671),
              75: (0.13134569403316293, 0.006354011948555056, 6878),
              75.5: (0.012296781255198376, 0.00047965529749348444, 269),
              76: (0.008162662865860123, 0.00016323082964936786, 25),
              100: (0.007461740091176969, 0.0001855599273217391, 0)},
             'FastANI_10M': {70: (0.02870725630117314,
               0.0028622330134834206,
               1157),
              74: (0.026005283570183176, 0.001283882737687499, 1157),
              74.5: (0.026968418766877483, 0.0013663221200504298, 1157),
              75: (0.019063042690977455, 0.00030048523766229183, 820),
              75.5: (0.010643383719559227, 0.00042982153340048084, 31),
              76: (0.010732984707823821, 0.0002164598195721771, 7),
              100: (0.009782045599339264, 0.0003774025452681914, 0)},
             'FastANI_10M_idscore_index': {70: (0.025365594441869428,
               0.0011593280773297676,
               1157),
              74: (0.02680653070232698, 0.0016524863296999003, 1157),
              74.5: (0.028776096033730676, 0.0010436131999026943, 1157),
              75: (0.019273441963429963, 0.000984737682480205, 820),
              75.5: (0.007973976171176348, 0.0005042330396128253, 31),
              76: (0.006788323496335319, 0.00032764627636280126, 7),
              100: (0.007602815701227102, 0.00015086889628080705, 0)},
             'FastANI_1M': {70: (0.01057246430510921,
               0.00038548075157751756,
               189),
              74: (0.010054575368495924, 0.0007158417672754033, 189),
              74.5: (0.009953754962022814, 0.00033068517009554203, 189),
              75: (0.009106716956677181, 0.00041553724932298473, 137),
              75.5: (0.007841865082404443, 0.00032497689850727253, 5),
              76: (0.008043607901781798, 0.00013000062436777543, 1),
              100: (0.008092965614050627, 0.0002623494930384889, 0)},
             'FastANI_1M_idscore_index': {70: (0.010368073947195496,
               0.0004298235798058846,
               189),
              74: (0.010450302212099945, 0.0003219277834274701, 189),
              74.5: (0.011102301113839662, 0.0003026790453018064, 189),
              75: (0.009990253381963287, 0.000382495024208171, 137),
              75.5: (0.007335341186927897, 0.0004285775219499855, 5),
              76: (0.007218612601448383, 0.00017512537519183635, 1),
              100: (0.007002200335264206, 0.00029213252727474374, 0)}})

In [58]: res2
Out[58]: 
{800: {'No index': {70: (1.3164149029180408, 0.11074052687025229, 75742),
   74: (1.2424138292138065, 0.08024230638097421, 75742),
   74.5: (1.2119144221235598, 0.05331521831726635, 75738),
   75: (0.8671864989612784, 0.02772441064827615, 53987),
   75.5: (0.20257084563906705, 0.0051997674440188765, 1996),
   76: (0.15969007474237257, 0.01642741074180528, 144),
   100: (0.1522687048651278, 0.010615731248378928, 0)},
  'Index': {70: (1.2273640327953868, 0.09180896265995622, 75742),
   74: (1.230183769017458, 0.02675073963847515, 75742),
   74.5: (1.2303273579371827, 0.05014704005621964, 75738),
   75: (0.8868207177147269, 0.027592522152504243, 53987),
   75.5: (0.03947870755302055, 0.003892399120297308, 1996),
   76: (0.009820263417703765, 0.0008879753474790217, 144),
   100: (0.007477043235142316, 0.00021646996621326267, 0)}},
 100: {'No index': {70: (0.16505602427891322, 0.006970730979256991, 9672),
   74: (0.16914036948499936, 0.0039399438925346135, 9672),
   74.5: (0.17011757313406892, 0.00911711125536625, 9671),
   75: (0.1275880230324609, 0.005088002896611981, 6878),
   75.5: (0.034764595742204356, 0.0014261061041255898, 269),
   76: (0.027173222481672255, 0.0017463624129536084, 25),
   100: (0.027918045701725142, 0.001680294726735482, 0)},
  'Index': {70: (0.19066272529640368, 0.018540303130775312, 9672),
   74: (0.17769963731989263, 0.006291966445413841, 9672),
   74.5: (0.18456687917932868, 0.010848827790178298, 9671),
   75: (0.13134569403316293, 0.006354011948555056, 6878),
   75.5: (0.012296781255198376, 0.00047965529749348444, 269),
   76: (0.008162662865860123, 0.00016323082964936786, 25),
   100: (0.007461740091176969, 0.0001855599273217391, 0)}},
 10: {'No index': {70: (0.02870725630117314, 0.0028622330134834206, 1157),
   74: (0.026005283570183176, 0.001283882737687499, 1157),
   74.5: (0.026968418766877483, 0.0013663221200504298, 1157),
   75: (0.019063042690977455, 0.00030048523766229183, 820),
   75.5: (0.010643383719559227, 0.00042982153340048084, 31),
   76: (0.010732984707823821, 0.0002164598195721771, 7),
   100: (0.009782045599339264, 0.0003774025452681914, 0)},
  'Index': {70: (0.025365594441869428, 0.0011593280773297676, 1157),
   74: (0.02680653070232698, 0.0016524863296999003, 1157),
   74.5: (0.028776096033730676, 0.0010436131999026943, 1157),
   75: (0.019273441963429963, 0.000984737682480205, 820),
   75.5: (0.007973976171176348, 0.0005042330396128253, 31),
   76: (0.006788323496335319, 0.00032764627636280126, 7),
   100: (0.007602815701227102, 0.00015086889628080705, 0)}},
 1: {'No index': {70: (0.01057246430510921, 0.00038548075157751756, 189),
   74: (0.010054575368495924, 0.0007158417672754033, 189),
   74.5: (0.009953754962022814, 0.00033068517009554203, 189),
   75: (0.009106716956677181, 0.00041553724932298473, 137),
   75.5: (0.007841865082404443, 0.00032497689850727253, 5),
   76: (0.008043607901781798, 0.00013000062436777543, 1),
   100: (0.008092965614050627, 0.0002623494930384889, 0)},
  'Index': {70: (0.010368073947195496, 0.0004298235798058846, 189),
   74: (0.010450302212099945, 0.0003219277834274701, 189),
   74.5: (0.011102301113839662, 0.0003026790453018064, 189),
   75: (0.009990253381963287, 0.000382495024208171, 137),
   75.5: (0.007335341186927897, 0.0004285775219499855, 5),
   76: (0.007218612601448383, 0.00017512537519183635, 1),
   100: (0.007002200335264206, 0.00029213252727474374, 0)}}}

In [59]: print_percid_data(targets[3], res2)
GCA_000817745
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.01057246430510921|0.010368073947195496|189|0.02870725630117314|0.025365594441869428|1157|0.16505602427891322|0.19066272529640368|9672|1.3164149029180408|1.2273640327953868|75742|
|74|0.010054575368495924|0.010450302212099945|189|0.026005283570183176|0.02680653070232698|1157|0.16914036948499936|0.17769963731989263|9672|1.2424138292138065|1.230183769017458|75742|
|74.5|0.009953754962022814|0.011102301113839662|189|0.026968418766877483|0.028776096033730676|1157|0.17011757313406892|0.18456687917932868|9671|1.2119144221235598|1.2303273579371827|75738|
|75|0.009106716956677181|0.009990253381963287|137|0.019063042690977455|0.019273441963429963|820|0.1275880230324609|0.13134569403316293|6878|0.8671864989612784|0.8868207177147269|53987|
|75.5|0.007841865082404443|0.007335341186927897|5|0.010643383719559227|0.007973976171176348|31|0.034764595742204356|0.012296781255198376|269|0.20257084563906705|0.03947870755302055|1996|
|76|0.008043607901781798|0.007218612601448383|1|0.010732984707823821|0.006788323496335319|7|0.027173222481672255|0.008162662865860123|25|0.15969007474237257|0.009820263417703765|144|
|100|0.008092965614050627|0.007002200335264206|0|0.009782045599339264|0.007602815701227102|0|0.027918045701725142|0.007461740091176969|0|0.1522687048651278|0.007477043235142316|0|
```

GCA_000817745
|%ID|1M edges, default index|1M edges, compound index|1M edges, count|10M edges, default index|10M edges, compound index|10M edges, count|100M edges, default index|100M edges, compound index|100M edges, count|800M edges, default index|800M edges, compound index|800M edges, count|
|--|--|--|--|--|--|--|--|--|--|--|--|--|
|70|0.01057246430510921|0.010368073947195496|189|0.02870725630117314|0.025365594441869428|1157|0.16505602427891322|0.19066272529640368|9672|1.3164149029180408|1.2273640327953868|75742|
|74|0.010054575368495924|0.010450302212099945|189|0.026005283570183176|0.02680653070232698|1157|0.16914036948499936|0.17769963731989263|9672|1.2424138292138065|1.230183769017458|75742|
|74.5|0.009953754962022814|0.011102301113839662|189|0.026968418766877483|0.028776096033730676|1157|0.17011757313406892|0.18456687917932868|9671|1.2119144221235598|1.2303273579371827|75738|
|75|0.009106716956677181|0.009990253381963287|137|0.019063042690977455|0.019273441963429963|820|0.1275880230324609|0.13134569403316293|6878|0.8671864989612784|0.8868207177147269|53987|
|75.5|0.007841865082404443|0.007335341186927897|5|0.010643383719559227|0.007973976171176348|31|0.034764595742204356|0.012296781255198376|269|0.20257084563906705|0.03947870755302055|1996|
|76|0.008043607901781798|0.007218612601448383|1|0.010732984707823821|0.006788323496335319|7|0.027173222481672255|0.008162662865860123|25|0.15969007474237257|0.009820263417703765|144|
|100|0.008092965614050627|0.007002200335264206|0|0.009782045599339264|0.007602815701227102|0|0.027918045701725142|0.007461740091176969|0|0.1522687048651278|0.007477043235142316|0|