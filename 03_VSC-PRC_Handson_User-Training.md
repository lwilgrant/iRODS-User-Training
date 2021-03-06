# Introduction to Python iRODS Client (PRC) and VSC-PRC tools
This training introduces you to the basics of using the iRODS client API implemented in Python as well as the additional functions and tools developed by VSC to extend the iRODS Python API functionalities. The main feature of the VSC extensions is the possibility of using wildcards ("\*") and tildes ("~") for specifying iRODS data objects and collections. 

## Goal of this training
You will learn how to use the Python iRODS API (PRC) to interact with the Tier-1 Data service iRODS infrastructure.
The following funtionalities will be covered:

- Upload and download data
- Upload and download data collections
- Add and edit metadata
- Set access permissions for data objects and collections
- Query for data using user defined metadata
- use the VSC-PRC command line tools


## Using the VSC-PRC interactively 

This training is intended to be executed on the VSC Tier-1 (BrENIAC) system. We assume you alreay have a vsc-account and rights to connect to the Tier-1 Compute system and to use the Tier-1 Data service.

### Environment setup

- Connect to BrENIAC with your vsc-account using your favourite client. 
- Copy the course material in your home directory


```sh
git clone https://github.com/hpcleuven/iRODS-User-Training.git
``` 
- The  Python iRODS API (PRC) and the VSC extensions are available as a module in the Tier-1 Compute system.
Before using it you will need to load the corresponding module:

```sh
module load Python/3.7.4-GCCcore-8.3.0
module load vsc-python-irodsclient/development
``` 

Note that the vsc-python-irodsclient/development will load a python module if there is none already loaded.
In order to have full control about which Python version is used it is recommended to prior to load the vsc-prc-python module 
to load the Python version you want to work with.

In addition before using the Python client it is needed to start an iRODS session by executing the command:

From Tier-1 login nodes or compute nodes:

```sh
ssh irods.hpc.kuleuven.be | bash
``` 

From Tier-2 login nodes: 

```sh
ssh irods.tier1.leuven.vsc | bash
``` 

These command will activate a temporary token for a period of 7 days. After the 7 days have passed you will need to reactivate 
your access by re-executing one of these commands again.

###  Working with collections and data objects

We will first start using irods interactively with ipython. 
So we will need to install locally ipython:

```sh
pip install --user ipython
``` 
and the corresponding directory to your path:


```sh
export PATH=$HOME/.local/bin/:$PATH
``` 


Start an ipython session:

```sh
ipython
``` 

The first thing we need to do is to import the VSCIRODSSession module and create an iRODS session.
This class is derived from PRC's irods.iRODSSession class, and as such you can still use it to do what PRC is capable off (see https://github.com/irods/python-irodsclient). Here, we will focus on the functionalities that are added by VSC-PRC

```py
from vsc_irods.session import VSCiRODSSession
session = VSCiRODSSession(txt='-')
```
In addition to the keyword arguments for irods.iRODSSession, it also accepts a txt argument. This specifies where the session's print output should be directed to, with the default '-' referring to stdout.

Note that when using the VSCiRODSSession class is best practice to initiate it using the with construct to 
ensure that the session is cleanly terminated, even if an error occurs. 

The VSC-PRC library provides a number of functions grouped by functionality:

- path functions: functions related with navigation and path information 
  - get_irods_home
  - get_irods_cwd
  - get_absolute_irods_path
  - ichdir
  - imkdir 

- bulk functions: functions to perform actions on multiple collections or dataobjects in once by using wildcards ("\*") and tildes ("~") for specifying iRODS data objects and collections. 
  - get
  - put
  - remove
  - metadata
  - size 
  - add_job_metadata

- search functions: advance search capabilities either by filename or by metadata entries in the format Attribute, Value:
  - iglob
  - walk
  - find 

Let's try some of those functions:

Check wich is the iRODS home directory and de current working directory:

```py
session.path.get_irods_home()
session.path.get_irods_cwd()
```

Create a collection `training` on your iRODS home directory:

```py
session.path.imkdir('training')
```
we can verify that the collection has been created by requesting the absolute path:

```py
session.path.get_absolute_irods_path('training')
```
Now we can change to the directory we just created:


```py
session.path.ichdir('training')
```

and we can see which is now the current working directory and the home:

```py
session.path.get_irods_cwd()
session.path.get_irods_home()
```

Let's upload some data into our training iRODS collection:

First let's use the iRODS Python to upload a single dataobject:

```py
session.data_objects.put('/data/leuven/307/vsc30706/IRODS/molecules/alcl3.xyz','/kuleuven_tier1_pilot/home/vsc30706/training/')
```
This demonstrate that even using the VSCIRODSSession class all the functionalities of the Python iRODS Client (PRC) are still available. 

Let's use the search.find() function to verify that the file has been uploaded to iRODS:

```py
irods_path = '/kuleuven_tier1_pilot/home/vsc30706/training/'
session.search.find(irods_path,types='f')
```
This command returns a list of iRODS data objects paths (types='f' means list files) which match a given pattern. As we have not defined a pattern then the default value ("\*") is used. So, the result will be a list containing the irods paths of all the files in the collection `irods_path`.  

If we want to print the list of the obtained files we can do:


```py
for item in session.search.find(irods_path, types='f'):
	print(item)
```

