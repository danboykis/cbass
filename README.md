# cbass

* Databases are for storing and finding data 
* HBase is great at that
* Clojure is great at "simple"

--
[![Clojars Project](http://clojars.org/cbass/latest-version.svg)](http://clojars.org/cbass)

## Show me

```clojure
(require '[cbass :refer [new-connection store find-by scan delete]])
```

## Connecting to HBase

```clojure
(def conf {"hbase.zookeeper.quorum" "127.0.0.1:2181" "zookeeper.session.timeout" 30000})
(def conn (new-connection conf))
```

## Storing data

```clojure 
;; args:      conn, table, row key, family, data

user=> (store conn "galaxy:planet" "earth" "galaxy" {:inhabited? true 
                                                     :population 7125000000 
                                                     :age "4.543 billion years"})
```

Depending on a key strategy/structure sometimes it makes sense to only store row-keys / families witout values:

```clojure
user=> (store conn "galaxy:planet" "pluto" "galaxy")
```

## Finding it

There are two primary ways data is found in HBase:

* by the row key: [HBase Get](http://hbase.apache.org/book.html#_get)
* by "anything": [HBase Scan](http://hbase.apache.org/book.html#scan)

### Finding by the row key

```clojure
;; args:        conn, table, row key, [family, columns]

user=> (find-by conn "galaxy:planet" "earth")
{:age "4.543 billion years", :inhabited? true, :population 7125000000}

user=> (find-by conn "galaxy:planet" "earth" "galaxy")
{:age "4.543 billion years", :inhabited? true, :population 7125000000}

user=> (find-by conn "galaxy:planet" "earth" "galaxy" #{:age :population})
{:age "4.543 billion years", :population 7125000000}
```

### Finding by "anything"

HBase calls them scanners, hence the `scan` function name.

Let's first look directly at HBase (shell) to see the data we are going to scan over:

```clojure
hbase(main):002:0> scan 'galaxy:planet'
ROW         COLUMN+CELL
 earth      column=galaxy:age, timestamp=1440880021543, value=NPY\x00i\x134.543 billion years
 earth      column=galaxy:inhabited?, timestamp=1440880021543, value=NPY\x00\x04\x01
 earth      column=galaxy:population, timestamp=1440880021543, value=NPY\x00+\x00\x00\x00\x01\xA8\xAE\xDF@
 mars       column=galaxy:age, timestamp=1440880028315, value=NPY\x00i\x134.503 billion years
 mars       column=galaxy:inhabited?, timestamp=1440880028315, value=NPY\x00\x04\x01
 mars       column=galaxy:population, timestamp=1440880028315, value=NPY\x00d\x03
 neptune    column=galaxy:age, timestamp=1440880036629, value=NPY\x00i\x134.503 billion years
 neptune    column=galaxy:inhabited?, timestamp=1440880036629, value=NPY\x00j\x07unknown
3 row(s) in 0.0230 seconds
```

HBase scanning is pretty flexible: by row key from/to prefixes, by time ranges, by families/columns, etc..

Here are some examples:

#### Scanning the whole table

```clojure
;; args:        conn, table, {:row-key-fn, :family, :columns, :from, :to, :time-range {:from-ms :to-ms}}

user=> (scan conn "galaxy:planet")

{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

#### Scanning with a row key function

By default cbass will assume row keys are strings, but in practice keys are prefixed and/or hashed.
Hence to read a row key from HBase, a custom row key function may come handy:

```clojure
;; args:        conn, table, {:row-key-fn, :family, :columns, :from, :to, :time-range {:from-ms :to-ms}}

user=> (scan conn "galaxy:planet" :row-key-fn #(keyword (String. %)))

{:earth
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 :mars {:age "4.503 billion years", :inhabited? true, :population 3},
 :neptune {:age "4.503 billion years", :inhabited? :unknown}}
```

#### Scanning families and columns

by family

```clojure
user=> (scan conn "galaxy:planet" :family "galaxy")

{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

specifying columns (qualifiers)

```clojure
user=> (scan conn "galaxy:planet" :family "galaxy" 
                                  :columns #{:age :inhabited?})

{"earth" {:age "4.543 billion years", :inhabited? true},
 "mars" {:age "4.503 billion years", :inhabited? true},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

#### Scanning by row key prefix

Data can be scanned by a row key prefix using `:from` and/or `:to` keys:

```clojure
user=> (scan conn "galaxy:planet" :from "ma")

{"mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

`:to` is exclusive:

```clojure
user=> (scan conn "galaxy:planet" :from "ea" 
                                  :to "ma")

{"earth" {:age "4.543 billion years", :inhabited? true, :population 7125000000}}
```

notice, no Neptune:

```clojure
user=> (scan conn "galaxy:planet" :to "nep")

{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3}}
```

#### Scanning by time range

If you look at the data from HBase shell (above), you'll see that every row has a timestamp associated with it.

These timestamps can be used to scan data within a certain time range:

```clojure
user=> (scan conn "galaxy:planet" :time-range {:from-ms 1440880021544 
                                               :to-ms 1440880036630})

{"mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

in case `:from-ms` is missing, it defauts to `0`:

```clojure
user=> (scan conn "galaxy:planet" :time-range {:to-ms 1440880036629})

{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3}}
```

same analogy with `:to-ms`, if it is mising, it defaults to `Long/MAX_VALUE`:

```clojure
user=> (scan conn "galaxy:planet" :time-range {:from-ms 1440880036629})

{"neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

#### Scanning in reverse

Here is a regular table scan with all the defaults:

```clojure
user=> (scan conn "galaxy:planet")

{"earth" {:age "4.543 billion years", :inhabited? true, :population 7125000000},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown}}
```

many times it makes sense to scan table in reverse order 
to have access to the latest updates first without scanning the whole search space:

```clojure
user=> (scan conn "galaxy:planet" :reverse? true)

{"neptune" {:age "4.503 billion years", :inhabited? :unknown},
 "mars" {:age "4.503 billion years", :inhabited? true, :population 3},
 "earth" {:age "4.543 billion years", :inhabited? true, :population 7125000000}}
```

#### Scanning by "anything"

Of course _all_ of the above can be combined together, and that's the beauty or scanners:

```clojure
user=> (scan conn "galaxy:planet" :family "galaxy" 
                                  :columns #{:age}
                                  :from "ma" 
                                  :to "z" 
                                  :time-range {:to-ms 1440880036630})

{"mars" {:age "4.503 billion years"},
 "neptune" {:age "4.503 billion years"}}
```

There are lots of other ways to "scan the cat", but for now here are several.

## Deleting it

#### Deleting specific columns

```clojure
;; args:       conn, table, row key, [family, columns]

user=> (delete conn "galaxy:planet" "earth" "galaxy" #{:age :population})

user=> (find-by conn "galaxy:planet" "earth")
{:inhabited true}
```

#### Deleting a column family

```clojure
;; args:       conn, table, row key, [family, columns]

user=> (delete conn "galaxy:planet" "earth" "galaxy")

user=> (find-by conn "galaxy:planet" "earth")
nil
```

#### Deleting a whole row

```clojure
;; args:       conn, table, row key, [family, columns]

user=> (delete conn "galaxy:planet" "mars")

user=> (find-by conn "galaxy:planet" "mars")
nil
```

### Deleting by anything

There is often a case where rows need to be deleted by a filter, that is similar to the one used in [scan](https://github.com/tolitius/cbass#scanning-by-anything) (i.e. by row key prefix, time range, etc.)
HBase does not really help there besides providing a [BulkDeleteEndpoint](http://archive.cloudera.com/cdh5/cdh/5/hbase/apidocs/org/apache/hadoop/hbase/coprocessor/example/BulkDeleteEndpoint.html) coprocessor.

This is not ideal as it delegates work to HBase "stored procedures" (effectively that is what coprocessors are).
It really pays off during massive data manipulation since it does happen _directly_ on the server,
but in simpler cases, which are many, coprocessors are less than ideal.

**cbass** achives "deleting by anything" by a trivial flow: "scan + multi delete" packed in a "delete-by" function
which preserves the "scan"'s syntax:

```clojure
user=> (scan conn "galaxy:planet")
{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown},
 "pluto" {},
 "saturday" {:age "24 hours", :inhabited? :sometimes},
 "saturn" {:age "4.503 billion years", :inhabited? :unknown}}

user=> (delete-by conn "galaxy:planet" :from "sat" :to "saz")
;; deleting [saturday saturn], since they both match the 'from/to' criteria
```

look ma, no saturn, no saturday:

```clojure
user=> (scan conn "galaxy:planet")
{"earth"
 {:age "4.543 billion years",
  :inhabited? true,
  :population 7125000000},
 "neptune" {:age "4.503 billion years", :inhabited? :unknown},
 "pluto" {}}
```

and of course any other criteria that is available in "scan" is available in "delete-by".

### Delete row key function

Most of the time HBase keys are prefixed (salted with a prefix).
This is done to avoid ["RegionServer hotspotting"](http://hbase.apache.org/book.html#rowkey.design).

"delete-by" internally does a "scan" and returns keys that matched. Hence in order to delete these keys
they have to be "re-salt-ed" according to the custom key design.

**cbass** addresses this by taking an optional `delete-key-fn`, which allows to "put some salt back" on those keys.

Here is a real world example:

```clojure
;; HBase data

user=> (scan conn "table:name")
{"���|8276345793754387439|transfer" {...},
 "���|8276345793754387439|match" {...},
 "���|8276345793754387439|trade" {...},
 "�d\k^|28768787578329|transfer" {...},
 "�d\k^|28768787578329|match" {...},
 "�d\k^|28768787578329|trade" {...}}
```

a couple observations about the key:

* it is prefixed with salt
* it is piped delimited 

In order to delete, say, all keys that start with `8276345793754387439`,
besides providing `:from` and `:to`, we would need to provide a `:row-key-fn` 
that would _de_ salt and split, and then a `delete-key-fn` that can reassemble it back:

```clojure
(delete-by conn progress :row-key-fn (comp split-key without-salt)
                         :delete-key-fn (fn [[x p]] (with-salt x p))
                         :from (-> "8276345793754387439" salt-pipe)
                         :to   (-> "8276345793754387439" salt-pipe+))))
```

`*salt`, `*split` and `*pipe` functions are not from **cbass**, 
they are here to illustrate the point of how "delete-by" can be used to take on the real world.

```clojure
;; HBase data after the "delete-by"

user=> (scan conn "table:name")
{"�d\k^|28768787578329|transfer" {...},
 "�d\k^|28768787578329|match" {...},
 "�d\k^|28768787578329|trade" {...}}
```

## License

Copyright © 2015 tolitius

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
