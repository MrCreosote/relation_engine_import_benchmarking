# Test environment setup

## Existing docker image

Started with an existing docker image I made a couple years or so ago with ipython.
Installed the arango 3.9.1 tarball similar to setups below that start from scratch.

Shell script:
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

Ipython code:
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
```

## Rebuild docker image

Original image was nuked by persons unknown

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

Assume docker commits after adding stuff from here on out

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
```

## Rebuild docker image again to reduce size

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
```

Save the image
```
$ docker commit c1a775a9517a        gavins_ipython_image_dont_delete
sha256:18cf889418dcffd5264c2b6967105ea75299fc6f583ad67157ceb8c8ce84aa21
~/relationengine$ docker save gavins_ipython_image_dont_delete | gzip > docker_images/gavins_python_image_dont_delete.tar.gz
```

