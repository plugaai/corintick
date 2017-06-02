# corintick
Column-based datastore for historical timeseries data.
Corintick is designed mainly to store [pandas](http://pandas.pydata.org/) DataFrames that represent timeseries.

 
## Quickstart

### Instalation:

In order to use Corintick you need MongoDB. 
See installation instructions [here](https://docs.mongodb.com/manual/installation/).

Corintick itself can be installed with `pip`:

```bash
$ pip install git+https://github.com/plugaai/corintick
```


### Preparation

Once MongoDB is up and running, we need a simple configuration file:

```yaml
host:
  host: localhost
  port: 27017
log: '~/corintick_logs'
database:
  name: corintick
collections:
  - corintick
  - orderbook
```

In case your MongoDB setup requires authentication, add the following to your configuration file:

```yaml
auth:
  name: yourusername
  password: yourpassword
```

Initialize Corintick with your configuration file:
 
```python
from corintick import Corintick
corin = Corintick(config='config.yml')
```

Now we need a DataFrame to insert into Corintick. For demonstration purposes, we will get data from [Quandl](https://www.quandl.com/):  

```python
import quandl
df1 = quandl.get('TSE/7203')
```

Here, `df1` looks like this:

```text
              Open    High     Low   Close      Volume
Date
2012-08-23  3240.0  3270.0  3220.0  3260.0   4652200.0
2012-08-24  3225.0  3245.0  3210.0  3235.0   3659600.0
2012-08-27  3250.0  3280.0  3215.0  3220.0   3614600.0
2012-08-28  3235.0  3260.0  3150.0  3180.0   6759100.0
2012-08-29  3180.0  3195.0  3160.0  3175.0   2614800.0
2012-08-30  3180.0  3190.0  3160.0  3170.0   3291700.0
2012-08-31  3135.0  3155.0  3095.0  3095.0   5663800.0
...
```

### Writing

Inserting `df1` into Corintick is simple:

```python
corin.write('7203.T', df1, source='Quandl', country='Japan')
```

The first argument passed to `corintick.write` is an UID (universal identifier)
 and must be unique for each timeseries inserted in a given collection. 
The second argument is the dataframe to be inserted. 
The remaining keyword arguments are optional metadata tags that can be attached to the 
dataframe/document for querying.


### Reading

Reading from Corintick is also straightforward:

```python
df2 = corin.read('7203.T')
```

You can also specify `start` and `end` as ISO-8601 datetime string...

```python
df2 = corin.read('7203.T', start='2014-01-01', end='2014-12-31')
```

```
              Open    High     Low   Close      Volume
2014-01-06  6360.0  6400.0  6280.0  6300.0  12249300.0
2014-01-07  6270.0  6340.0  6260.0  6270.0   7891400.0
2014-01-08  6310.0  6320.0  6260.0  6300.0   7184100.0
2014-01-09  6310.0  6340.0  6260.0  6270.0   8653000.0
2014-01-10  6260.0  6310.0  6250.0  6290.0   7815900.0
...
2014-12-24  7645.0  7687.0  7639.0  7657.0  9287900.0
2014-12-25  7600.0  7655.0  7597.0  7611.0  5362700.0
2014-12-26  7629.0  7700.0  7615.0  7696.0  6069100.0
2014-12-29  7740.0  7746.0  7565.0  7662.0  9942800.0
2014-12-30  7652.0  7674.0  7558.0  7558.0  7821200.0
```


... and which columns you want retrived:

```python
df2 = corin.read('7203.T', columns=['Close', 'Volume'], start='2017-05-10')
```

```text
             Close      Volume
2017-05-10  6081.0   7823700.0
2017-05-11  6123.0  13511900.0
2017-05-12  6047.0   8216600.0
2017-05-15  6009.0   5925200.0
2017-05-16  6093.0   6449300.0
...
```


## Collections

Corintick can use multiple collections to better organize data. 
A Corintick collection is implemented as a MongoDB collection.
In each collection, only a single dataframe/document can exist for a given UID 
for a given time period.  

In case you need to store two different types of data for a same UID over an overlapping 
time frame (i.e. trade data and order book data for a given stock), you should separate 
the two different types of data into different collections.

The default collection is the first one listed in the configuration file and  
can be checked by the `Corintick.default_collection` property:

```python
>>> corin.collection
Collection(Database(MongoClient(host=['localhost:27017'], document_class=dict, tz_aware=False, connect=True), 'corintick'), 'corintick')
```

The default collection can be changed by assigning a string to `Corintick.default_collection`. 
This works even if the new collection is not listed in your configuration file:

```python
>>> corin.collection = 'another_collection'
```

Collections can also be changed on a method call basis, without changing 
the `collection` property:

```python
df = corin.read('7203.T', collection='orderbook')
```


## Corintick mechanics

During writing, Corintick does the following:

1) Takes the input DataFrame and splits into columns
2) Serializes/compresses each using the LZ4 compression algorithm
3) Generates a MongoDB document containing the binary blobs corresponding to each column and other metadata

During reading, the opposite takes places:
1) Documents are fetched
2) Data is decompressed and converted back to numpy arrays 
3) DataFrame is reconstructed and returned to the user

## Background

Corintick was inspired and by and aims to be a simplified
 version of Man AHL's [Arctic](https://github.com/manahl/arctic).

#### Differences from Arctic
Corintick has a single storage engine, which is column-based and not 
versioned, similar to Arctic's TickStore. However, differently from 
TickStore, it does support non-numerical `object` dtype columns by parsing 
them into MessagePack string objects

#### Naming
Corintick aimed from the beggining to be a column-based data storage.
"Corintick" is a blend of "Corinthan" (style of Roman columns) and "tick". 

## Benchmarks
**TODO**
- **vs InfluxDB**
- **vs vanila MongoDB**
- **vs MySQL**
- **vs KDB+ (32-bit)**

## Contributing

To contribute, fork the repository on GitHub, make your changes and 
submit a pull request :)
Pytrthree is not a mature project yet, so just simply raising issues is 
also greatly appreciated :)

### Extending Corintick

**TODO**

### Future goals
 - Implement a Python agnostic HTTP API so that clients don't need 
   the Python/Corintick installed locally