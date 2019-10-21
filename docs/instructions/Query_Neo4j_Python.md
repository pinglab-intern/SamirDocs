# Querying a Neo4j Instance in Python

This instruction set requires having a running Neo4j instance. 

## Running Neo4j
Neo4j is most easily run locally using the desktop GUI version. Alternatively from the command line it can be run with `neo4j start`

## Importing required modules


```python
from neo4j import GraphDatabase
```

## Initializing Driver and Creating Cypher Query

The driver uses the neo4j-driver python package to connect via neo4j's api to my locally running Reactome Database. 

Queries are written the the Cypher query language. A good cheatsheet for commands can be found [here](https://neo4j.com/docs/cypher-refcard/current/)


```python
# Initialize connection to database
driver = GraphDatabase.driver('bolt://localhost:7687', auth=('neo4j', 'Akre1234'))
```


```python
query = 'MATCH (a:Drug) RETURN a.name, a.stId LIMIT 5'
```

## Run Cypher query and print desired results

The query returned name and stId properties of 5 drugs. Each returned record will only contain the specified information form the query.   


```python
with driver.session() as session:
    info = session.run(query)
    for item in info:
        print('Name:', item.values()[0], 'id:', item.values()[1])
```

    Name: ['trastuzumab', 'herceptin', 'D5v8', 'R-597'] id: R-ALL-9634466
    Name: ['CP-724714'] id: R-ALL-9649889
    Name: ['Afatinib', 'BIBW2992', 'Irreversible TKI inhibitor afatinib generic inhibits  EGFR and  ERBB2 (HER2)'] id: R-ALL-1220577
    Name: ['AZ5104'] id: R-ALL-9649879
    Name: ['Sapitinib'] id: R-ALL-9649894

