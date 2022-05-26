Trying to figure out why loading 100M GCA results in 611 errors in the logs while
loading 100M results in 10 10M edge batches results in 620

```
root@194be4682dd0:/arangobenchmark/data# ipython
Python 3.9.5 (default, May  4 2021, 18:15:18) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.23.1 -- An enhanced Interactive Python. Type '?' for help.

In [6]: def load_logs(file_):
   ...:     with open(file_) as f:
   ...:         ret = []
   ...:         for line in f:
   ...:             if 'WARNING' in line:
   ...:                 jsonstr = line.split('offending document: ')[1]
   ...:                 j = json.loads(jsonstr)
   ...:                 ret.append((j['_from'][5:], j['_to'][5:]))
   ...:     return ret
   ...: 

In [7]: fullimport = load_logs('100Mimport.log')

In [8]: len(fullimport)
Out[8]: 611

# concatenated the batch logs into this file
In [9]: batchedimport = load_logs('NCBI_Prok-matrix.txt.gz.GCAonly.head.merged.txt.out')

In [10]: len(batchedimport)
Out[10]: 620

In [11]: len(set(batchedimport))
Out[11]: 620

In [13]: set(fullimport) - set(batchedimport)
Out[13]: set()

In [14]: set(batchedimport) - set(fullimport)
Out[14]: 
{('GCA_000444855', 'GCA_000511425'),
 ('GCA_000564735', 'GCA_900097835'),
 ('GCA_000704165', 'GCA_001111405'),
 ('GCA_001084585', 'GCA_001758445'),
 ('GCA_001195845', 'GCA_000373445'),
 ('GCA_001234345', 'GCA_900126495'),
 ('GCA_001511115', 'GCA_001402675'),
 ('GCA_001828795', 'GCA_001571265'),
 ('GCA_900085865', 'GCA_900144405')}
 ```

 Then just grepping the source files for those IDs to try and figure out what's going on

 Turns out this is the issue: https://github.com/arangodb/arangodb/issues/16337
