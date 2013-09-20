{:title "AuthorDB: Parsing and Importing OpenLibrary.org's Author data into Datomic"
 :slug "authordb-datomic-tutorial"}

Pretend we're making a GoodReads.com clone where users will be able to manage their personal library of books, view all the books an author has published, and all the authors that have contributed to a book.

- Our app will need to be able to look up book and author data from its own database.
- We've chosen Datomic as our database for our Clojure application. 
- We'll get our data from OpenLibrary's CSV data dumps and import into a local Datomic database.

Since we're new to Datomic, we'll only focus on Author data for now.

```shell
$ lein new authordb
$ cd authordb
$ git init
```

## Intended audience

I had never used Datomic until a couple days ago. 

This isn't an organized, focused tutorial. Instead, it's more of a brain dump chronology of what I went through to get 1.45gb of OpenLibrary's author data parsed and imported into my Datomic database so that I could query it.

If you're interested in Datomic or even Clojure, I hope this whirlwind tour has something for you.

## What we're building

The author data set we're importing includes over 6.8 million authors.

For each author, we'll be storing:

- name: The author's name. 
- lastModified: When OpenLibrary's record was last modified
- olid: OpenLibrary's unique id
- revision: A number that OpenLibrary increments by one each time a record is modified.

lastModified, olid, and revision are going to be useful for when we need to sync up our database to OpenLibrary's monthly data dump.

lastModified is a bit extraneous, but I included it as an example for parsing a datetime-string into the java.util.Date that Datomic expects.

## Dependencies

- **data.json** for parsing the JSON data embedded in the authors data dump.
- **datomic-free** is all we need for Datomic.
- **clj-time** is for parsing arbitrary datetime-strings into  `java.util.Date` that Datomic expects.

```clojure
;; project.clj

:dependencies [...
               [org.clojure/data.json "0.2.3"]
               [com.datomic/datomic-free "0.8.4020.24"]
               [clj-time "0.6.0"]]
```

Let's create a file to experiment in:

    authordb/src/authordb/sandbox.clj
    
For convenience, here's the full `ns` declaration I'm using so that I don't need to fully-qualify every namespace in my code examples.

```clojure
(ns authordb.sandbox
  (:use [clojure.repl])
  (:require [clojure.java.io :as io]
            [clojure.string :as str]
            [datomic.api :as d]
            [clojure.data.json :as json]
            [clj-time.format :as time
             :refer [formatter formatters parse]]
            [clojure.pprint :refer [pprint]]
            [clojure.edn :as edn])
  (:import [java.util Date]
           [datomic Util]
           [java.io PushbackReader]))
```

Note that `clojure.repl` gives us functions like `doc` and `source` which are good for remembering what functions do, what they expect, or how they're implemented. 

Try out `(doc println)` and `(source println)` if you're new to the `clojure.repl` namespace. Their output is printed to your repl.

## Downloading the dump of author data

(I believe you have to register at Open Library to visit the Developer pages)

Open Library has a page with various bulk-data offerings:

