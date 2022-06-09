# Imports with full strain names

[Test data setup](../create_test_data.md#full-strain-names)

[Environment setup](../environment_setup.md#rebuild-docker-image)

```
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