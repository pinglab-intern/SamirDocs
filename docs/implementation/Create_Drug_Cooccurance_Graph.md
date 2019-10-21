# Creating Co-Occurence Graph of Drugs-Chemicals-PMIDs
This notebook creates a graph database storing the relation of drugs and 
chemicals related to oxidative stress and their occurance in PubMed abstracts

Nodes are created for drugs, chemicals, articles, and MeSH terms.


```python
import neo4j_functions.driver as neo4j_driver
import pandas as pd
import importlib
```


```python
drug_list_df = pd.read_csv('lib/Drug list total 04.05.19   - Overview Drug list.csv')
drug_occurance_df = pd.read_csv('lib/Drug_PMID_occurances.csv')

chemical_list_df = pd.read_csv('lib/Oxidative Stress Text Mining Targets 4.1 - Summary of Oxidative Stress.csv')
chemical_occurance_df = pd.read_csv('lib/Chemical_PMID_occurances.csv')
```

## Merging Drug List with Drug Occurance Data Sets

1. Duplicate drug names in the lab provided list are merged, the drug with an associated category is kept if possible
2. Deduplicated list of drug names is merged with a dataframe for drug occurance generated on the CaseOLAP cloud instance
    - Notebook used to generate drug occurance list located at `/home/ubuntu/RotationStd/elasticsearch/chemical_drug_elastic_occurance.ipynb`
3. Final merged dataframe saved in to `import` folder of neo4j instance


```python
# Removing Duplicate drug names, keeping version with a drug category if possible
deduped_drug_list = drug_list_df.sort_values(by='Drug Category').drop_duplicates(subset=['Name'], keep='first')
```


```python
# Merging drug list with drug occurance list
drug_occurance_df['MeSH'] = drug_occurance_df['MeSH'].str.replace('[', '').str.replace(']', '').str.replace("'", '')

drug_list_occurance_df = drug_occurance_df.merge(
    deduped_drug_list.rename(columns={
        'Name': 'drug',
        'Drug Category': 'category',
        'MeSH Descriptor': 'drug_mesh',
    }),
    how='inner',
    validate='m:1'
)
# Values with NaN for category or synonym replaced
# NaN for synonym replaced with drug name, category replaced with 'None'
drug_list_occurance_df['drug']  = drug_list_occurance_df['drug'].str.strip()
drug_list_occurance_df.loc[drug_list_occurance_df.MeSH == '', 'MeSH'] = 'None'

drug_list_occurance_df.loc[drug_list_occurance_df.category.isnull(), 'category']  = 'None'
drug_list_occurance_df.loc[drug_list_occurance_df.drug_mesh.isnull(), 'drug_mesh']  = 'None'
drug_list_occurance_df.loc[drug_list_occurance_df.Synonyms.isnull(), 'Synonyms']  = drug_list_occurance_df[drug_list_occurance_df.Synonyms.isnull()].drug
```


