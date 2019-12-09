# Create a subgraph from exports made on the Reactome graph Database
- This section relies on having exported a subgraph of all desired drugs to a file called `all_drugs.graphml`
- File is created in the `Export_Subgraph.ipynb`
- Manually must move the file from the reactome graphdb to the import folder of a new graphdb


__Note:__ Must have started running a new empty neo4j graph database to succesfully connect


```python
import neo4j_functions.driver as neo4j_driver
import pandas as pd
import importlib
import progressbar
import os
```


```python
# Connect to new empty local database
driver = neo4j_driver.driver(uri = "bolt://localhost:7687", user = "neo4j", password = "subgraph1234")
```

If you encounter the error `java.lang.OutOfMemoryError: Java heap space` try going in to the neo4j.conf file and removing/commenting out the lines:
```
dbms.memory.heap.initial_size=512m
dbms.memory.heap.max_size=1G
```


```python
query = (
"""
CALL apoc.import.graphml('file:/all_drugs.graphml', {readLabels:true, storeNodeIds:true});
""")
driver.run_query(query)

```




    <neo4j.BoltStatementResult at 0x11e2dfb38>




```python

```
