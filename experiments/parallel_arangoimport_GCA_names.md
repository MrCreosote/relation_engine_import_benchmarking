# Test 1 arangoimport instance per coordinator

[Test data setup](../create_test_data.md#100m-gca-id-edges-split-into-10m-chunks)

[Environment setup](../environment_setup.md#rebuild-docker-image-again-to-reduce-size)

## Special data setup

```
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head100M.key.txt.gz | head -30000000 | gzip > NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head100M.key.txt.gz | tail +30000001 | head -30000000 | gzip > NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head100M.key.txt.gz | tail +60000001 | head -30000000 | gzip > NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz | wc -l
30000000
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz | wc -l
30000000
~/relationengine/testdata$ gunzip -c NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz | wc -l
30000000
```

## Using the standard load script & python code:

```
In [6]: files = ['data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz',
   ...:          'data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz',
   ...:          'data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz']

In [7]: ret = run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz***

In [8]: ret
Out[8]: 
[{'time': 289.53619480133057, 'disk': 1665199761, 'index': 2269228525},
 {'time': 321.61439847946167, 'disk': 2824724037, 'index': 3538599054},
 {'time': 348.3608248233795, 'disk': 3977928570, 'index': 4848246519}]

In [10]: print_res(ret)
|Docs loaded (M)|Cumulative time (s)|Cumulative disk size (B)|Cumulative index size (B)|
|---|---|---|---|
|10|289.53619480133057|1665199761|2269228525|
|20|611.1505932807922|2824724037|3538599054|
|30|959.5114181041718|3977928570|4848246519|
```

## Parallel processing with 1 arangoimport instance per coordinator

Requires some special setup.

Import script:
```
root@57cc052eeabb:/arangobenchmark# cat import.parameterized.host.sh 
#!/usr/bin/env sh

date
arango/3.9.1/bin/arangoimport \
    --file $INFILE \
    --headers-file data/NCBI_Prok-matrix.txt.key.headers.txt \
    --type csv \
    --separator "," \
    --progress true \
    --server.endpoint $HOST \
    --server.username gavin \
    --server.password $ARANGO_PWD_CI \
    --server.database gavin_test \
    --collection FastANI \
    --log.foreground-tty true \
    --from-collection-prefix node \
    --to-collection-prefix node \
    --threads $THREADS
date

```

ipython:

(Pretty hacky but works. Also apparently the best way to do async system call is via 
asyncio.create_subprocess_exec)

```
In [1]: import time

In [2]: import os

In [3]: import subprocess

In [4]: import arango

In [6]: files = ['data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz',
   ...:          'data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz',
   ...:          'data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz']

In [11]: def prl_run_imports(files, threads):
    ...:     pwd = os.environ['ARANGO_PWD_CI']
    ...:     acli = arango.ArangoClient(hosts='http://10.58.1.211:8531')
    ...:     db = acli.db('gavin_test', username='gavin', password=pwd)
    ...:     col = db.collection('FastANI')
    ...:     hosts = ['tcp://10.58.1.211:8531',
    ...:              'tcp://10.58.1.212:8531',
    ...:              'tcp://10.58.1.213:8531']
    ...:     f_to_proc = {}
    ...:     t1 = time.time()
    ...:     for f, h in zip(files, hosts):
    ...:         print(f"***{f} {h}***")
    ...:         f_to_proc[f] = subprocess.Popen(
    ...:             './import.parameterized.host.sh',
    ...:             stdout=subprocess.PIPE,
    ...:             stderr=subprocess.PIPE,
    ...:             env={
    ...:                 'HOST': h,
    ...:                 'ARANGO_PWD_CI': pwd,
    ...:                 'INFILE': f,
    ...:                 'THREADS': str(threads)
    ...:                 }
    ...:             )
    ...:     for p in f_to_proc.values():
    ...:         p.wait()
    ...:     t = time.time() - t1
    ...:     for f in f_to_proc:
    ...:         res = f_to_proc[f]
    ...:         so, se = res.communicate()
    ...:         if res.returncode > 0:
    ...:             print(f)
    ...:             print("stdout")
    ...:             print(so)
    ...:             print("stderr")
    ...:             print(se)
    ...:         else:
    ...:             with open(f + ".out", 'wb') as logout:
    ...:                 logout.write(so)
    ...:     stats = col.statistics()
    ...:     return {
    ...:             'time': t,
    ...:             'disk': stats['documents_size'],
    ...:             'index': stats['indexes']['size']
    ...:             }
    ...: 

In [12]: ret = prl_run_imports(files, 5)
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz tcp://10.58.1.211:8531***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz tcp://10.58.1.212:8531***
***data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz tcp://10.58.1.213:8531***

In [13]: ret
Out[13]: {'time': 935.9380848407745, 'disk': 4249960561, 'index': 5159185669}
```

Confirm the scripts were running in parallel:
```
root@57cc052eeabb:/arangobenchmark# head -1 data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz.out data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz.out data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz.out
==> data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz.out <==
Fri Jun 10 02:33:06 UTC 2022

==> data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz.out <==
Fri Jun 10 02:33:06 UTC 2022

==> data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz.out <==
Fri Jun 10 02:33:06 UTC 2022
root@57cc052eeabb:/arangobenchmark# tail -6 data/NCBI_Prok-matrix.txt.gz.GCAonly.head0-30M.key.txt.gz.out 
created:          29999804
warnings/errors:  196
updated/replaced: 0
ignored:          0
lines read:       30000001
Fri Jun 10 02:48:41 UTC 2022
root@57cc052eeabb:/arangobenchmark# tail -6 data/NCBI_Prok-matrix.txt.gz.GCAonly.head30-60M.key.txt.gz.out 
created:          29999811
warnings/errors:  189
updated/replaced: 0
ignored:          0
lines read:       30000001
Fri Jun 10 02:48:40 UTC 2022
root@57cc052eeabb:/arangobenchmark# tail -6 data/NCBI_Prok-matrix.txt.gz.GCAonly.head60-90M.key.txt.gz.out 
created:          29999831
warnings/errors:  169
updated/replaced: 0
ignored:          0
lines read:       30000001
Fri Jun 10 02:48:42 UTC 2022
```