```python
# Saving file to import area of local neo4j instance
drug_list_occurance_file = '/Users/akre96/Library/Application Support/Neo4j Desktop/Application/neo4jDatabases/database-dc2bbd3b-84e9-421e-8594-9fe29be9bb02/installation-3.5.6/import/drug_list_occurance.csv'
drug_list_occurance_df.to_csv(drug_list_occurance_file, index=False)
drug_list_occurance_df.head()
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
      <th>MeSH</th>
      <th>PMID</th>
      <th>abstract</th>
      <th>title</th>
      <th>drug</th>
      <th>category</th>
      <th>#</th>
      <th>Synonyms</th>
      <th>drug_mesh</th>
      <th>MeSH tree(s)</th>
      <th>Common adverse effects</th>
      <th>Dosage (freq/amount/time/delivery)</th>
      <th>Duration (time)</th>
      <th>Pham Action</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>Actinomycetales, chemistry, enzymology, Adenos...</td>
      <td>8784428</td>
      <td>a phosphotransferase which modifies the alpha ...</td>
      <td>Acarbose 7-phosphotransferase from Actinoplane...</td>
      <td>Acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>54</td>
      <td>Acarbosa, Acarbose, Acarbosum</td>
      <td>Acarbose</td>
      <td>D09.698.629.802.100</td>
      <td>Hypoglycaemia, Hypoglycaemic \ncoma, pneumatos...</td>
      <td>3/25-50-100mg/day/po</td>
      <td>4-8 weeks intervals</td>
      <td>Glycoside \nHydrolase Inhibitors</td>
    </tr>
    <tr>
      <td>1</td>
      <td>Acarbose, Adult, Blood Glucose, metabolism, Cl...</td>
      <td>6350115</td>
      <td>in a double blind study we have compared the e...</td>
      <td>Effect of acarbose, pectin, a combination of a...</td>
      <td>Acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>54</td>
      <td>Acarbosa, Acarbose, Acarbosum</td>
      <td>Acarbose</td>
      <td>D09.698.629.802.100</td>
      <td>Hypoglycaemia, Hypoglycaemic \ncoma, pneumatos...</td>
      <td>3/25-50-100mg/day/po</td>
      <td>4-8 weeks intervals</td>
      <td>Glycoside \nHydrolase Inhibitors</td>
    </tr>
    <tr>
      <td>2</td>
      <td>Acarbose, Adult, Aged, Blood Glucose, metaboli...</td>
      <td>9663365</td>
      <td>acarbose is an alpha glucosidase inhibitor app...</td>
      <td>Effects of beano on the tolerability and pharm...</td>
      <td>Acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>54</td>
      <td>Acarbosa, Acarbose, Acarbosum</td>
      <td>Acarbose</td>
      <td>D09.698.629.802.100</td>
      <td>Hypoglycaemia, Hypoglycaemic \ncoma, pneumatos...</td>
      <td>3/25-50-100mg/day/po</td>
      <td>4-8 weeks intervals</td>
      <td>Glycoside \nHydrolase Inhibitors</td>
    </tr>
    <tr>
      <td>3</td>
      <td>Acarbose, administration &amp; dosage, Animals, Bo...</td>
      <td>11779583</td>
      <td>as alpha glucosidase inhibitor, the antidiabet...</td>
      <td>Chronic acarbose-feeding increases GLUT1 prote...</td>
      <td>Acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>54</td>
      <td>Acarbosa, Acarbose, Acarbosum</td>
      <td>Acarbose</td>
      <td>D09.698.629.802.100</td>
      <td>Hypoglycaemia, Hypoglycaemic \ncoma, pneumatos...</td>
      <td>3/25-50-100mg/day/po</td>
      <td>4-8 weeks intervals</td>
      <td>Glycoside \nHydrolase Inhibitors</td>
    </tr>
    <tr>
      <td>4</td>
      <td>Acarbose, Aged, Blood Glucose, metabolism, Dia...</td>
      <td>9428831</td>
      <td>to compare the therapeutic potential of acarbo...</td>
      <td>Efficacy of 24-week monotherapy with acarbose,...</td>
      <td>Acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>54</td>
      <td>Acarbosa, Acarbose, Acarbosum</td>
      <td>Acarbose</td>
      <td>D09.698.629.802.100</td>
      <td>Hypoglycaemia, Hypoglycaemic \ncoma, pneumatos...</td>
      <td>3/25-50-100mg/day/po</td>
      <td>4-8 weeks intervals</td>
      <td>Glycoside \nHydrolase Inhibitors</td>
    </tr>
  </tbody>
</table>
</div>



## Creating Neo4J Graph Database for Drug occurance in PMIDs
1. Neo4J Driver initialized
2. Query formed to import data from list generated in previous section of this notebook
    - Loading csv
    - Creating drug entities with name, category, and synonym fields
    - Creating article entities with PMID, abstract, title, and MeSH fields
    - Creating edges labeled OCCURANCE for connecting drugs referenced by a PMID


```python
importlib.reload(neo4j_driver)
driver = neo4j_driver.driver(uri = "bolt://localhost:7687", user = "neo4j", password = "drug1234")
```


```python
import_data_query = (
    "LOAD CSV WITH HEADERS FROM %s AS row"
    " MERGE (drug:Drug {name: row.drug, category: row.category, synonyms: row.Synonyms})"
    " MERGE (article:Article {PMID: row.PMID, abstract: row.abstract, title: row.title, MeSH: split(row.MeSH, %s)})"
    " MERGE (drug)-[:OCCURANCE]->(article)"
    " MERGE (drugmesh:MeSH {name: row.drug_mesh})"
    " MERGE (drug)-[:OCCURANCE]->(article)"
    " MERGE (drug)-[:HAS_MESH]->(drugmesh)"
    % ('"file:///' + 'drug_list_occurance.csv' + '"', "', '")
)
print('Query:\n\t', import_data_query)
with driver.driver.session() as session:
    result = session.run(import_data_query)
