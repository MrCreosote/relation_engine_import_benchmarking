# Test data creation

## 100M GCA ID edges, split into 10M chunks

Since the FastANI matrix has no headers, make a headers file:

```
root@194be4682dd0:/arangobenchmark# cat data/NCBI_Prok-matrix.txt.gz.headers.GCA.key.txt 
_from,_to,idscore,_key
```

In some cases we use a simpler name for the file:
```
root@cf6859c67159:/arangobenchmark/data# cat NCBI_Prok-matrix.txt.key.headers.txt 
_from,_to,idscore,_key
```

To reduce edge sizes to ~180B from ~450B, extract just the GCA ids from the names:

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

**Note**: There are 611 duplicate edges due to synonyms in the source file with the same GCA
ID.

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

## ~790M GCA edges

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

## 780M CGA edges + 10M GCA edges

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

## Full strain names

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