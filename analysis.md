# Introduction

The presented performance evaluation of [elasticsearch] has the following goals:

* Simulate typical value-based query use-cases for MEDS
* Measuring throughput of queries and indexing writes
* Examine the above behavior under single node and cluster conditions

# Setup

An elasticsearch cluster has been deployed to Openstratus/C3 having the following properties per node:

- 2 VCPUs, 6GB RAM, 30GB Disk (medium)
- Red Hat Enterprise Linux Server release 6.3 (Santiago)
- Linux kernel 2.6.32 (64-bit)

The cluster includes 6 nodes having the default elasticsearch index configuration which is as follows:

- 5 primary shards
- 5 replicat shards

![Elasticsearch cluster shards overview](cluster_shards.png)

![Elasticsearch VM overview](cluster_vm.png)

# Index

## Mapping

The index mapping is defined as follows:

    {
        "mappings": {
            "entry": {
                "properties": {
                    "attribute_name": {
                        "index": "no",
                        "store": "yes",
                        "type": "string"
                    },
                    "identity_uuid": {
                        "index": "no",
                        "store": "yes",
                        "type": "string"
                    },
                    "namespace_id": {
                        "index": "no",
                        "store": "yes",
                        "type": "long"
                    },
                    "value_name": {
                        "index": "analyzed",
                        "store": "no",
                        "type": "string"
                    },
                    "value_number": {
                        "index": "analyzed",
                        "store": "no",
                        "type": "long"
                    }
                }
            }
        }
    }

It is a simple mapping with the following result attribute tuple:

* attribute_name
* identity_uuid
* namespace_id

and the following possible query value attributes:

* value_number: A random number in the range [0, 98]
* value_name: A random 5 character string with the following possible alphanumeric character values: (a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r, s, t, u, v, w, x, y, z, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9)

The attribute value_name was choosen such that there is a low probability of a document to be found with a random query string. The attribute value_number was choosen such that there is a high probability of a document to be found with a random query number.

# Scenarios

- index single document: This scenario inserts a single document per request
- bulk index 100 documents: This scenario
- low hit probability query
- high hit probability query

## Common scenario properties

All scenarios share the following properties:

- the measurements are being performed with 1,5 and 10 parallel users
- in the case of index, low-p-query and high-p-query each user executes 50,000 requests
- in the case of a bulk index request each user executes 5,000 requests each including 100 bulk document inserts.

# Results

## Low probability query

The following query is being executed resulting in a very low result hit probability:

    {
      "query": {
        "term": { "value_name": "$value_name" }
      }
    }

The resutling document size is mostly 0 or 1 documents.

![50,000 requests - 1 user](getlowp_50000_1.png)

![50,000 requests - 5 users](getlowp_50000_5.png)

![50,000 requests - 10 users](getlowp_50000_10.png)

## High probability query

The following query is being executed resulting in a very high result hit probability:

    {
      "query": {
        "term": { "value_number": $value_number }
      }
    }

The resulting document size is mostly ~100,000 documents (inter-shard hits are very probable).

![50,000 requests - 1 user](gethighp_50000_1.png)

![50,000 requests - 5 users](gethighp_50000_5.png)

![50,000 requests - 10 users](gethighp_50000_10.png)

## Index single document

The index scenario inserts the following document per request:

    {
      "attribute_name": "test",
      "identity_uuid": "$identity_uuid",
      "namespace_id": $namespace_id,
      "value_name": "$value_name",
      "value_number": $value_number
    }

![50,000 inserts - 1 user](index_50000_1.png)

![50,000 inserts - 5 users](index_50000_5.png)

![50,000 inserts - 10 users](index_50000_10.png)

## Bulk index 100 documents per request

The bulk index scenario inserts the following 100 documents per request:

    100 times {
        { "index" : { "_index" : "entries", "_type" : "entry" } }
        { "attribute_name": "test", "identity_uuid": "$identity_uuid", "namespace_id": $namespace_id, "value_name": "$value_name", "value_number": $value_number}
    }

### Cluster results

![5,000 inserts (500,000 documents) - 1 user](bulk_5000_1.png)

![5,000 inserts (500,000 documents) - 5 users](bulk_5000_5.png)

![5,000 inserts (500,000 documents) - 10 users](bulk_5000_10.png)

[gatling]: http://gatling-tool.org 
[elasticsearch]: http://elasticsearch.org

## Cold cluster query results

The following graphs show the behavior of a cold cluster (i.e. newly started VMs) for a high-p and low-p query for 1 user:

![50,000 requests - 1 user - cold cluster](getlowp_50000_1_coldcluster.png)
![50,000 requests - 1 user - cold cluster](gethighp_50000_1_coldcluster.png)

## 14 hours test

Two long running tests performed on two seperate client machines were executed on the cluster:

- 10 concurrent users per client (20 concurrent users total) executing a high-p query
- 300 concurrent users per client (600 concurrent users total) executing a high-p query

The results show a linear behavior with 20 concurrent users:

![20 users across 2 clients over 14 hours summary](14hrs_10users_avg.png)
![20 users across 2 clients over 14 hours req/sec](14hrs_10users_reqpersec.png)

Stressing the cluster with 600 users over 14 hours shows a two hickups which were symmetrical between the clients so the conjecture is local network issues. No exceptions could be found on the elasticsearch instances:

![600 users across 2 clients over 14 hours summary](client1_600_14hrs.png)
![600 users across 2 clients over 14 hours req/sec](client2_600_14hrs.png)

## Comparison with Voyager

Elasticsearch was compared with Voyager with a production index taken from the subsidiary site mobile.de. Both Voyager and elasticsearch were deployed in a production data center in Frankfurt/Germany having two physical nodes each. Elasticsearch was set up with only one shard and one replica on the other node.

The total amount of documents was 2,5 million occupying 2,5GB of index space.

The following results reflect random real-production queries fired against voyager without faceting/grouping:

### Without faceting/grouping queries

![](voyager40_without_facets.png)

In comparison elasticsearch without faceting/grouping:

![](es40_without_facets.png)

### With faceting/grouping queries

The next results reflect random real-production queries fired against voyager including faceting/grouping:

![](voyager40_with_facets.png)

In comparison elasticsearch with faceting/grouping:

![](es40_with_facets.png)
