## Purpose
This project is a key-value pair database intended as an alternative to memcached.
It is an easy to use, single-file, header-only C++11-compatible project.

Memcached has a number of out-of-the-box limitations including lack of persistent storage,
type-inflexibility, no direct redundancy or failover capabilities, and poor scalability. 
This project resolves these issues by wrapping key-value pair operations around 
[ScyllaDB](https://www.scylladb.com), a Cassandra-compatible database written in C++.

ScyllaDB is a disk-backed data store with an in-RAM cache.
As such, it performs extremely well for this application.
In many cases, the entire data set can be stored in the database cache, resulting in 100% cache hits.
Under the rare circumstances where data is not in the ScyllaDB cache, the database itself is one of the highest 
performing disk-based databases that exists anywhere.
A single properly spec'ed server can serve as many as a million transactions per second even if it has to hit the disk.

ScyllaDB is a NoSQL eventually-consistent database, which is fine for many cache applications.
Memcached makes no guarantees that a key will return a value that was previously stored.
When a memcached node goes down that data is lost.
ScyllaDB, with built in redundancy, can almost always return something, even if it is an older version.
Inconsistencies can be repaired and resolved.

With memcached you were limited to the amount of RAM allocated on each memcached node.
There was no automatic way to scale ever higher as cache evictions increased cache misses.
ScyllaDB lets you easily scale up arbitrarily as demand increases.
With configurable levels of redundancy, you can decide how many copies of each piece of data you want on each database
node according to your own tolerance for failure.

ScyllaDB also supports tunable consistency. This project makes use of this by seeking high levels of
consistency, but allowing dynamically for lower levels of consistency in exchange for higher availability.
It is also possible to tune this to require full quorum-level consistency that mirrors memcached's all or
nothing availability.

Using a fully typed database, we can do more than just "string => string" key-value pairs.
The project supports integers, floating-points, strings, bytes (blobs), and uuids.
C++ templates make it easy to integrate different combinations.

There is one important caveat.
While memcached allows support for a fixed memory profile, the ScyllaDB data store does not. 
Memcached keeps performance guarantees by evicting cached data, while ScyllaDB retrieves it from disk.
The extreme performance of ScyllaDB makes this relatively painless for most applications.
To maintain strictly absolute RAM-based access performance, enough memory is required to store the full data set.
Alternatively, precision use of TTL records for automatic deletion of old cache records is supported.

## Key Features
* Single header-only implementation makes it easy to drop into C++ projects.
* A optional backlog queues data in the event that the database is temporarily down.
* Adaptive fault tolerance, consistency, and availability.
* Supports a variety of native C++ data types in the keys and values.
 * 8-, 16-, 32-, and 64-bit signed integers
 * 32-bit unsigned integers
 * single- and double-precision floating point numbers
 * boolean
 * strings
 * binary byte arrays
 * UUID
* Simple API: Only a single store and a single retrieve function are needed. There is no need to write database queries.
* RAM-like performance for most applications.
* There is no need to batch read or write requests for performance.
* There is no special client configuration required for redundancy, scalability, or multi-thread performance.

## Backlog
This project incorporates a backlog to queue changes locally for times when the remote server is unavailable.
If a store request fails for any reason, it can be cached in the backlog to be committed later.
In memcached this data would be lost or require the application to wait until the server returned.
This is well-suited to asynchronous producer/consumer applications where the producer doesn't want to wait around
for the server to become available and it is okay if the consumer gets the data eventually.
It's another layer of redundancy on top of an already solid database backend.

Backlog use is optional and its use can be selected individually for each store request.

In order to maximize performance of the store functionality by using (nearly) a lockless design,
a few design trade-offs were made:
1. If the backlog receives two entries with the same key, it will not remove the older one.
   They will be applied in chronological order, however, so eventually the newer one will replace the older.
1. Older backlog entries may be processed after newer successful non-backlogged store requests.
   The default backlog mode should not be used if losing backlogged data is more acceptable than the chance of 
   having overwritten newer data.
   Store requests can be configured to only use the backlog.
   While this reduces maximum performance, it eliminates any data consistency issues.
1. Retrievals do not check the backlog.
1. If the client cannot connect to a server and never has, failed ValuStor::store() calls will use the backlog queue.
   Even if the server becomes accessible, the backlog thread will not begin to process automatically.
   The backlog processing will only being once the first ValuStor::store() call is successful.

## Configuration
Configuration can use either a configuration file or setting the same configuration at runtime. See the API documentation. The only requirement is to set the following fields:
```
  table = <database>.<table>
  key_field = <key field>
  value_field = <value field>
  username = <username>
  password = <password>
  ip_addresses = <ip_address_1>,<ip_address_2>,<ip_address_3>
```

The schema of a scylla table should be setup as follows:
```sql
  CREATE TABLE <database>.<table> (
    <key_field> bigint PRIMARY KEY,
    <value_field> text
  ) WITH compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'};
```

The following Cassandra data types are supported in the CREATE TABLE:
* tinyint (int8_t)
* smallint (int16_t)
* int (int32_t)
* bigint (uint32_t and int64_t)
* float (float)
* double (double)
* boolean (cass_bool_t)
* varchar, text, and ascii (std::string)
* blob (cass_byte_t*)
* uuid (CassUuid)

## API
ValuStor is implemented as a template class using two constructors. See the usage documentation.
```C++
  template<typename Key_T, typename Val_T> class ValuStor

  ValuStor(std::string config_file)
  ValuStor(std::map<std::string, std::string> configuration_kvp)
```

The public API is very simple:
```C++
  ValuStor::Result store(Key_T key, Val_T value, uint32_t seconds_ttl, InsertMode_t insert_mode)
  ValuStor::Result retrieve(Key_T key)
```

The optional seconds TTL is the number of seconds before the stored value expires in the database.

The optional insert modes are `ValuStor::DISALLOW_BACKLOG`, `ValuStor::ALLOW_BACKLOG`, and `ValuStor::USE_ONLY_BACKLOG`.

The ValuStor::Result has the following data members:
```C++
  ErrorCode_t error_code
  std::string result_message
  std::string data
```

The ValuStor::ErrorCode_t is one of the following:
```C++
  ValuStor::VALUE_ERROR
  ValuStor::UNKNOWN_ERROR
  ValuStor::BIND_ERROR
  ValuStor::QUERY_ERROR
  ValuStor::CONSISTENCY_ERROR
  ValuStor::PREPARED_SELECT_FAILED
  ValuStor::PREPARED_INSERT_FAILED
  ValuStor::SESSION_FAILED
  ValuStor::SUCCESS
  ValuStor::NOT_FOUND
```

## Usage

Code:
```C++
  ValuStor<long, std::string> valuestore("example.conf");
  auto store_result = valuestore.store(1234, "value");
  if(store_result){
    auto retrieve_result = valuestore.retrieve(1234);
    if(retrieve_result){
      std::cout << 1234 << " => " << result.data << std::endl;
    }
  }
```

or

```C++
  ValuStor<long, std::string> valuestore({
        {"table", "cache.values"},
        {"key_field", "key_field"},
        {"value_field", "value_field"},
        {"username", "username"},
        {"password", "password"},
        {"ip_addresses", "127.0.0.1"}
  });
  auto store_result = valuestore.store(1234, "value");
  if(store_result){
    auto retrieve_result = valuestore.retrieve(1234);
    if(retrieve_result){
      std::cout << 1234 << " => " << result.data << std::endl;
    }
  }
```

Output:
```
  1234 => value
```


## Dependencies
The Cassandra C/C++ driver is required. See https://github.com/datastax/cpp-driver/releases
This project has only been tested with version 2.7.1 and 2.8.1, but in principle it should work with other versions.
If using g++, it must be linked with -L/path/to/libcassandra.so/ -lcassandra. Example installation:
```sh
  wget https://github.com/datastax/cpp-driver/archive/2.8.1.tar.gz
  tar xvfz 2.8.1.tar.gz
  cd cpp-driver-2.8.1
  mkdir build
  cd build
  cmake ..
  make
  make install
```

An installation of either Cassandra or ScyllaDB is required. The latter is strongly
recommended for this application, as the former has much worse performance. ScyllaDB is incredibly [easy
to setup](http://docs.scylladb.com/getting-started/). This project has been tested with ScyllaDB v.2.x.

## Thread Safety
The cassandra driver fully supports multi-threaded access.
This project is completely thread safe.
It is lockless, except for using the backlog. Locks are only held if needed and for as short a time as possible.
Multi-threaded stores are generally higher performing than single-threaded stores if it can use multiple CPU cores.

NOTE: The multi-threaded performance of the cassandra driver is higher performing than the backlog thread.
      The backlog should only be used to increase data availability, not to increase performance.
      It is single-thread and uses locking, so it will always have worse performance.

## Atomicity
All write operations are performed atomically, but depending on the consistency level unexpected results may occur.
If the order is strictly important, all reads and writes must be performed at QUORUM consistency or higher.
There is no way to read-and-modify (including prepending/appending) data atomically.

## License
MIT License

Copyright (c) 2017-2018 Sensaphone

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

Except as contained in this notice, the name(s) of the above copyright holders
shall not be used in advertising or otherwise to promote the sale, use or
other dealings in this Software without prior written authorization.
