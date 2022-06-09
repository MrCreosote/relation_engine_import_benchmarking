# Import 800M GCA ID edges

## Importing ~800M edges

[Test data setup](../create_test_data.md#790m-gca-edges)

[Environment setup](../environment_setup.md#existing-docker-image)

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

## Import 10M edges into a collection with 780M edges

[Test data setup](../create_test_data.md#780m-cga-edges--10m-gca-edges)

[Environment setup](../environment_setup.md#existing-docker-image)

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