Getting data out of Automate via the CLI. All of these curls need to be run from on the automate server if you're using the built in Elasticsearch. If you're running your own it needs to be from a node that can reach 9200 on your ES servers.

Make sure ES works
[root@automate ~]# curl localhost:9200
{
  "name" : "Piotr Rasputin",
  "cluster_name" : "chef-insights",
  "cluster_uuid" : "hTTtuPxvRu6ZPKsUOlW54A",
  "version" : {
    "number" : "2.4.1",
    "build_hash" : "c67dc32e24162035d18d6fe1e952c4cbcbe79d16",
    "build_timestamp" : "2016-09-27T18:57:55Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.2"
  },
  "tagline" : "You Know, for Search"
}

Show indices and their sizes. Useful to make sure that data is getting into ES and planning out data usage.
We use -s to prevent curl from showing connection info and sort to order the indices by date.
Yellow for index health means that the shards only exist on one node. This is normal for a single server automate cluster, for multi-node they should be green. Red means either ES just started and isn't fully up yet, or that data is corrupted.
[root@automate ~]# curl -s localhost:9200/_cat/indices | sort
yellow open .automate           5 1  1 0   3.8kb   3.8kb
yellow open insights-2017.03.16 5 1  2 0   289kb   289kb
yellow open insights-2017.03.21 5 1 60 0 338.8kb 338.8kb
yellow open insights-2017.03.22 5 1 15 0 398.9kb 398.9kb
yellow open insights-2017.03.23 5 1 44 0   3.3mb   3.3mb
yellow open insights-2017.03.29 5 1 14 0   2.2mb   2.2mb
yellow open insights-2017.03.30 5 1 10 0   1.6mb   1.6mb
yellow open insights-2017.03.31 5 1 33 0   2.4mb   2.4mb
yellow open insights-2017.04.07 5 1 12 0   1.7mb   1.7mb
yellow open insights-2017.04.10 5 1 44 0 615.8kb 615.8kb
yellow open .kibana             5 1 14 1 107.4kb 107.4kb
yellow open node-state-1        5 1  3 1     1mb     1mb
yellow open saved-searches      5 1  2 0  11.2kb  11.2kb

Show health of elasticsearch. Since this response comes back as json we're adding "pretty=true" to make it easier to read. You can leave this off if you're polling this via a script for monitoring purposes.
[root@automate ~]# curl -s localhost:9200/_cluster/health?pretty=true
{
  "cluster_name" : "chef-insights",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 65,
  "active_shards" : 65,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 65,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 50.0
}

ES queries are fed in as JSON and can be passed in as url args, however this is fairly unwieldy. It's easier to put your json in a file and pass it in with -d. It's also important to make your searches as narrow as possible since node data is quite large you can easily request that ES feed you gigabytes of data. Here's a request searching for one converge item and only returning the name of the node that returned.


curl -XPOST -s localhost:9200/insights-2017.03.29/_search?pretty=true -d@data.json

[root@automate tmp]# cat data.json
{
    "from" : 0, "size" : 1,
    "_source": ["node_name"],
    "query" : {
        "term" : { "_type" : "converge" }
    }
}

{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 14,
    "max_score" : 1.0,
    "hits" : [ {
      "_index" : "insights-2017.03.29",
      "_type" : "converge",
      "_id" : "AVsb3ptQVZcNcHgTt-sE",
      "_score" : 1.0,
      "_source" : {
        "node_name" : "jhud_apache01"
      }
    } ]
  }
}

You can take the limiter out and add use bash to make this a usefull check to see how many times each node showed up in ES. Every Chef run should make two converge entries, one at the start of the run and one when it finishes.

[root@automate tmp]# cat data.json
curl -XPOST -s localhost:9200/insights-2017.03.29/_search?pretty=true -d@data.json | grep node_name | sort | wc -l

{
    "_source": ["node_name"],
    "query" : {
        "term" : { "_type" : "converge" }
    }
}

[root@automate tmp]# curl -XPOST -s localhost:9200/insights-2017.03.29/_search?pretty=true -d@data.json | grep node_name | sort | uniq -c
      8         "node_name" : "jhud_apache01"
      2         "node_name" : "runner.e9.io"

So on march 29th jhud_apache01 checked in 4 times, and runner.e9.io once.

You can also pull one single node object out. This query is more advanced than the ones we've done before so let's walk through it. First we want the first page(page 0) and 1 result per page. This effectively limits our search to one result. We then exclude "node" from the _source object. This is because node is a raw string that contains the original node object, all of the items are later in the json so it's redundant here. Then since we want multiple terms we use a bool query to chain them together. We're searching for a converge run that has succeeded, so the node object pushed up at the end of a run,

curl -XPOST -s localhost:9200/insights-2017.03.29/_search?pretty=true -d@data.json

[root@automate tmp]# cat data.json
{
  "from" : 0, "size" : 1,
  "_source" : {
    "exclude" : ["node"]
    },
  "query" : {
    "bool" : {
      "must" : [
        {
          "term" : { "_type" : "converge" }
        },
        {
          "term" : { "status" : "success" }
        }
      ]
    }
  }
}