- [OpenLibrary.org: Open Library Date Dumps](http://openlibrary.org/developers/dumps)

There isn't much documentation to be found, but basically Open Library has a concept of:

- Authors
- Works
- Editions

A `Work` is the abstract representation of a book. It at least has a title and a set of authors associated with it.

An `Author` can belong to many books.

An `Edition` is a concrete publication of a Work. It can contain things like publication dates and it can represent hardcover books, paperbacks, and ebooks. I didn't even crack open the edition data dump (ran out of hard-drive space) but I believe Editions can even have their own Authors.

But our app is only concerned with a Work-has-many-Authors representation. We don't care about Editions. And this post only demonstrates Author data, anyways.

Fortunately, Open Library splits its dumps into those three concepts, so we can download the author data a la carte. 

- [ol_dump_authors_latest.txt.gz](http://openlibrary.org/data/ol_dump_authors_latest.txt.gz) [2.17gb]

Unzip it into `authordb/resources/authors.txt`.

In hindsight, it was pretty painful to iterate on trial and error on a 2.17gb text file with 6.8+ million rows.

You might want to create a copy with only 100 rows until you get it working. However, working with a massive data set quickly teaches you how to embrace lazy evaluation. No more `slurp` for you!

## Making sense of authors.txt

Take a gander at the data.

```shell
cat authors.txt | less
```

Open Library's Bulk Data page explains that columns are delimited by tabs and the column map to these representations:

- **type** - type of record (/type/edition, /type/work etc.)
- **key** - unique key of the record. (/books/OL1M etc.)
- **revision** - revision number of the record
- **last_modified** - last modified timestamp
- **JSON** - the complete record in JSON format

Let's grab the first line of the CSV to play with using `line-seq` (which returns a lazy sequence of lines):

```clojure
(def line
  (first (line-seq (io/reader (io/resource "authors.txt")))))
  
line
;=> "/type/author\t/authors/OL1000057A\t2\t2008-08-20T17:57:09.66187\t
 {\"name\": \"Kha\\u0304lid Muh\\u0323ammad \\u02bbAli\\u0304 
 al-H\\u0323a\\u0304jj\", \"personal_name\": \"Kha\\u0304lid 
 Muh\\u0323ammad \\u02bbAli\\u0304 al-H\\u0323a\\u0304jj\",
 \"last_modified\": {\"type\": \"/type/datetime\", \"value\":
 \"2008-08-20T17:57:09.66187\"}, \"key\": \"/authors/OL1000057A\",
 \"type\": {\"key\": \"/type/author\"}, \"revision\": 2}"
```

You can see the tabs represented as `\t`.

```clojure
(str/split line #"\t")

;=> ["/type/author"
     "/authors/OL1000057A"
     "2"
     "2008-08-20T17:57:09.66187"
     "{...}"]
```

If you look at the JSON data (last element after the split), it actually has everything we need:

- **name**
- **personal_name**: I couldn't find a good usage example of it in the entire set. It's usually either nil or equivalent to the name. So we won't worry about it.
- **last_modified**: a datetime string
- **key**: A key looks like `/authors/OL1000057A`. The `OL...` bit is the unique id of the author entity in Open Library's system. We extract the `OL...` portion into the `olid` that we'll store in our database.
- **revision**

So first let's write a function that lets us: `(map parse-line lines)`.

It will parse the JSON out of each string:

```clojure
{:name "Khālid Muḥammad ʻAlī al-Ḥājj",
 :personal_name "Khālid Muḥammad ʻAlī al-Ḥājj",
 :last_modified
 {:type "/type/datetime", :value "2008-08-20T17:57:09.66187"},
 :key "/authors/OL1000057A",
 :type {:key "/type/author"},
 :revision 2}
```

And then return only the stuff we care about:

```clojure
{:name "Khālid Muḥammad ʻAlī al-Ḥājj",
 :last_modified #inst "2008-02-02 17:57:09"
 :olid "OL1000057A"
 :revision 2}
```

Our `parse-line` function looks like this:

```clojure
(defn parse-line [line]
  (let [[_ _ _ _ json-string] (str/split line #"\t")
        json (json/read-str json-string :key-fn keyword)
        {:keys [name last_modified key revision]} json]
   {:name name
    :last_modified (parse-date (:value last_modified))
    :olid (parse-olid key)
    :revision revision}))
```

We'll also need to define two helper functions:

- `parse-date`: Takes a string like "2008-08-20T17:57:09.66187" and returns a `java.util.Date` object (Datomic's Instant datatype expect it).
- `parse-olid`: Takes a string like "/authors/OL1000A" and returns just the unique ID parse "OL1000A".

`parse-olid` is simple:

```clojure
(defn parse-olid
  "/authors/OL1000624A -> OL10000624A"
  [s]
  (let [[_ olid] (re-find #"\/(OL\w+)" s)]
    olid))
```

To write our `parse-date` function, we need to know the format of the strings that the data dump represents its datetimes with.

I found it only varied by two formats:

- "2008-08-20T17:57:09.66187"
- "2008-08-20T17:57:09"

After fiddling with the clj-time library, I found that:

- Its `:date-hour-minute-second-fraction` formatter can parse the first date-string above.
- Its `:date-hour-minute-second` formatter can parse the second.

Here's what we're going for:

```clojure
(assert (= (parse-date "2008-08-20T17:57:09.66187")
           #inst "2008-08-20T17:57:09.661-00:00"))

(assert (= (parse-date "2008-08-20T17:57:09")
           #inst "2008-08-20T17:57:09.000-00:00"))

(assert (= (parse-date "2008")
           (throws? IllegalArgumentException]))
```

I scribbled out this implementation:

```clojure
(defn parse-date [s]
  (let [;;  2008-08-20T17:57:09.66187
        fmt1 (time/formatters :date-hour-minute-second-fraction)
        ;;  2008-08-20T17:57:09
        fmt2 (time/formatters :date-hour-minute-second)]
    (try
      (.toDate (time/parse fmt1 s))
      (catch IllegalArgumentExeption e
        (if (re-find #"Invalid format" (.getMessage e))
          (time/parse fmt2 s)
          (throw e))))))
```

If the first parse attempt fails, it catches the invalid format error and tries again with with the other format.

If they both fail, then the exception is let through.

I didn't look very hard for a superior way to do this and it doesn't scale very well if we had to handle many formats, but it works for now.

You can test our `parse-line` function by mapping it over the first 5 lines of authors.txt:

```clojure
(let [lines (take 5 (line-seq (io/reader (io/resource "authors.txt"))))]
  (map parse-line lines))
```

You should get back five maps that look about right.

Each key in the map represents a "column" (attribute) in our Datomic schema which we'll make next.

## Installing the Datomic transactor

While the Datomic project dependency is all we need if we just want an in-memory database, we'll need to download Datomic externally so that we can launch the free transactor which supports file persistence.

In other words, running the external transactor will save the state of our database to an H2 database file.

- Download the latest .zip: [http://downloads.datomic.com/free.html](http://downloads.datomic.com/free.html)
- `cd` into the folder
- Start the transactor with:

```shell
bin/transactor config/samples/free-transactor-template.properties
```

It should print the URI that you can reach it at: `datomic:free://localhost:4334/<DB-NAME>`.

Since we're going to be importing lots of data, I recommend reading the comments in that free-transactor-template.properties file as well as getting an understanding of [http://docs.datomic.com/capacity.html](http://docs.datomic.com/capacity.html) to tweak the transactor's settings for an import job that makes sense on your machine.

Don't forget that you can increase the transactor's heap size max with:

```shell
bin/transactor -Xmx4g config/samples/free-transactor-template.properties
```

You can also set the max heap size of your app (peer) with this key in project.clj:

```clojure
  :jvm-opts ["-Xmx4g"] 
```

The import was painful on a 2013 Macbook Air with 8gb RAM and I just set 4gb of heap size for both since I don't really know what I'm doing. I also went with the property file's example import specs.

So, get that running and then go back to our `sandbox.clj` file.

## A bare-bones Datomic database

Let's create our database.

Our database has a single entity: an author.

An author has four attributes:

- One `:author/olid` string
- One `:author/name` string
- One `:author/lastModified` instant
- One `:author/revision` long

Here's how we can represent our schema as a datastructure:

```clojure
(def schema
 ;; :author/olid (identity index)
 {:db/id #db/id[:db.part/db]
  :db/ident :author/olid
  :db/valueType :db.type/string
  :db/unique :db.unique/identity
  :db/index true
  :db/cardinality :db.cardinality/one
  :db.install/_attribute :db.part/db}
 ;; :author/name (fulltext index)
 {:db/id #db/id[:db.part/db]
  :db/ident :author/name
  :db/valueType :db.type/string
  :db/fulltext true
  :db/index true
  :db/cardinality :db.cardinality/one
  :db.install/_attribute :db.part/db}
 ;; :author/revision
 {:db/id #db/id[:db.part/db]
  :db/ident :author/revision
  :db/valueType :db.type/long
  :db/cardinality :db.cardinality/one
  :db.install/_attribute :db.part/db}
 ;; :author/lastModified
 {:db/id #db/id[:db.part/db]
  :db/ident :author/lastModified
  :db/valueType :db.type/instant
  :db/cardinality :db.cardinality/one
  :db.install/_attribute :db.part/db}])
```

To better understand `#db/id` and `#inst`, check out:

- [The Reader: Tagged Literals](http://clojure.org/reader#The Reader--Tagged Literals)
- [github.com/edn-format/edn#tagged-elements](https://github.com/edn-format/edn#tagged-elements)

But here's my layman's understanding:

- `#db/id [:db.part/db]` expands into an auto-incremented id that is unique to the `:db.part/db` partition. You can do something similar by evaluating `(d/tempid :db.part/db)`. You'll see it increment if you keep evaluating it.
- The `:db.part/db` partition is reserved for system things like schema. We'll be putting our actual data in the `:db.part/user` partition. I believe the third built-in partition is `:db.part/tx` for, I suppose, transactions.
- `:db.install/_attribute :db.part/db` is the key/value pair that tells Datomic that this is attribute schema.

Read more here: [http://docs.datomic.com/schema.html](http://docs.datomic.com/schema.html)

I encourage you to skim that link to see what the available Datomic datatypes are and other assorted facts.

Let's create a transaction that actually creates our schema.

```clojure
(def uri "datomic:free://localhost:4334/authordb")

(def conn (d/connect uri))

(defn db
  "Returns latest value of the database."
  []
  (d/db conn))

(defn reset-db []
  (d/delete-database uri)
  (d/create-database uri)
  @(d/transact (d/connect uri) schema))
```

`(reset-db)` demonstates a few useful functions and populates our `authordb` database with our schema.

(Since `d/transact` returns a Future, I deref it with `@` simply to block until it's finished. Though our schema loads instantly, it seems to be a good habit and/or necessary in certain contexts from what I gathered reading other people's code.)

To be sure that anything actually happened, let's find all `:db/ident` attributes and return their value:

```clojure
(d/q '[:find ?value
       :where [?entity :db/ident ?value]]
     (db))
```

You should see some of our attributes show up.

## Importing your own authors

To see what it's like, let's insert our own authors. 

Then I'll show a few query examples.

First, these are two similar ways you can express some sample data:

```clojure
(def custom-authors
  [{:db/id #db/id [:db.part/user]
    :author/name "Jane Doe"
    :author/olid "abc"
    :author/revision 1
    :author/lastModified #inst "2013-09-11"}
   {:db/id #db/id [:db.part/user]
    :author/name "Billy Joe"
    :author/olid "def"}])
```

Which is the same, as far as I can tell, to:

```clojure
(def custom-authors
  (let [jane (d/tempid :db.part/user)
        billy (d/tempid :db.part/user)]
    [{:db/id jane-id
      :author/name "Jane Doe"
      :author/olid "abc"
      :author/revision 1
      :author/lastModified #inst "2013-09-11"}
     {:db/id billy-id
      :author/name "Billy Joe"
      :author/olid "def"}]))
```

Notice that `:db/id` is generated for the user partition `:db.part/user` instead of the db partition `:db.part/db` like our schema.

```clojure
@(d/transact conn custom-authors)


(d/q '[:find ?e
       :where [?e :author/name]]
     (db))
;=> #{[123] [124]}

(d/q '[:find ?e ?v
       :where [?e :author/name ?v]]
     (db))
;=> #{[123 "Jane Doe"] [124 "Billy Joe"]}

(d/q '[:find ?v
       :where [_ :author/name ?v]]
     (db))
;=> #{["Jane Doe"] ["Billy Joe"]}

(let [ids (d/q '[:find ?e
       :where [?e :author/name]]
     (db))
;=> #{[123] [124]}

```

Here are some ways to actually interact with the results set.

```clojure
(let [results (d/q '[:find ?e
                     :where [?e :author/name]]
                   (db))]
  ;; results  ;=> #{[123] [124]}

  (let [ids (map first results)]
    ;; ids  ;=> [123 124]

    (let [entities (map #(d/entity (db) %) ids)]
      ;; entities  ;=> [<EntityMap> <EntityMap>]

      (let [maps (map d/touch entities)]
        ;; maps  ;=> [{:author/name "Jane Doe"
                 ;     :author/olid "abc"
                 ;     ...}
                 ;    {:author/name "Billy Joe"
                 ;     :author/olid "def"}

        (:author/name (first maps))
        ;=> "Jane Doe"
        ))))
```

Just `(reset-db)` when you want to clear your changes.

## Importing our parsed authors.txt data from the repl

Continuing where we left off, we've successfully parsed author.txt lines into maps that look like this:

```clojure
{:name "Khālid Muḥammad ʻAlī al-Ḥājj",
 :last_modified #inst "2008-02-02 17:57:09"
 :olid "OL1000057A"
 :revision 2}
```

It's a good intermediate data structure since we can use it to test our parser.

Now let's write a function that converts that map so that it can be imported into our database. The resulting map should look like this:

```clojure
{:db/id (d/tempid :db.part/user)
 :author/name "Khālid Muḥammad ʻAlī al-Ḥājj",
 :author/lastModified #inst "2008-02-02 17:57:09"
 :author/olid "OL1000057A"
 :author/revision 2}
```

That function could look like this.

```clojure
(defn filter-nil-values
  "Remove k/v pairs from a map where v is nil."
  [m]
  (into {} (remove (comp nil? second) m)))

(defn to-author-map [parsed-map]
  (let [{:keys [name last_modified olid revision]} parsed-map
        full-map {:db/id (d/tempid :db.part/user)
                  :author/name name
                  :author/lastModified last_modified
                  :author/olid olid
                  :author/revision revision}]
    (filter-nil-values full-map)))
```

I wrote a `filter-nil-values` utility function so that nils aren't persisted to the database.

```clojure
(filter-nil-values {:a 1, :b false, :c nil})
;=> {:a 1, :b false}
```

For instance, if a user hasn't selected a favorite color, then it makes sense for the user to have no `:user/favoriteColor` attribute rather than a nil `:user/favoriteColor`.

You now have what it takes to do a naive import batches of 100 lines at a time into the database with something like this:

```clojure
(doseq [line-batch (partition 100 (line-seq (io/reader "resources/authors.txt")))]
  @(d/transact conn (map (comp to-author-map parse-line) line-batch)))
```

That will take forever. But you can at least test it on the first 100 lines:

```clojure
(doseq [line-batch (take 100 (line-seq (io/reader "resources/authors.txt")))]
  @(d/transact conn (map (comp to-author-map parse-line) line-batch)))
```

(Notice the `take`) 

Clojure's laziness is a pleasure to work with. Remember that functions like map, filter, take, drop, line-seq, and remove return lazy sequences so it's trivial to compose them together without evaluating the entire collection. In this case, 6.8+ million strings.

Anyways, instead of importing data from the repl, I decided to import data from a file.

## Parsing authors.txt into authors.edn

I decided to parse authors.txt into Clojure maps and then serialize them into authors.edn.

- The first reason was that the Datomic resources I encountered demonstrated how to import .edn files into Datomic.
- The other reason was that having an intermediate .edn file between authors.txt and the Datomic import would let me debug any parsing issues.

Let's write a function that opens up "authors.txt" for reading, "authors.edn" for writing, parses each line of "authors.txt" into a map that can be directly imported into Datomic, and writes it to "authors.edn" one at a time.

```clojure
(defn write-author-maps []
  (with-open [r (io/reader "resources/authors.txt")
              w (io/writer "resources/authors.edn")]
    (doseq [line-string (line-seq r)]
      (let [author-map (to-author-map (parse-line line-string))]
        (.write w "{")
        (doseq [[k v] author-map]
          (.write w (str " " (pr-str k) " " (pr-str v) "\n")))
        (.write w "}\n"))))
  (println "DONE writing authors.edn"))
```

There's probably a better way but we're faking it til we make it.

I use `pr-str` to print the literal representation of Clojure values.

Run it in your repl. It'll take a while, but when it's done you should have a large "resources/authors.edn" file with a bunch of Clojure maps. We pretty much converted "authors.txt" into a representation compatible with our system. Imagine what the world will be like when more people discover the magic of Clojure and .edn universally replaces .txt, .csv, .html, .doc, .css, and .sql. Truly majestic.

Alright, now let's import it.

## Importing authors.edn

Unlike `line-seq`, there doesn't seem to be a built-in `edn-seq` function for lazily reading in our Clojure structures.

But someone wrote one on Stackoverflow for us:

```clojure
(defn edn-seq
  "Returns the objects from stream as a lazy sequence."
  ([]
     (edn-seq *in*))
  ([stream]
     (edn-seq {} stream))
  ([opts stream]
     (lazy-seq (cons (edn/read opts stream)
                     (edn-seq opts stream)))))

(defn swallow-eof
  "Ignore an EOF exception raised when consuming seq."
  [seq]
  (-> (try
        (cons (first seq) (swallow-eof (rest seq)))
        (catch java.lang.RuntimeException e
          (when-not (= (.getMessage e) "EOF while reading")
            (throw e))))
      lazy-seq))
```

If it makes you feel better, I never spent a moment figuring out how this interaction between PushbackReader and EOF exceptions works, but with only a minor pang of guilt for not understanding what my tools are actually doing, I cruised ahead.

Test those functions by reading the first 5 maps from our .edn file.

```clojure
(with-open [stream (PushbackReader. (io/reader "resources/authors.edn"))]
  (dorun (map println (take 5 (swallow-eof (edn-seq stream))))))
```

Now let's write the actual function that reads authors.edn into our database 1000 maps at a time.

We're pretty deep in my hack-it-until-it-works code, so forgive me. It's a JIT productivity optimization.

```clojure
(defn assoc-id 
  "Adds the :db/id key to a map with a unique value."
  [m]
  (assoc m :db/id (d/tempid :db.part/user)))
  
(defn transact-all []
  (doseq [tx-batch (as-> (io/reader "resources/authors.edn") _
                         (PushbackReader. _)
                         (edn-seq _)
                         (swallow-eof _)
                         (map assoc-id _)
                         (filter :author/olid _)
                         (partition-all 1000 _)
                         (pmap #(d/transact-async conn %) _))]
    @tx-batch)
  (println "DONE transact-all"))
```

(`->>` would have worked here but I find `as->` easier to read when I'm chaining together a mudball.)

Basically this is what's happening, 

- So, we're reading this lazy sequence of maps from authors.edn. 
- Then I map an `assoc-id` function over them (lazily) to give each one its own unique id (in the `:db.part/user` partition since it's our data)
- I use filter to (lazily) ignore any map without an `:author/olid` key/value as a last-second hack that will end up ignoring 4 of the 6.8 million rows which seem to be malformed legacy data.
- Then I partition them into batches of 1000 (lazily) so that I only insert 1000 at a time. I just came up with 1000 since it's less than 6.8 million.
- Then I `pmap` the `d/transact-async` function across all batches in parallel. (So far I've only demonstrated `d/transact`)
- `(d/transact-async conn batch-of-1000)` is executed in another thread due to `pmap` (as opposed to `map`) and returns a Future to this thread where we immediately deref it.

When it's all done, you should have 6.8+ million authors in your database.

I certainly didn't come up with that `pmap` -> `d/transact-async` -> `@deref` battleplan and I don't fully understand it, but it seems to have something to do with "pipelining" which I had trouble googling and there seems to be some significance in deref'ing in another thread than where the `d/transact-async` takes place.

But it seems to be the recommended way for importing lots of data.

See this gist (with a benchmark):

- [https://gist.github.com/danneu/6532819](https://gist.github.com/danneu/6532819)

And this Datomic Google Group question:

- [https://groups.google.com/forum/#!msg/datomic/wfns7jIuv8o/tbSlCiYNLXgJ](https://groups.google.com/forum/#!msg/datomic/wfns7jIuv8o/tbSlCiYNLXgJ)

## Querying our imported data

Now you should be able to query authors locally.

To be sure:

```clojure
(->> (d/q '[:find ?e
            :where [?e :author/name "John Scalzi"]])
     (ffirst)
     (d/entity (db))
     (d/touch))
```

Maybe you want to count how many authors you have in your database:


```clojure
(d/q '[:find (count ?e)
       :where [?e :author/name]]
     (db))
```

(Takes a while)

## That's it

That's as far as I managed to wing it for now.

I extracted this tutorial from files scatter-brained with random code experimentation, so this tutorial inherited the trait as well. I hope it helped in some way. Let me know if I left out some obvious details or if you get stuck.

My next step is to import OpenLibrary's 8gb book dump and match it up with the authors.

## Useful resources

- [day-of-datomic](https://github.com/Datomic/day-of-datomic) is a great Github repo with all sorts of Datomic examples that you can evaluate one line at a time in the repl. It's what got me started with Datomic.
- Datomic's [MusicBrainz sample database](http://blog.datomic.com/2013/07/datomic-musicbrainz-sample-database.html) demonstrates how to recreate most of MusicBrainz' relational database with Datomic.
- [Learn Datalog Today](http://www.learndatalogtoday.org/) provides an introduction to Datalog with in-browser tutorials.
- [Datomic: up and running](https://www.youtube.com/watch?v=ao7xEwCjrWQ) (Video): A screencast of someone creating an owners-have-many-pets tutorial app. A much better intro than my blog post. It even demonstrates a TDD workflow using the Expectations library.