As the PRC funtion works well has the disadvantage that only works with single data object or collections.
Here the VSC-PRC bulk functionalities provide more flexibility as they allow to select files using wildcards. 

So, let's now upload to the training iRODS collection all files with extension .xyz.

```py
session.bulk.put('./molecules/*.xyz', irods_path)
```
We can execute again the same find command we used before to veryfy that indeed all files with extension .xyz have been uploaded
while the files README and molecule_names.txt were not. 

```py
for item in session.search.find(irods_path, types='f'):
	print(item)
```
Let's now create a subdirectory `results` on our molecules collection and see how we can both list all 
subcollections and files of a given collection with the find function:

```py
session.path.imkdir('results')
for item in session.search.find(irods_path, types='d,f'):
        print(item)
```


###  Adding metadata to files 

Until now we have seen how to use the search.find function to search for collections and data objects 
based on their paths and filenames. But iRODS offer also the possibility to add metadata to them 
in the form of tuples or triples Attribute-Value-[Unit] also called AVUs. 

Let's start adding metadata to our `training` collection to identify it with the associated research project.
We will add a tuple Attribute-Value (Project, Training) and we will apply this metadata to the training collection
and all its subcollections and data objects by using the recursive option. As we have also selected the verbose 
option we will see to which collections and dataobjects the metadata is added: 

```py
avu1= ('Project', 'Training')
session.bulk.metadata(irods_path, collection_avu=avu1, action='add', recurse=True, verbose=True)
session.bulk.metadata(irods_path + '/*' , object_avu=avu1, action='add', recurse=True, verbose=True)
```

We will now create two other AVUs pairs to add to the xyz files the experiment information.
The molecules (c6h6.xyz, ch2och2.xyz and ch3cooh.xyz) where used in experiment1 while the file sih4.xyz in experiment2.
The rest of the molecules has not yet being used in any experiment. 

```py
avu2 = ('Experiment', 'Experiment1')
avu3 = ('Experiment', 'Experiment2')
session.bulk.metadata(irods_path + '/c*.xyz' , object_avu=avu2, action='add', recurse=True, verbose=True)
session.bulk.metadata(irods_path + '/sih4.xyz' , object_avu=avu3, action='add', recurse=True, verbose=True)
```

Now we can use this metadata information to perform searches on the iRODS collections.
And use this information to download all the selected files to our local directory:
Let's download all the files that were used in Experiment1  using the bulk.get function: 


```py
iterator = session.search.find(irods_path, object_avu=avu2)
session.bulk.get(iterator, local_path='.', verbose=True)
```

To finish let's clean up by removing all the `training` collection. 
We will use the option recurse=True to remove the collection and all its subcollections 
and dataobjects and the option force=True to completely remove the files (iRODS by default move it to a Trash area). 

```py
session.bulk.remove(irods_path, recurse=True, force=True, verbose=True)
```


##  Exercises 

### Exercise 1 

VSC-PRC also comes with a set of scripts which make it easy to use the
Python module from a Unix shell:

- vsc-prc-find
- vsc-prc-iget
- vsc-prc-iput
- vsc-prc-imkdir
- vsc-prc-irm
- vsc-prc-imeta
- vsc-prc-add-job-metadata

Typing e.g. :code:`vsc-prc-find --help` will show a description of the
recognized arguments. The command-line equivalents of the three Python
snippets above, for example, would look like this:

```sh 
vsc-prc-iget '~/my_irods_collection/*.txt' -d .
vsc-prc-find '~' -n '*.txt' --object_avu='Author;Me'
vsc-prc-find '~' -n '*.txt' --object_avu='Author;Me' | xargs -i vsc-prc-iget {} -d .
```

Try to reproduce the what we have done on the previous section from ipython but this time using the vsc-prc
command line tools

### Exercice 2 

Let's put everthing together. Write a python program that does the following:

- Create a directory in iRODS named `training`
- Upload the directory `molecules` as subcollection of the training directory
- Add the following metadata to the `training` collections and its subcollections:
  - Project: Training
  - PI: Homer Simpson

- Add the following metadata to the the c\*.xyz files:
  - SoftwareName: OpenBabel
  - SoftwareVersion: 3.1.1
  - SoftwareLicense: GNU General Public License, version 2

- Modify the permissions of the training collection to be readable by all the lt1_es2020 group. 
Hint: You will need to use the following  PRC functions:

  - Get the collection information: 

  ```py
  coll = session.collections.get('/kuleuven_tier1_pilot/home/vsc30706/training')
  ```

  This command will store in the coll varianble all the information related to the collection. 
  The information that can be obtained:

  - coll.id 
  - coll.path
  - coll.data_objects
  - coll.subcollections


  - Set up the new permissions by doing:

  ```py
  from irods.access import iRODSAccess
  acl = iRODSAccess('read', coll.path, 'lt1_es2020', session.zone)
  session.permissions.set(acl)
  ```

  where:

  - 'read' is the kind of permission we want to give (other option are own, write, null)
  - coll.path is tha path of the collection we want to modify obtained the previous session.collection.get()
  - 'lt1_es2020' is the name of the group to whom we want to give read access
  - session.zone is the iRODS zone to which this collection belongs and will be taken from the session information (in our case is: kuleuven_tier1_pilot

- Finally create a new local directory named `molecules.copy` and download all the files that where created with the OPenBabel software.










