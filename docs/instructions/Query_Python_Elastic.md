
# Querying ElasticSearch in Python

This instruction set requires having a running elasticsearch instance. 

## Running the ElasticSearch Server
To run the ElasticSearch instance in the cloud navigate to the folder ~/software/elasticsearch-6.5.4/bin:  
`cd ~/software/elasticsearch-6.5.4/bin`  

Then type: `./elasticsearch`


## Importing required modules


```python
from elasticsearch import Elasticsearch
from elasticsearch_dsl import Search, Q
```

## Connecting to the ElasticSearch Instance


```python
es = Elasticsearch()
```

## Example search on pubmed article title's


```python
search_term = 'herceptin'

# `hits` variable will store all hits for the search term defined above
hits = Search(
    using=es,
    index="pubmed"
    ).params(
        request_timeout=300
    ).query(
        "match_phrase",
        title=search_term, # title is the defined field to look for search term in, can be modified to abstract/date/etc.
    )

```

## Parsing hits for relevant pubmed articles


```python
for hit in hits:
    print('Title:',hit.title, 'PMID:', hit.pmid)

```

    Title: [Combination therapy herceptin+taxotere/Herceptin+navelbine]. PMID: 23573614
    Title: Herceptin. PMID: 10214591
    Title: [Herceptin]. PMID: 10883160
    Title: Herceptin. PMID: 18071947
    Title: [Herceptin (trastuzumab)]. PMID: 17687197
    Title: Trastuzumab (herceptin). PMID: 21816914
    Title: [Trastuzumab (Herceptin)]. PMID: 11977556
    Title: Development of herceptin. PMID: 15687596
    Title: Subcutaneous herceptin therapy. PMID: 23996323
    Title: Dose scheduling--Herceptin. PMID: 11694785



```python
# Showing available properties in the pubmed hit
dir(hit)
```




    ['MeSH', 'abstract', 'date', 'journal', 'location', 'meta', 'pmid', 'title']


