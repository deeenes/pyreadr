# py<span style="color:blue">r</span>ead<span style="color:blue">r</span>

A python package to read and write R RData and Rds files into/from 
pandas dataframes. It does not need to have R or other external
dependencies installed.
<br> 

This module is based on the [librdata](https://github.com/WizardMac/librdata) C library by 
[Evan Miller](https://www.evanmiller.org/) and a modified version of the cython wrapper around 
librdata
[jamovi-readstat](https://github.com/jamovi/jamovi-readstat)
by the [Jamovi](https://www.jamovi.org/) team.

Detailed documentation on all available methods is in the 
[Module documentation](https://ofajardo.github.io/pyreadr/)

## Table of Contents

- [Dependencies](#dependencies)
- [Installation](#installation)
  * [Using pip](#using-pip)
  * [From the latest sources](#from-the-latest-sources)
- [Usage](#usage)
  * [Basic Usage: reading files](#basic-usage--reading-files)
  * [Basic Usage: writing files](#basic-usage--writing-files)
  * [Reading selected objects](#reading-selected-objects)
  * [List objects and column names](#list-objects-and-column-names)
  * [Reading timestamps and timezones](#reading-timestamps-and-timezones)
  * [What objects can be read](#what-objects-can-be-read)
  * [More on writing files](#more-on-writing-files)
- [Known limitations](#known-limitations)
- [Change Log](#change-log)
- [People](#people)

## Dependencies

The module depends on pandas, which you normally have installed if you got Anaconda (highly recommended.) If creating
a new conda or virtual environment or if you don't have it in your base installation, you will have to install it 
manually before using pyreadr. Pandas is not selected as a dependency in the pip package, as that would install 
pandas with pip and many people would prefer installing it with conda.

In order to compile from source, you will need a C compiler (see installation) and cython 
(version >= 0.28).

librdata also depends on zlib; it was reported not to be installed on Lubuntu. If you face this problem intalling the 
library solves it.

## Installation

### Using pip

Probably the easiest way: from your conda, virtualenv or just base installation do:

```
pip install pyreadr
```

If you are running on a machine without admin rights, and you want to install against your base installation you can do:

```
pip install pyreadr --user
```

We offer pre-compiled wheels for python 3.5, 3.6 and 3.7 for Windows,
linux and macOs.

### From the latest sources

Download or clone the repo, open a command window and type:

```
python3 setup.py install
```

If you don't have admin privileges to the machine do:

```
python3 setup.py install --user
```

You can also install from the github repo directly (without cloning). Use the flag --user if necessary.

```
pip install git+https://github.com/ofajardo/pyreadr.git
```

You need a working C compiler and cython.

## Usage

### Basic Usage: reading files

Pass the path to a RData or Rds file to the function read_r. It will return a dictionary 
with object names as keys and pandas data frames as values.

For example, in order to read a RData file:

```python
import pyreadr

result = pyreadr.read_r('test_data/basic/two.RData')

# done! let's see what we got
print(result.keys()) # let's check what objects we got
df1 = result["df1"] # extract the pandas data frame for object df1
```

reading a Rds file is equally simple. Rds files have one single object, 
which you can access with the key None:

```python
import pyreadr

result = pyreadr.read_r('test_data/basic/one.Rds')

# done! let's see what we got
print(result.keys()) # let's check what objects we got: there is only None
df1 = result[None] # extract the pandas data frame for the only object available
```

Here there is a relation of all functions available. 
You can also check the [Module documentation](https://ofajardo.github.io/pyreadr/).

| Function in this package | Purpose |
| ------------------- | ----------- |
| read_r        | reads RData and Rds files |
| list_objects  | list objects and column names contained in RData or Rds file |
| write_rdata   | writes RData files |
| write_rds     | writes Rds files   |

### Basic Usage: writing files

Pyreadr allows you to write one single pandas data frame into a single R dataframe
and store it into a RData or Rds file. Other python or R object types 
are not supported. Writing more than one object is not supported.

The operation is simple. For RData files:

```python
import pyreadr
import pandas as pd

# prepare a pandas dataframe
df = pd.DataFrame([["a",1],["b",2]], columns=["A", "B"])

# let's write into RData
# df_name is the name for the dataframe in R, by default dataset
pyreadr.write_rdata("test.RData", df, df_name="dataset")

# now let's write a Rds
pyreadr.write_rds("test.Rds", df)

# done!

```

now you can check the result in R:

```r
load("test.RData")
print(dataset)

dataset2 <- readRDS("test.Rds")
print(dataset2)

```


### Reading selected objects

You can use the argument use_objects of the function read_r to specify which objects
should be read. 

```python
import pyreadr

result = pyreadr.read_r('test_data/basic/two.RData', use_objects=["df1"])

# done! let's see what we got
print(result.keys()) # let's check what objects we got, now only df1 is listed
df1 = result["df1"] # extract the pandas data frame for object df1
```

### List objects and column names

The function list_objects gives a dictionary with object names contained in the
RData or Rds file as keys and a list of column names as values.
It is not always possible to retrieve column names without reading the whole file
in those cases you would get None instead of a column name.

```python

import pyreadr

object_list = pyreadr.list_objects('test_data/basic/two.RData')

# done! let's see what we got
print(object_list) # let's check what objects we got and what columns those have

```

### Reading timestamps and timezones

R datetime objects (POSIXct and POSIXlt) are internally stored as UTC timestamps, and may have additional timezone
information if the user set it explicitly. librdata cannot retrieve that timezone information. If no timezone information
was set by the user R uses the local timezone for display. 

As timezone information is not available from librdata, pyreadr display UTC time by default, which will not match the
display in R. You can set explicitly some timezone (your local timezone for example) with the argument timezone for the
function read_r

```python
import pyreadr

result = pyreadr.read_r('test_data/basic/two.RData', timezone='CET')

```

if you would like to just use your local timezone as R does, you can 
get it with tzone (you need to install it first with pip) and pass the 
information to read_r:

```python

import tzlocal
import pyreadr

my_timezone = tzlocal.get_localzone().zone
result = pyreadr.read_r('test_data/basic/two.RData', timezone=my_timezone)

```

If you have control over the data in R, a good option is to transform
the POSIX object to character, then transform it to a datetime in python.

### What objects can be read

Data frames composed of character, numeric (double), integer, timestamp (POSIXct 
and POSIXlt), logical atomic vectors. Factors are also supported.

Tibbles are also supported.

Atomic vectors as described before can also be directly read, but as librdata
does not give the information of the type of object it parsed everything
is translated to a pandas data frame.

### More on writing files

For converting python/numpy types to R types the following rules are
followed:

| Python Type         | R Type    |
| ------------------- | --------- |
| np.int32 or lower   | integer   |
| np.int64, np.float  | numeric   |
| str                 | character |
| bool                | logical   |
| datetime, date      | character |
| category            | depends on the original dtype |
| any other object    | character |
| empty string        | NA        |
| column all missing  | logical   |

A few interesting points:

* datetime and date objects are translated to character to avoid problems
with timezones. These characters can be easily translated back to POSIXct/lt in R
using as.POSIXct/lt. The format of the datetimes/dates is prepared for this
but can be controlled with the arguments dateformat and datetimeformat 
for write_rdata and write_rds. Those arguments take python standard
formatting strings.

* Pandas categories are NOT translated to R factors. Instead the original
data type of the category is preserved and transformed according to the
rules. This is because R factors are integers and levels are always
strings, in pandas factors can be any type and leves any type as well, therefore
it is not always adecquate to coerce everything to the integer/character system.
In the other hand, pandas category level information is lost in the process.

* Any other object is transformed to a character using the str representation
of the object.

* R integers are 32 bit. Therefore python 64 bit integer have to be 
promoted to numeric in order to fit.

* A pandas column containing only missing values is transformed to logical,
following R's behavior.

* librdata represents character missing values as an empty string. Therefore
any empty string in pandas will be transformed into NA in R.

## Known limitations

* As explained before, although atomic vectors can also be directly read, as librdata
does not give the information of the type of object it parsed everything
is translated to a pandas data frame.

* When reading missing values in character vectors librdata gives empty strings. 
Those cannot be distinguished from a valid empty string, therefore pyreadr
gives back just an empty string and not a np.nan value. The inverse problem
also exists: when writing an empty string will be transformed to a NA missing
value.

* POSIXct and POSIXlt objects in R are stored internally as UTC timestamps and may have
in addition time zone information. librdata does not return time zone information and
thefore the display of the tiemstamps in R and in pandas may differ.

* Matrices and arrays are read, but librdata does not return information about
the dimensions, therefore those cannot be arranged properly multidimensional
numpy arrays. They are translated to pandas data frames with one single column.

* Lists are not read.

* Objects that depend on non base R packages (Bioconductor for example) cannot be read.
The error code in this case is a bit obscure:

```python
ValueError: Unable to read from file
```

* Data frames with special values like arrays, matrices and other data frames
are not supported

* Writing is supported only for a single pandas data frame to a single
R data frame. Other data types are not supported. Multiple data frames
for rdata files are not supported.

## Change Log

A log with the changes for each version can be found [here](https://github.com/ofajardo/pyreadr/blob/write_support/change_log.md)
## People

Otto Fajardo - author, maintainer

[Jonathon Love](https://jona.thon.love/) - contributor (original cython wrapper from jamovi-readstat)