```

    Query:
    	 LOAD CSV WITH HEADERS FROM "file:///drug_list_occurance.csv" AS row MERGE (drug:Drug {name: row.drug, category: row.category, synonyms: row.Synonyms}) MERGE (article:Article {PMID: row.PMID, abstract: row.abstract, title: row.title, MeSH: split(row.MeSH, ', ')}) MERGE (drug)-[:OCCURANCE]->(article) MERGE (drugmesh:MeSH {name: row.drug_mesh}) MERGE (drug)-[:OCCURANCE]->(article) MERGE (drug)-[:HAS_MESH]->(drugmesh)


## Merging Chemical List with Chemical Occurance Data Sets


```python
deduped_chem_list = chemical_list_df\
    .dropna(subset=['Molecule/Enzyme/Protein'])\
    .sort_values(by='Molecular and Functional Categories')\
    .drop_duplicates(subset=['Molecule/Enzyme/Protein'], keep='first')\
    .fillna('None')
```


```python
chemical_occurance_df['MeSH'] = chemical_occurance_df['MeSH'].str.replace('[', '').str.replace(']', '').str.replace("'", '')
chem_list_occurance_df = chemical_occurance_df.merge(
    deduped_chem_list.rename(columns={
        'Molecule/Enzyme/Protein': 'chemical',
        'Chemical Formula': 'formula',
        'Molecular and Functional Categories': 'GO_MF',
        'Biological Events of Oxidative Stress': 'GO_Oxidative_Stress',
        'MeSH Heading': 'chemical_mesh'
    }),
    how='inner',
    validate='m:1'
).fillna('None')
chem_list_occurance_df['chemical'] = chem_list_occurance_df.chemical.str.strip()
chem_list_occurance_df.loc[chem_list_occurance_df.MeSH == '', 'MeSH'] = 'None'
chem_list_occurance_df.head()

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
      <th>MeSH</th>
      <th>PMID</th>
      <th>abstract</th>
      <th>title</th>
      <th>chemical</th>
      <th>GO_Oxidative_Stress</th>
      <th>GO_MF</th>
      <th>chemical_mesh</th>
      <th>MeSH Supplementary</th>
      <th>MeSH tree numbers</th>
      <th>formula</th>
      <th>Examples</th>
      <th>Pharm Actions</th>
      <th>Tree Numbers</th>
      <th>References</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>None</td>
      <td>31368101</td>
      <td>coronary spasm plays an important role in the ...</td>
      <td>Association of East Asian Variant Aldehyde Deh...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>135</td>
      <td>Lipid Peroxidation Products</td>
      <td>Aldehydes</td>
      <td>4-hydroxy-2-nonenal</td>
      <td>D02.047</td>
      <td>C9H16O2</td>
      <td>4-HNE, MDA</td>
      <td>Cross-Linking Reagents</td>
      <td>D27.720.470.410.210</td>
      <td>None</td>
    </tr>
    <tr>
      <td>1</td>
      <td>Acetylcholinesterase, metabolism, Aldehydes, m...</td>
      <td>10463393</td>
      <td>we have investigated the effect of soman induc...</td>
      <td>Increased levels of nitrogen oxides and lipid ...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>135</td>
      <td>Lipid Peroxidation Products</td>
      <td>Aldehydes</td>
      <td>4-hydroxy-2-nonenal</td>
      <td>D02.047</td>
      <td>C9H16O2</td>
      <td>4-HNE, MDA</td>
      <td>Cross-Linking Reagents</td>
      <td>D27.720.470.410.210</td>
      <td>None</td>
    </tr>
    <tr>
      <td>2</td>
      <td>Aldehydes, chemistry, Amines, chemistry, Benzy...</td>
      <td>8448343</td>
      <td>the reaction of trans 4 hydroxy 2 nonenal (4 h...</td>
      <td>Pyrrole formation from 4-hydroxynonenal and pr...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>135</td>
      <td>Lipid Peroxidation Products</td>
      <td>Aldehydes</td>
      <td>4-hydroxy-2-nonenal</td>
      <td>D02.047</td>
      <td>C9H16O2</td>
      <td>4-HNE, MDA</td>
      <td>Cross-Linking Reagents</td>
      <td>D27.720.470.410.210</td>
      <td>None</td>
    </tr>
    <tr>
      <td>3</td>
      <td>Animals, Blood-Brain Barrier, metabolism, path...</td>
      <td>29775963</td>
      <td>brain ischemic preconditioning (ipc) with mild...</td>
      <td>Brain ischemic preconditioning protects agains...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>135</td>
      <td>Lipid Peroxidation Products</td>
      <td>Aldehydes</td>
      <td>4-hydroxy-2-nonenal</td>
      <td>D02.047</td>
      <td>C9H16O2</td>
      <td>4-HNE, MDA</td>
      <td>Cross-Linking Reagents</td>
      <td>D27.720.470.410.210</td>
      <td>None</td>
    </tr>
    <tr>
      <td>4</td>
      <td>Alzheimer Disease, drug therapy, enzymology, p...</td>
      <td>30218858</td>
      <td>excessive production of amyloid β (aβ) induced...</td>
      <td>Neuro-protective effects of aloperine in an Al...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>135</td>
      <td>Lipid Peroxidation Products</td>
      <td>Aldehydes</td>
      <td>4-hydroxy-2-nonenal</td>
      <td>D02.047</td>
      <td>C9H16O2</td>
      <td>4-HNE, MDA</td>
      <td>Cross-Linking Reagents</td>
      <td>D27.720.470.410.210</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Saving file to import area of local neo4j instance
chem_list_occurance_file = '/Users/akre96/Library/Application Support/Neo4j Desktop/Application/neo4jDatabases/database-dc2bbd3b-84e9-421e-8594-9fe29be9bb02/installation-3.5.6/import/chem_list_occurance.csv'
chem_list_occurance_df.to_csv(chem_list_occurance_file, index=False)
```

## Adding to Neo4J Graph Database for Chemical occurance in PMIDs
1. Query formed to import data from list generated in previous section of this notebook
    - Loading csv
    - Creating chemical entities with name, example, and formula fields
    - Merges article entities with PMID, abstract, title, and MeSH fields
    - Creating edges labeled OCCURANCE for connecting drugs referenced by a PMID


```python
import_chemical_data_query = (
    "LOAD CSV WITH HEADERS FROM %s AS row"
    " MERGE (chem:Chemical {name: row.chemical, example: row.Examples, formula: row.formula})"
    " MERGE (article:Article {PMID: row.PMID, abstract: row.abstract, title: row.title, MeSH: split(row.MeSH, %s)})"
    " MERGE (mesh:MeSH {name: row.chemical_mesh})"
    " MERGE (chem)-[:OCCURANCE]->(article)"
    " MERGE (chem)-[:HAS_MESH]->(mesh)"
    % ('"file:///' + 'chem_list_occurance.csv' + '"', "', '")

)
print('Query:\n\t', import_chemical_data_query)
with driver.driver.session() as session:
    result = session.run(import_chemical_data_query)
```

    Query:
    	 LOAD CSV WITH HEADERS FROM "file:///chem_list_occurance.csv" AS row MERGE (chem:Chemical {name: row.chemical, example: row.Examples, formula: row.formula}) MERGE (article:Article {PMID: row.PMID, abstract: row.abstract, title: row.title, MeSH: split(row.MeSH, ', ')}) MERGE (mesh:MeSH {name: row.chemical_mesh}) MERGE (chem)-[:OCCURANCE]->(article) MERGE (chem)-[:HAS_MESH]->(mesh)


## Adding MeSH descriptors from Articles as MeSH node
1. Adds nodes from mesh descriptor list
2. Deletes "none" node


```python
article_mesh_descriptors_query = (
    "MATCH (article:Article)"
    " UNWIND article.MeSH AS m"
    " MERGE (artMesh:MeSH {name: m})"
    " MERGE (article)-[:HAS_MESH]->(artMesh)"
)
print('Query:\n\t', article_mesh_descriptors_query)
with driver.driver.session() as session:
    result = session.run(article_mesh_descriptors_query)
```

    Query:
    	 MATCH (article:Article) UNWIND article.MeSH AS m MERGE (artMesh:MeSH {name: m}) MERGE (article)-[:HAS_MESH]->(artMesh)



```python
delete_none_mesh_descriptors_query = (
    "MATCH (m:MeSH {name: 'None'})"
    " DETACH DELETE m"
)
print('Query:\n\t', delete_none_mesh_descriptors_query)
with driver.driver.session() as session:
    result = session.run(delete_none_mesh_descriptors_query)
```

    Query:
    	 MATCH (m:MeSH {name: 'None'}) DETACH DELETE m



```python

```
