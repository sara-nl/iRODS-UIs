# iRODS and compute, the python API for iRODS
**Authors**
- Arthur Newton (SURFsara)
- Christine Staiger (SURFsara)

**License**
Copyright 2017 SURFsara BV

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.


## Exploring and debugging python workflow in interactive mode

### Prerequisites
- ipython
- [python-irodsclient](https://github.com/irods/python-irodsclient)

We are working a client machine providing us with the *ipython* interpreter and the python package *irods*. *ipython* offers a nice environment to go through code line by line debugging and making sure that the workflow is setup correctly. Later on we will create a script that calls the workflow and runs it on several remote worker nodes without any direct interaction of us.

We provide some useful python code in our iRODS instance under
`/aliceZone/home/arthur/pythonscripts`

**Exercise** Before we start, please download the folder to your unix home directory. (If you are working on your own iRODS instance, we will provide you wityh the code. Please contact us.)

Make sure you are in the folder:

```
cd ~/files_for_homefolder_students/pythonscripts
```

Start an ipython session:

```sh
ipython
```

We also prepared some handy python functions for you. You can find and import them from:

```py
from ensure_dir import ensure_dir
from wordcount import wordcount
```

### Creating the data directory on scratch
Lisa knows several different file systems. One is your home directory, which is accessible from every node in the lisa cluster.

Next to it you have access to a local filesystem, the scratch file system. This file system and thus all data you save there is only visible to the local node, i.e. the login node or the worker node. The path to the scratch file system follows some logic but is different for each node in the cluster. Luckily, on each node there is by default a bash variable that points to the scratch space. You can retrieve this name in python with

```py
import os
os.environ['TMPDIR'] #scratch space on our compute cluster
```

`os.environ` is an object that stores all environment variables. 
You can use them to easily build paths independen of the user who executes the script. On our compute clusters we have a variable called *TMPDIR* which always points to the respective scratch storage of the node you are working on.

**Exercise** How would you retrieve the path tp your *HOME* directory in python?

Now that we know where, we can create a temporary data directory and a temporary results directory on scratch space or your *HOME* directory (please choose accordingly). We will make use of a function we implemented for you.

```py
from helperFunctions import ensure_dir
```

```py
dataDir = os.environ['HOME']+"/dataFilesToCompute"
ensure_dir(dataDir)
resultsDir = os.environ['HOME']+"/results"
ensure_dir(resultsDir)
```

**Exercise** Inspect the *os* module in python (type in `os.` and hit the tab key). Which function can you use to list the newly created directories?

## Working in iRODS
### Connect to iRODS

If you already have logged in to iRODS with the icommands *iinit* you can create an iRODS session in python with:

```sh
from irods.session import iRODSSession
sess = iRODSSession(irods_env_file=os.environ['HOME']+'/.irods/irods_environment.json')
```
This is the securest way, since python reads the scrambled password in '.irodsA' and provides it to the server.

If you do not have such a file, we need to take security into account when creating a session. The module *getpass* asks for passwords without printing the input on screen. With en encoding function we prevent that the variable contains the plain password.

```py
import getpass
pw = getpass.getpass().encode('base64')
```

Now we can create an iRODS session:
```
from irods.session import iRODSSession
sess = iRODSSession(host='aliceZone', port=1247, user='alice', password=pw.decode('base64'), zone='aliceZone')
```

and test wheter we have access:

```py
coll = sess.collections.get("/aliceZone/home/irods-user1")
print coll.data_objects
print coll.subcollections
```

The output looks a bit strange, but we will see further down how to work with listing, fetching and creating data and metadata with the python API.

### Create a data object

Creating data in iRODS with python is a two-step process. You first create the iRODS logical path and then upload the contents. Remeber that in the *icommands* each command comes with a lot of flags, i.e. extra features. These are listed and accessible in python in the *irods.keywords* module. To create checlsums upon upload, we needed to use *iput -K* previously. In pythom we need to set a dictionary (hashtable) with these options and pass them on when we create the content of the data object:

```py
import irods.keywords as kw
options = {kw.REG_CHKSUM_KW: ''}
```
Now we create the logical path:

```py
iPath = '/aliceZone/home/irods-user1/pythonAPI.txt'
obj = sess.data_objects.create(iPath)
```
The object carries some vital system information, otherwise it is empty. 

```
myObj = sess.data_objects.get(iPath)
print "Name: ", myObj.name
print "Owner: ", myObj.owner_name
print "Size: ", myObj.size
print "Checksum:", myObj.checksum
print "Create: ", myObj.create_time
print "Modify: ", myObj.modify_time
print "Metadata: ", myObj.metadata.items()
```
Now we will modify the data object and create some contents:

```
content = 'My contents!'
with obj.open('w', options) as obj_desc:
    obj_desc.write(content)
obj = sess.data_objects.get(iPath)
```

And check what changed:

```py
myObj = sess.data_objects.get(iPath)
print "Name: ", myObj.name
print "Owner: ", myObj.owner_name
print "Size: ", myObj.size
print "Checksum:", myObj.checksum
print "Create: ", myObj.create_time
print "Modify: ", myObj.modify_time
print "Metadata: ", myObj.metadata.items()
```

### Creating metadata

To give a complete overview over data object handling with the iRODS python API, we will shortly show how to create metadata. We will dive into metadata creation at the end of this tutorial.

Working with metadata is not completely intuitive, you need a good understanding of python ductionaries and the iRODS python API classes *dataobject*, *collection*, *iRODSMetaData* and *iRODSMetaCollection*.

```py
print obj.metadata.items()
```

Create a key, value, unit entry for our data object:

```py
obj.metadata.add('SOURCE', 'python API training', 'version 1')
obj.metadata.add('TYPE', 'test file')
```
If you now print the metadata again, you will see a cryptic list:

```py
print obj.metadata.items()
```
To work with the metadata you need to iterate over them and extract the AVU triples:

```py
[(item.name, item.value, item.units) for item in obj.metadata.items()]
```

### Searching for data in iRODS

We will now translate the *iquest* command you have already explored in the first part of the tutorial to the python API. 
Recall we are looking for all files with the *author* *Lewis Carroll*. And we need to assemble the iRODS logical path.

```py
from irods.models import Collection, DataObject, CollectionMeta, DataObjectMeta
```
We need the collection name and data object name of the data objects. This command will give us all data objects we have access to:

```py
query = sess.query(Collection.name, DataObject.name)
```
Now we select the files we are interested in according to our previous *iquest* query:

```py
filteredQuery = query.filter(DataObjectMeta.name == 'author' and \
    DataObjectMeta.value == 'Lewis Carroll')
print filteredQuery.all()
```

Python prints the results neatly on the prompt, however to extract the information and parsing it to other functions is pretty complicated. Every entry you see in the output is not a string, but actually a python object with many functions. That gives you the advantage to link the output to the rows and comlumns in the sql database running in the background of iRODS. For normal user interaction, however, it needs some explanation and help.

### Parsing the iquest output
To work with the results of the query, we need to get them in an iterable format:

```py
results = filteredQuery.get_results()
```

We can now iterate over the results and build our iRODS paths (*COLL_NAME/DATA_NAME*) of the data files:

```py
iPaths = []

for item in results:
    for k in item.keys():
        if k.icat_key == 'DATA_NAME':
            name = item[k]
        elif k.icat_key == 'COLL_NAME':
            coll = item[k]
        else:
            continue
    iPaths.append(coll+'/'+name)
```

How did we know which keys to use? We asked in the query for *Collection.name* and *DataObject.name*.

Have look at these two objects:

```py
Collection.name.icat_key
DataObject.name.icat_key
```
The *icat_key* is the keyword we also use in the icommands *iquery*.
To form queries you need a good understanding of these two python classes.

**Exercise** How would you retrieve a file with a certain checksum? Search for
*eae6050a20085c1ec4dca3304836360a*.

**Solution**
The class *DataObject* carries an attribute *checksum*

```py
DataObject.checksum
```
which we can use in the query:

```py
query = sess.query(Collection.name, DataObject.name, DataObject.checksum)
filteredQuery = query.filter(DataObject.checksum == 'eae6050a20085c1ec4dca3304836360a')
print filteredQuery.all()
```
Metadata that the user creates with *imeta* or *obj.metadata.add* are accessible  via *DataObjectMeta* or *CollectionMeta* respectively. Other metadata is directly stored as attributes in *Collection* or *DataObject*.

### Retrieving the files

Now we need another translation for the *iget* command. Make sure that your variable  *dataDir* still points to right location:

```py
print dataDir
```

```py
for iPath in iPaths:
    buff = sess.data_objects.open(iPath, 'r').read()
    with open(dataDir+'/'+os.path.basename(iPath), 'wb') as f:
    f.write(buff) 
```

**Note**: The content of the files is read into memory. Hence, this only works with smaller files or on a computer with large memory. There is also no on-the-fly checksum check.

**Exercise** How can you check whether data is downloaded and ready in the data directory?

**Exercise** Retrieve the checksum from the iCAT database using python, calculate the checksum of your copied file and compare them.

**Solution** (Prepare for training as file with gaps.)

```py
import hashlib
import base64

iPaths = 
    ['/aliceZone/home/alice/books/AliceAdventuresInWonderLand.txt',
     '/aliceZone/home/alice/books/TheHuntingOfTheSnark.txt',
     '/aliceZone/home/alice/books/ThroughTheLookingGlass.txt'] 

dataDir = os.environ['HOME']+"/dataFilesToCompute"
filePaths = [dataDir +'/'+ os.path.basename(iPath) for iPath in iPaths] 

#Now for all files
iRODSchecksum = \
    [sess.data_objects.get(p).name+' :'+sess.data_objects.get(p).checksum 
        for p in iPaths]

#calculate checksum for local file
#md5
fchecksum = \
    [f +' :'+ hashlib.md5(open(f, 'rb').read()).hexdigest()
        for f in filePaths]
		
#sha256
fchecksum = \
    [f +' :'+ base64.b64encode(
        hashlib.sha256(open(f, 'rb').read()).digest()).decode()
            for f in filePaths]
```

### Starting some processing on the data
Now that the data is ready in our data directory we can start any compute workflow and program on it. Here we will count how often each word appears in the sample data. 
First we need to get a list of all files we want to analyse:

```
dataFiles = [dataDir+'/'+f for f in os.listdir(dataDir)]
```

We previously imported the function *wordcount*. Let us execute this function on the files we downloaded:

```py
from helperFunctions import wordcount
resFile = wordcount(dataFiles, resultsDir)
```

*resFile* is the path to the file with our results. Before we upload it to iRODS, let us first read the file in and inspect it a bit:

```py
import json
words = json.loads(open(resFile,"r").read())
```
The variable *words* is a python dictionary. You can now look for the occurrences of single words like this:

```py
print words['is']
print words['the']
```

**Exercise** How often is the word "Alice" mentioned in the collection. Hint: sort the list alphabetically.**

### Recursive upload of folders
In the previous step we created one file whoch can be uploaded as shown in the section "Create a data object". Now we turn to how to recursively upload a whole directory, namely our *results* folder and how to annotate it with metadata.

With the icommands recursive uploads were pretty easily done by using the flag *iput -r*. In python we need to do a bit more. We need to recurse on the file system inspecting the folder we would like to update and create data objects and collections accordingly.

The command for walking through a whole filesystem tree is:

```py
for srcDir, dirs, files in os.walk(resultsDir):
    print srcDir, dirs, files
```

We need to create the source directory in iRODS and all the files. First we determine under which collection in iRODS the data should be stored:

```py
iPath = '/'+sess.zone+'/home/'+sess.username
```

Now we walk the tree and create our subcollections and data objects:

```py
options = {kw.REG_CHKSUM_KW: ''}
for srcDir, dirs, files in os.walk(resultsDir):
    print srcDir, files
    iPath = iPath + '/' + os.path.basename(srcDir)
    print iPath
    print
    newColl = sess.collections.create(iPath)
    for fname in files:
        print newColl.path+'/'+fname
        obj = sess.data_objects.create(newColl.path+'/'+fname)
        with open(srcDir+'/'+fname, 'r') as f:
            content = f.read()
        with obj.open('w', options) as obj_desc:
            obj_desc.write(content)
```

**Exercise** Write aome code that checks all the checksums of the uploaded files. Verify that the folder hjas been transferred cortrectly.

**Solution**
We need to iterate over collections and data objects in iRODS. The python API provides a similar function *walk* for collections. However, the function does not return strings, but real python objects. That means, for each collection we receive a list of data objects and can access their system metadata directly:

```py
iPathResults = '/'+sess.zone+'/home/'+sess.username+'/'+\
    os.path.basename(resultsDir)
iColl = sess.collections.get(iPathResults)
for srcColl, subColls, objs in iColl.walk():
    print srcColl.path
    objects = [obj.path+' : '+obj.checksum+'\n' for obj in objs]
    print ''.join(objects)
```

### Link source data and results
**What information would you store? Which metadata items would you create?**

Valuable data to store:

- Date of executing the query
- The query (in case a lot of files are returned)
- The input files themselves
- ...

We need to identify the iRODS collection that holds our data:

```py
iPathResults = '/'+sess.zone+'/home/'+sess.username+'/'+\
     os.path.basename(resultsDir)
iColl = sess.collections.get(iPathResults)

for iPath in iPaths:
    iColl.metadata.add('SOURCE', iPath)
    print "Metadata added to results", iColl
```

**Exercise** Restrieve the metadata for the results collection.

**Solution** `[(item.name, item.value, item.units) for item in iColl.metadata.items()]`
