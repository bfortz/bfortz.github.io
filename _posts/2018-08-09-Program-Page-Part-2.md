---
layout: post
title: "A new conference program page: Building static data files"
tags: EURO conferences OR2018 clojure
---

*The current way the conference program is handled generates too many databases access to be efficient. The relevant information should be stored in a static file that could then be cached efficiently. We will see how to achieve this.* 

In [the previous post]({% post_url 2018-08-06-Program-Page-Part-1 %}), I described some choices for the development of a progressive web app to allow to browse the [program](https://www.euro-online.org/conf/or2018/program) of conferences using the [EURO](https://www.euro-online.org) conference tool, and in particular the [Operations Research 2018](https://www.or2018.be) that I am organising. 

The first step is the creation of a single data file with the relevant information. To achieve this, we will build a small clojure program that takes the information from the database, transform it to eliminate information we don't need here and correct some inefficiencies in the design of the database (an important task in the future will be to redesign that database, but this is another story).

Enough talk, let's go. The full code presented in this post is available
[here](https://github.com/bfortz/euro-edn-conference-program/tree/cd0b0d8025285f223590d767a17acd78f582a1ce).

**Update (August 10, 2018):** *I just refactored the code for efficiency. The new version
is in [the master branch of the
repository](https://github.com/bfortz/euro-edn-conference-program). I might
make a post about the improvements if I find time...*

## Project requirements and configuration

This is a simple clojure app, we are going to use [HugSQL](https://www.hugsql.org/) to query the mariadb database, so we have really few dependencies in our `project.clj` file:

{% highlight clojure %}

  :dependencies [[org.clojure/clojure "1.9.0"]
                 [com.layerware/hugsql "0.4.9"]
                 [mysql/mysql-connector-java "8.0.12"]]

{% endhighlight %}

In order to be able to run the program on different systems (useful for debugging, but also in the long run if one day the system gets fully open-sourced), we define all the system-dependent configuration variables in `env/euro_edn_conference_program/config.clj`. For obvious security reasons, this file is not in the repository but a template of it looks like this:

{% highlight clojure %}

(ns euro-edn-conference-program.config)

(def output-path "/path/to/output/files")
(def userdb "databases-of-users")

(defn db-spec [dbname]
  "Returns mysql connection information for database dbase"
  {:subprotocol "mysql"
   :subname (str "//server:port/" dbname)
   :user "username"
   :password "password"})

{% endhighlight %}

`output-path` is the directory where the edn file will be saved. Since the file will be read by the client app, this should be in place served by the webserver. `userdb` is the name of the database where user profiles are stored. Finally, `db-spec` is a function that takes a database name as argument and returns all the connection information necessary to run queries in our application.

## Getting data from the database

[HugSQL](https://www.hugsql.org/) is a very convenient library that transform a set of SQL queries into Clojure functions, with results of queries stored as Clojure hash-maps.

For our purpose, to minimize the load on the database, we simply import all the information we need. These very basic sql queries are defined in `src/euro_edn_conference_program/queries.sql`. As an example, to get all the timeslots, we simply write:

{% highlight sql %}

-- :name all-timeslots :? :*
-- :doc Get all timeslots
select id, schedule, day, time from timeslots

{% endhighlight %}

and we get a Clojure function `all-timeslots` that can be used to get a hash-map of all timeslots.

In our application (`src/euro_edn_conference_program/queries.sql`), we first require all dependencies and load the functions associated with these queries:

{% highlight clojure %}

(ns euro-edn-conference-program.core 
  (:require [hugsql.core :as hugsql] 
            [clojure.set :as s]
            [euro-edn-conference-program.config :as cf])
  (:gen-class))

(hugsql/def-db-fns "euro_edn_conference_program/queries.sql")

{% endhighlight %}

## Data transformation

The client app we will be next should allow to browse the program by time,
stream, author or keyword. To lighten the load on the client, we will transform
the data to allow fast access without too much processing.

### Timeslots, rooms, keywords

For many simple data, we just need to access it by a given key. The most efficient way to do that in Clojure(script) is to use hash-maps. As we also need to display information in a sorted way, we will use sorted maps. The raw data extracted from the database is simply a list of hash-maps with the attributed selected in the sql query.  We start by defining two little helper functions:

{% highlight clojure %}

(defn to-hashmap [k x]
  "Transforms a list of hashmaps in a hashmap of hasmaps using key k"
  (reduce #(assoc %1 (k %2) (dissoc %2 k)) (sorted-map) x))

(defn extract-data-fn [f k]
  "Creates a function that extracts from the database with function f and makes a hashmap based on key k"
  (fn [conf]
    (->> (f (cf/db conf))
         (to-hashmap k))))

{% endhighlight %}

The first function is used to convert a list of hashmaps into a sorted map based on a given key:

{% highlight clojure %}
euro-edn-conference-program.core=> (to-hashmap :key '({:key 1, :value "first val"} {:key 2, :value "seconv val"}))
{1 {:value "first val"}, 2 {:value "seconv val"}}
{% endhighlight %}

I won't enter in a detailed explanation of it, but it is quite straightforward once you know a bit of functional programming and the way Clojure handles hash-maps.

The second function creates a function with a single argument `conf` (a database name), and uses HUGSql function `f` to retrieve data from this database then transforms it to a sorted map indexed on `k`.
This code might seem a bit strange if you are not familiar with clojure, but it
is pretty straightforward: the threading macro `->>` takes an value and applies
a series of operations on it, using the result of the last operation as last
argument to the next one. 
The newly created function applies `f` to the database connection obtained from the configuration file by applying the `db` function to the `conf` argument, then applies `to-hashmap` with the key `k` to the list obtained in the first step. 

Now we can easily build function extraction data for timeslots, rooms and keywords. We will also need to retrieve users by username later on, so we also build such a helper function:

{% highlight clojure %}

(def timeslots (extract-data-fn all-timeslots :id))
(def rooms (extract-data-fn all-rooms :track))
(def keywords (extract-data-fn all-keywords :id))
(def users-by-username (extract-data-fn all-profiles :username))

{% endhighlight %}

and we get the data for a given conference very easily:

{% highlight clojure %}

euro-edn-conference-program.core=> (timeslots "or2018")
{1 {:schedule "Wednesday, 9:00-10:30", :day "W", :time "A"}, 2 {:schedule "Wednesday, 11:00-12:40", :day "W", :time "B"}, 3 {:schedule "Wednesday, 14:00-15:40", :day "W", :time "C"}, 4 {:schedule "Wednesday, 16:10-17:50", :day "W", :time "D"}, 5 {:schedule "Thursday, 9:00-10:15", :day "T", :time "A"}, ...}

{% endhighlight %}

### User helper functions

The design of the conference databases has some problems that we are trying to counter here. One of the major ones is that most table involving users (such as the assignment of session chairs or co-authors of papers) store the username, while the id of the user would be more convenient.

In order to lookup a user by username easily, we use our transformation function above to get a function mapping user attributes to usernames:

{% highlight clojure %}
(def users-by-username (extract-data-fn all-profiles :username))
{% endhighlight %}

We can then use the map returned by this function applied to a given conference database in order replace usernames by userids in the data returned by HugSql.

{% highlight clojure %}
(defn convert-usernames [f usermap conf]
  "Creates a function that extracts from the database with function f and converts usernames to id using hashmap usermap"
  (letfn [(username->id [x] 
            (->> (:user x) 
                 (get usermap)
                 (:id)
                 (assoc x :user)))]
    (->> (f (cf/db conf))
      (map username->id))))
{% endhighlight %}

The function takes a HugSql function `f` for looking up the database, a usermap as returned by `users-by-username` and a conference database name, and returns the result of getting data with `f` for the conference `conf` where the usernames are replaced by ids. To achieve this, we apply a helper function `username->id` to each element of the list returned by the call to `f`. This helper function is quite straightforward: it takes the `:user` attribute in the element `x` (which is a username), gets the corresponding map of user details in usermap, then lookup the id of the user in the map and finally replaces the username by this id in the `x` map.

### Streams

In addition to the raw stream data in the corresponding table in the database, as we want to be able to browse the program by stream easily, it will be useful to have a list of sessions in the stream ordered chronologically. 

To achieve this, we need to order sessions in the stream by their timeslot id. Unfortunately, due to some bad legacy decisions in the design of the database, sessions are assigned to a time slot by a day and time instead of timeslot id. A handy helper function, taking as arguments a day, a time and a map of timeslots (as the one returned by the timeslots function), will lookup the timeslots map and return the id for the day and time:

{% highlight clojure %}

(defn daytime->id [timeslots day time]
  (->> timeslots
       (filter #(and (= day (:day (second %)))
                     (= time (:time (second %)))))
       (first)
       (first)))

{% endhighlight %}

The function uses the useful threading macro `->>` again to transform our data.
We start with the list of timeslots. This list is
made of pairs of elements, the first one being the id of the timeslot, the
second one a map of the timeslot attributes. The first operation filters the
list to only keep elements with the required day and time. Given an element of
the list, the day and time and given by the `:day` and `time` keys in the
second element of the pair. After this operation, we are left with a list with
only one element. We apply first to get this single element of the list, which
is thus a pair (id, attributes), and as we only want the id, we apply first a
second time.

{% highlight clojure %}
euro-edn-conference-program.core=> (daytime->id (timeslots "or2018") "W" "B")
2
{% endhighlight %}

Now we can define a function that returns a map of streams, indexed on their id, with an additional `:sessions` attribute which is the list of sessions in the stream ordered by timeslot id. This function takes as argument raw streams and session lists as given by the HugSQL functions, and the map of timeslots.

{% highlight clojure %}

(defn streams [rawstreams rawsessions timeslots]
  "Returns a hashmap of streams by id, with an embedded list of sessions ordered by timeslot"
  (letfn [(addsessions [x]
            "Adds list of sessions"
            (let [s (->> rawsessions
                         (filter #(= (:cluster %) (:id x)))
                         (sort-by #(daytime->id timeslots (:day %) (:time %)))
                         (map :id))]
              (assoc x :sessions s)))]
    (->> rawstreams
      (map addsessions)
      (to-hashmap :id)))) 

{% endhighlight %}

The `letfn` statement define a helper function that takes a stream `x` and adds to the map an attribute `:sessions` with the list `s` of sessions in the stream, ordered by timeslot id.
To do this, we filter the list of sessions to only keep those for which the `:cluster` attribute is the id of the current stream (streams are called clusters in the database for legacy reasons). The next step sorts the session according to the timeslot id, that we obtain with our helper function `daytime->id`. Finally, we only need a list of session ids and not the full information, so we map the result to the function that returns the `:id` attribute of the session.

The main body of the function takes the list of streams, apply the `addsessions` function to each element in order to add the `:sessions` attribute, and finally converts the result to an ordered map of streams indexed by id.

{% highlight clojure %}

euro-edn-conference-program.core=> (streams rawstreams rawsessions ts)
{1 {:name "Business Track", :order 2, :sessions (46)}, 2 {:name "Decision Theory and Multiple Criteria Decision Making", :order 4, :sessions (85 2 175 176 177)}, 3 {:name "Business Analytics, Artificial Intelligence and Forecasting", :order 1, :sessions (163 201 200 202)}, 4 {:name "Finance", :order 7, :sessions (195 196 197 198)}, 5 {:name "Health Care Management", :order 11, :sessions (170 171 173 172 174)}, ...}

{% endhighlight %}

### Sessions

Sessions information is handled similarly to streams. Starting with a list of raw session information, we want to easily reach the associated papers, find the session chair(s), and know the timeslot. Again, legacy decisions make our work a bit harder:

- Papers are associated to sessions by session submission code, and not by session id.  The helper function `addpapers` takes a session `x`, retrieves the papers with the same code as `x`, orders them by `:order` attributes, and add the list of paper ids as attribute `:papers` to `x`.
With these preliminaries, the code of the `sessions` function 
- The timeslot for a session is defined by day and time, we prefer the timeslot id that is added to the map by `addtimeslot`.
- The list of chairs is extracted by `addchairs` from the list `chairs` passed as argument to the function. This list should be the result of `convert-usernames` applied to HugSql function `all-chairs` so that users are already mapped to their id.

Finally, in order to be consistent in naming, we define a `:stream` attribute with the `:cluster` value, and remove the information that is not needed anymore, namely the code, day, time, cluster.

{% highlight clojure %}

(defn sessions [rawsessions rawpapers chairs timeslots]
  "Returns a hashmap of sessions by id, with an embedded list of papers"
  (letfn [(addpapers [x]
            "Adds list of abstracts"
            (let [p (->> rawpapers
                         (filter #(= (:code x) (:sessioncode %)))
                         (sort-by :order)
                         (map :id))]
              (assoc x :papers p)))
          (addtimeslot [x]
            "Adds timeslot id for day and time"
            (assoc x :timeslot (daytime->id timeslots (:day x) (:time x))))
          (addchairs [x]
            "Adds list of chairs"
            (let [c (->> chairs
                         (filter #(= (:session %) (:id x)))
                         (map :user))]
              (assoc x :chairs c)))]
    (->> rawsessions
      (map addpapers)
      (map addtimeslot)
      (map addchairs)
      (map #(assoc % :stream (:cluster %)))
      (map #(dissoc % :code :day :time :cluster)))))

{% endhighlight %}

### Papers

Papers are handled in a similar way. We want to add the list of authors and to identify the session by id instead of code. So we first add this little helper:

{% highlight clojure %}
(defn code->id [rawsessions code]
  (->> rawsessions
       (filter #(= (:code %) code))
       (first)
       (:id)))
{% endhighlight %}

and the code follows a structure similar to what we saw before:

{% highlight clojure %}
(defn papers [rawpapers rawsessions authors]
  "Returns a hashmap of papers by id, with session codes converted to session ids and an embedded list of authors"
  (letfn [(addsession [x] 
            "Replaces session code by session ids"
            (assoc x :session (code->id rawsessions (:sessioncode x))))
          (addauthors [x]
            "Adds list of authors"
            (let [a (->> authors
                         (filter #(= (:paper %) (:id x)))
                         (sort-by :speaker)
                         (map :user))]
              (assoc x :authors a)))]
    (->> rawpapers 
      (map addsession)
      (map addauthors)
      (map #(dissoc % :order :sessioncode))
      (to-hashmap :id))))
{% endhighlight %}

Note that we remove the session code as it has been replaced by the id, and also the `:order` attribute as the order of papers in a session is now embedded in the list of papers inside the session map.

### Users

The last missing part is user information. For each user, we want to easily access the sessions she/he is involved in as author or session chair. For session chairs, the information is directly accessible in the `chair` map passed as argument (which contains as before the list of session chairs information returned by `all-chairs` with converted usernames).
On the other hand, the `authors` map associates users to papers for which they are co-author. A helper session will give us the session id for a given paper id:

{% highlight clojure %}
(defn paper-session [rawpapers rawsessions id]
  "Returns the sessionid for paper id"
  (->> rawpapers
       (filter #(= (:id %) id))
       (first)
       (:sessioncode)
       (code->id rawsessions)))
{% endhighlight %}

What it does is filter the paper list to find the matching id, takes the single result returned and extract the session code, then converts it to the session id using `code->id`.

Next we add to each user the list of relevant sessions: we create one set of sessions where the user is chair, and the second for sessions where the user is co-author. We use a set structure here because we want to remove duplicates. We then merge these two sets, and sort the resulting sessions by timeslot id. In the main body, we also filter the list to eliminate all users from the database that are not involved in any session.

{% highlight clojure %}
(defn users [rawusers authors chairs rawpapers rawsessions timeslots]
  "Returns a hashmap of users by id, with a list of sessions there are involved in ordered by timeslots"
  (letfn [(addsessions [x]
            "Adds a list of sessions"
            (let [id (:id x)
                  as-chair (->> chairs
                                (filter #(= id (:user %)))
                                (map :session)
                                (set))
                  as-author (->> authors
                                 (filter #(= id (:user %)))
                                 (map :paper)
                                 (map #(paper-session rawpapers rawsessions %))
                                 (filter identity)
                                 (set))
                  session-set (s/union as-chair as-author)
                  s (->> rawsessions
                         (filter #(contains? session-set (:id %)))
                         (sort-by #(daytime->id timeslots (:day %) (:time %)))
                         (map :id))]
              (assoc x :sessions s)))]
    (->> rawusers
      (map addsessions)
      (filter #(not (empty? (:sessions %))))
      (map #(dissoc % :username)) 
      (to-hashmap :id))))
{% endhighlight %}

This function is quite heavy if run on all users in the database, while most of them are not involved in the conference. To speed up things, we write a filtering function that only keeps users that are either authors or chairs. The functions takes a list of users `u` and a list of sets of maps with a `:user` attribute, and keeps only users that belong to these sets. The function `to-set` returns a set of user ids present in map `x`. Then userset is obtained by merging the sets obtained by applying `to-set` to every map in `s`, and we only keep users belonging to `userset`.

{% highlight clojure %}
(defn to-set [x]
  (set (map :user x)))

(defn filterusers [u & s]
  "Keeps only users in u that are in key :user for sequences provided"
  (let [userset (reduce #(s/union %1 (to-set %2)) #{} s)]
    (filter #(contains? userset (:id %)) u)))
{% endhighlight %}

## The full program data

Time to put everything together. The `program-data` function takes a conference
database name as argument, and returns a map that will be consumed by our
client app.  The `let` blocks initializes all the raw data from the database
(to avoid querying repeatedly the database) and the map is then filled by
calling all the previously defined functions.

{% highlight clojure %}

(defn program-data [conf]
  "Returns a hashmap with all the data of the program of conference conf"
  (let [ts (timeslots conf)
        rawpapers (all-papers (cf/db conf) conf)
        rawsessions (all-sessions (cf/db conf) conf)
        rawstreams (all-streams (cf/db conf) conf)
        umap (users-by-username cf/userdb)
        ca (convert-usernames all-coauthors umap conf)
        ch (convert-usernames all-chairs umap conf)
        allusers (all-profiles (cf/db cf/userdb) cf/userdb)
        rawusers (filterusers allusers ca ch)]
    {:timeslots ts, 
     :streams (streams rawstreams rawsessions ts),
     :sessions (sessions rawsessions rawpapers ch ts),
     :rooms (rooms conf),
     :papers (papers rawpapers rawsessions ca),
     :users (users rawusers ca ch rawpapers rawsessions ts)}))

{% endhighlight %}

## Writing the data to a file

Our objective is to write the program data to a static file that can be served
by the webserver to be consumed by a client app. The program takes a conference
database name `conf` on the command-line, generates the corresponding data with
`program-data` and writes the resulting map to a file called `conf.edn` in the configured
output path directory (which should be defined as the path for the webserver to
serve it).

Our program will be run regularly (e.g. by putting it in a crontab). If data did not change, it would be inefficient to overwrite the existing file, because it would break caching features. So we add a little bit of code that takes the data and the output filename, and checks if the data changed by comparing the hash of the two maps of data. The file is only written if the file does not exist yet or if the two hashes differ, meaning information changed.

{% highlight clojure %}
(defn changed? [data filename]
  (try
    (let [new-hash (hash data)
          old-hash (hash (read-string (slurp filename)))]
      (not= new-hash old-hash))
   (catch Exception _ true))) 

(defn -main
  "Generates an edn program file format for a database passed as first argument"
  [& args]
  (let [conf (first args)
        filename (str cf/output-path "/" conf ".edn")
        newdata (program-data conf)]
    (when (changed? newdata filename) 
      (spit filename newdata))))

{% endhighlight %}

## Putting it to work

This program is now run every three hours for the [OR2018](https://www.or2018.be) conference. It takes about 10 seconds for each run, and the resulting file size is 840K. To check if it scales, I also tested the generation on the [EURO2018](http://euro2018valencia.com/) conference data, which was much bigger. Generation takes around 27 seconds (*down to 12 with recent improvements*) for a file of size 3.1M. 

I also configured Cloudflare to force caching of these files, and the access time is very good, and the data will only be reloaded when it changes. 
Check by yourself:
- [OR2018 data](https://www.euro-online.org/conferences/program/or2018.edn)
- [EURO2018 data](https://www.euro-online.org/conferences/program/euro29.edn)

This is a good start for the app development, which we will start working on in the next post. Keep tuned!

As always, I would be happy to get your feedback. Contact me by [by
email](mailto:bernard.fortz@gmail.com), on
[Twitter](https://www.twitter.com/djkrazyben) or by leaving comments below.
