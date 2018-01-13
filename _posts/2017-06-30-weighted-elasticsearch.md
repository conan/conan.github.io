---
layout: post
title: "Weighted Tags in Elasticsearch"
date: 2017-06-29 13:30:00
redirect_from: /elasticsearch/2017/06/29/weighted-elasticsearch.html
categories: elasticsearch
comments: true
published: true
---

Elasticsearch is good for searching the contents of documents.  By default the queries you write are run against the words in each document, and the documents are scored and returned to you.  Sometimes however, you want to use some other aspect of a document (rather than its contents) to determine its score.  In this post, I'll give an example of how to search for and score documents based on their _weighted tags_ - essentially metadata about each document - rather than the contents of the document itself.

## Tagging

Tagging items is a useful way of providing metadata, and we can use that to search. Let's look at an example: a car database. Each car has a colour, or maybe several colours. We can tag each car with each colour it hsa. When somebody searches for the term "red", we want to show them all the cars that are red. However, we might also want to rank them by _how red they are_.  

Some cars (e.g. the [Mini Cooper](http://www.malaysiaminilover.com/photo/mk1/MiniMK1_minicooper1.jpg)) come in an all-red version, so we want to score those very highly.  Some cars come with only a bit of red (e.g. the [white Fiat 500 with red, green and white stripes](http://www.canadianautoreview.ca/images/car_photos/2015-fiat-500-turbo/normal/fiat-500-turbo-fs1.jpg)), so we want to return those but rank them low.  We tag each car with all the colours it has, and we give each of these tags a weight: the Mini Cooper which is all red gets weight 1 the `red` tag, the Fiat 500 with the stripes gets the `red` tag weighted at 0.1. 

We can't use standard Elasticsearch scoring for this ranking, as that's based on the occurrences of words within a document.  The word "red" only occurs once for each red car, and it's the _weight_ we want to use for scoring. Luckily, Elasticsearch allows us to do this.


## Querying weighted tags

First, let's set up Elasticsearch to index cars. It's easy to write a query that will return cars that have `name` and `brand` fields if we search for "Fiat" or "500" by matching the name or brand. But if we simply added a `colour` field in the same way, we could list all the colours that the car is, but we don't have a way of telling Elasticsearch what the weighting for each colour should be. 

Instead, we create a `tag` [nested document](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html) for each colour we want to tag the car with.  These documents will have a tag and a weight.  Here are the mappings:

{% highlight javascript %}
PUT /cars
{
	"mappings": {
		"car": {
			"properties": {
				"brand": {
					"type": "text"
				},
				"name": {
					"type": "text"
				},
				"tags": {
					"type": "nested",
					"properties": {
						"tag": {
							"type": "text"
						},
						"weight": {
							"type": "float"
						}
					}
				}
			}
		}
	}
}
{% endhighlight %}

Nested documents are indexed just like normal documents except they cannot be queried independently of their parent; this is what we want - we'll use them to find their parents.  Now we can add tags to our documents like this:

{% highlight javascript %}
PUT /cars/car/1
{
	"brand": "Fiat",
	"name": "500",
	"tags": [
		{
			"tag": "red",
			"weight": 0.1
		},
		{
			"tag": "green",
			"weight": 0.1
		},
		{
			"tag": "white",
			"weight": 1
		}
	]
}
{% endhighlight %}


We can query them by using a [nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html) for the string "red":

{% highlight javascript %}
PUT /cars
{
	"query":{
		"nested": {
			"path": "tags",
			"score_mode": "sum",
			"query": {
				"function_score": {
					"query": {
						"match": {
							"tags.tag": "red"
						}
					},
					"field_value_factor": {
						"field": "tags.weight"
					},
					"boost_mode": "replace"
				}
			},
			"inner_hits": {"size": 10}
		}
	}
}
{% endhighlight %}


Let's break this down.  

* We've told Elasticsearch we want to query nested documents by using the `nested` query type, and we've specified the `path` we want to query against - this is how Elasticsearch knows which set of nested documents to query (in case you have several). 

* The `score_mode` we've set is saying to add up the scores of all the nested documents that match the query.

* The query we use for the nested documents is a [Function Score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html) - this simply means that we're going to use a normal query, but tell Elasticsearch how to score the documents that match rather than using the default scoring algorithm. First, we pass the Function Score query a simple match query on the tag, with our input string.  Note that we have to use the nested path, because nested documents are always found via their parent. We then specify what scoring system to use; there are [several available](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#score-functions), but we're using a [field_value_factor](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-field-value-factor), which lets us tell Elasticsearch to use the value of one of the fields in the document as the score - in our case, the tag weight. Finally with the `boost_mode` we're telling Elasticsearch to replace its default score entirely with the field value (it's possible to use the field value as a modifier instead of the whole score).

* The `inner_hits` is optional, but it allows us to see which nested documents matched the query, which is handy for things like highlighting which words matched in the input string.

Here's an example of the response we get:

{% highlight javascript %}
{
    "took": 26,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.1,
        "hits": [
            {
                "_index": "cars",
                "_type": "car",
                "_id": "1",
                "_score": 1.1,
                "_source": {
                    "name": "500",
                    "brand": "Fiat",
                    "tags": [
                        {
                            "tag": "red",
                            "weight": 0.1
                        },
                        {
                            "tag": "green",
                            "weight": 0.1
                        },
                        {
                            "tag": "white",
                            "weight": 1
                        }
                    ]
                },
                "inner_hits": {
                    "tags": {
                        "hits": {
                            "total": 2,
                            "max_score": 1,
                            "hits": [
                                {
                                    "_nested": {
                                        "field": "tags",
                                        "offset": 2
                                    },
                                    "_score": 1,
                                    "_source": {
                                        "tag": "white",
                                        "weight": 1
                                    }
                                },
                                {
                                    "_nested": {
                                        "field": "tags",
                                        "offset": 0
                                    },
                                    "_score": 0.1,
                                    "_source": {
                                        "tag": "red",
                                        "weight": 0.1
                                    }
                                }
                            ]
                        }
                    }
                }
            }
        ]
    }
}
{% endhighlight %}

We can see that in this case there was a single hit (we've only indexed one document, the Fiat 500), and the score for that hit was 0.1 - the weight of the "red" tag.  If we search for "red white", we get a score of 1.1 - the sum of the weights of the `red` tag and the `white` tag.

## Combining queries

We may want to combine different types of query to allow the user to search by a car's name, brand and tags.  There are several options for this depending on your exact use case, such as [bool](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) or [dis_max](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html).
