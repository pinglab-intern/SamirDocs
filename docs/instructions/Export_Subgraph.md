# Export Subgraph of Reactome
- This files generates cypher exports of the neighborhood around each drug found in the reactome database related to oxidative stress.
- Files export to \[Path_To_Local_Graph_DB\]/import/drug_subgraphs/\[Drug Name\].cypher


```python
import neo4j_functions.driver as neo4j_driver
import pandas as pd
import importlib
import progressbar
```


```python
drug_pathway_caseolap = pd.read_csv('output/drug_reactome_pathways_caseolap.csv')
drug_pathway_caseolap.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Drug</th>
      <th>Pathway</th>
      <th>Species</th>
      <th>edgeLength</th>
      <th>IoOS</th>
      <th>OoOS</th>
      <th>RoOS</th>
      <th>drug</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>warfarin [cytosol]</td>
      <td>Chaperone Mediated Autophagy</td>
      <td>Homo sapiens</td>
      <td>2.0</td>
      <td>0.077812</td>
      <td>0.261603</td>
      <td>0.164026</td>
      <td>warfarin</td>
    </tr>
    <tr>
      <td>1</td>
      <td>warfarin [cytosol]</td>
      <td>Pink/Parkin Mediated Mitophagy</td>
      <td>Homo sapiens</td>
      <td>2.0</td>
      <td>0.077812</td>
      <td>0.261603</td>
      <td>0.164026</td>
      <td>warfarin</td>
    </tr>
    <tr>
      <td>2</td>
      <td>warfarin [cytosol]</td>
      <td>Receptor Mediated Mitophagy</td>
      <td>Homo sapiens</td>
      <td>2.0</td>
      <td>0.077812</td>
      <td>0.261603</td>
      <td>0.164026</td>
      <td>warfarin</td>
    </tr>
    <tr>
      <td>3</td>
      <td>warfarin [cytosol]</td>
      <td>Microautophagy</td>
      <td>Homo sapiens</td>
      <td>2.0</td>
      <td>0.077812</td>
      <td>0.261603</td>
      <td>0.164026</td>
      <td>warfarin</td>
    </tr>
    <tr>
      <td>4</td>
      <td>warfarin [cytosol]</td>
      <td>Amplification  of signal from unattached  kine...</td>
      <td>Homo sapiens</td>
      <td>2.0</td>
      <td>0.077812</td>
      <td>0.261603</td>
      <td>0.164026</td>
      <td>warfarin</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Connect to local Reactome
driver = neo4j_driver.driver(uri = "bolt://localhost:7687", user = "neo4j", password = "Akre1234")
```


```python
drugs = drug_pathway_caseolap.Drug.unique()
lowered_drugs = [d.lower() for d in drugs]
query = (
"""
CALL apoc.export.graphml.query(
"MATCH (d:Drug)-[a*1..2]-(b) WHERE toLower(d.displayName) IN %s RETURN d,a,b",
"/all_drugs.graphml",
{readLabels:true, storeNodeIds: true}
)

""" % ("['" + "', '".join(lowered_drugs) + "']")
)
print(query)
driver.run_query(query);
```

    
    CALL apoc.export.graphml.query(
    "MATCH (d:Drug)-[a*1..2]-(b) WHERE toLower(d.displayName) IN ['warfarin [cytosol]', 'rivaroxaban [extracellular region]', 'dabigatran [extracellular region]', 'apixaban [extracellular region]', 'edoxaban [extracellular region]', 'carvedilol [extracellular region]', 'propranolol [extracellular region]', 'metoprolol [extracellular region]', 'sotalol [extracellular region]', 'nebivolol [extracellular region]', 'acebutolol [extracellular region]', 'atenolol [extracellular region]', 'betaxolol [extracellular region]', 'esmolol [extracellular region]', 'labetalol [extracellular region]', 'ticagrelor [extracellular region]', 'ticlopidine [extracellular region]', 'clopidogrel [extracellular region]', 'r-138727 [extracellular region]', 'cangrelor [extracellular region]', 'dobutamine [extracellular region]', 'pindolol [extracellular region]', 'isoprenaline [extracellular region]', 'procainamide [extracellular region]', 'lidocaine [extracellular region]', 'disopyramide [extracellular region]', 'quinidine [extracellular region]', 'phenytoin [extracellular region]', 'mexiletine [extracellular region]', 'tocainide [extracellular region]', 'propafenone [extracellular region]', 'flecainide [extracellular region]', 'dofetilide [extracellular region]', 'ibutilide [extracellular region]', 'amlodipine [extracellular region]', 'diltiazem [extracellular region]', 'isradipine [extracellular region]', 'nifedipine [extracellular region]', 'felodipine [extracellular region]', 'nicardipine [extracellular region]', 'nisoldipine [extracellular region]', 'verapamil [extracellular region]', 'captopril [extracellular region]', 'lisinopril [extracellular region]', 'irbesartan [extracellular region]', 'telmisartan [extracellular region]', 'losartan [extracellular region]', 'olmesartan [extracellular region]', 'candesartan [extracellular region]', 'valsartan [extracellular region]', 'benazepril [endoplasmic reticulum lumen]', 'perindopril [endoplasmic reticulum lumen]', 'ramipril [endoplasmic reticulum lumen]', 'quinapril [endoplasmic reticulum lumen]', 'fosinopril [endoplasmic reticulum lumen]', 'enalapril [endoplasmic reticulum lumen]', 'enoximone [cytosol]', 'milrinone [cytosol]'] RETURN d,a,b",
    "/all_drugs.graphml",
    {readLabels:true, storeNodeIds: true}
    )
    
    



```python

```
