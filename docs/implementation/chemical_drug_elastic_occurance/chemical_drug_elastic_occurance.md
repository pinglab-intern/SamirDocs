
# Finding Occurrences of Chemicals and Drugs in PubMed Abstracts

Searches the ElasticSearch index created during CaseOLAP pipeline run for curated list of Drugs and Chemicals related to oxidative stress

### Output:
- __Chemical_PMID_occurances.csv__: CSV table where each row is the occurance of a chemical in PubMed
- __Drug_PMID_occurances.csv__: CSV table where each row is the occurance of a drug in PubMed


```python
from elasticsearch import Elasticsearch
from elasticsearch_dsl import Search, Q
import pandas as pd
from itertools import product
import seaborn as sns
import numpy as np
import time
import matplotlib.pyplot as plt
import json
import progressbar
```

## Load Drug and Chemical lists, initialize Elastic Search

- Requires elastic search engine to be running on cluster. Must have PMID index


```python
chemical_list_df = pd.read_csv('input/oxidative_stress_chemicals_SA_10222019.csv')
chemical_list_df['Molecule/Enzyme/Protein'] = chemical_list_df['Molecule/Enzyme/Protein'].str.lower().str.strip()
```


```python
chemical_list_df.head()
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
      <th>Biological Events of Oxidative Stress</th>
      <th>Molecular and Functional Categories</th>
      <th>Molecule/Enzyme/Protein</th>
      <th>MeSH Heading</th>
      <th>MeSH Supplementary</th>
      <th>MeSH tree numbers</th>
      <th>Chemical Formula</th>
      <th>Examples</th>
      <th>Pharm Actions</th>
      <th>Tree Numbers</th>
      <th>References</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Initiation of Oxidative</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>superoxide (anion radical)</td>
      <td>Superoxides</td>
      <td>NaN</td>
      <td>D01.248.497.158.685.750.850; D01.339.431.374.8...</td>
      <td>O2-</td>
      <td>Superoxide, Hydrogen Peroxide</td>
      <td>Oxidants</td>
      <td>D27.720.642,\nD27.888.569.540</td>
      <td>PMID: 25547488</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Initiation of Oxidative</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>hydrogen peroxide</td>
      <td>Hydrogen Peroxide</td>
      <td>NaN</td>
      <td>D01.248.497.158.685.750.424; D01.339.431.374.4...</td>
      <td>H2O2</td>
      <td>NaN</td>
      <td>Anti-Infective Agents, Local</td>
      <td>D27.505.954.122.187</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Initiation of Oxidative</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Oxidants</td>
      <td>D27.720.642,\nD27.888.569.540</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Initiation of Oxidative</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>hydroxyl (radical)</td>
      <td>Hydroxyl Radical</td>
      <td>NaN</td>
      <td>D01.339.431.249; D01.248.497.158.459.300; D01....</td>
      <td>HO</td>
      <td>NaN</td>
      <td>Oxidants</td>
      <td>D27.720.642,\nD27.888.569.540</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Initiation of Oxidative</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>alpha oxygen</td>
      <td>None listed</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
drug_list_df = pd.read_csv('input/drug_list_SA_10222019.csv')
drug_list_df['Name'] = drug_list_df['Name'].str.lower().str.strip()
drug_list_df.head()
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
      <th>Drug Category</th>
      <th>#</th>
      <th>Name</th>
      <th>Synonyms</th>
      <th>MeSH Descriptor</th>
      <th>MeSH tree(s)</th>
      <th>Common adverse effects</th>
      <th>Dosage (freq/amount/time/delivery)</th>
      <th>Duration (time)</th>
      <th>Pharm Action</th>
      <th>...</th>
      <th>Unnamed: 1015</th>
      <th>Unnamed: 1016</th>
      <th>Unnamed: 1017</th>
      <th>Unnamed: 1018</th>
      <th>Unnamed: 1019</th>
      <th>Unnamed: 1020</th>
      <th>Unnamed: 1021</th>
      <th>Unnamed: 1022</th>
      <th>Unnamed: 1023</th>
      <th>Unnamed: 1024</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Anticoagulants</td>
      <td>1.0</td>
      <td>heparin</td>
      <td>['Calciparine', 'Eparina', 'heparina', 'Hepari...</td>
      <td>heparin</td>
      <td>D09.698.373.400</td>
      <td>Thrombocytopenia, Cerebral haemorrhage, Haemog...</td>
      <td>1/18U/kg/iv</td>
      <td>2 days</td>
      <td>Anticoagulants, \nFibrinolytic Agents</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Anticoagulants</td>
      <td>2.0</td>
      <td>warfarin</td>
      <td>['4-Hydroxy-3-(3-oxo-1-phenylbutyl)coumarin', ...</td>
      <td>warfarin</td>
      <td>D03.383.663.283.446.520.914\nD03.633.100.150.4...</td>
      <td>Haemorrhage, Haematoma, anaemia, Epistaxis, hy...</td>
      <td>1/2-10mg/day/po</td>
      <td>As needed</td>
      <td>Anticoagulants, \nRodenticides</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Thrombolytics</td>
      <td>3.0</td>
      <td>streptokinase</td>
      <td>['Streptokinase C precursor']</td>
      <td>streptokinase</td>
      <td>D08.811.277.656.300.775\nD12.776.124.125.662.537</td>
      <td>blurred vision, confusion, dizziness, fever, s...</td>
      <td>1/1,500,000 IU/iv</td>
      <td>60min</td>
      <td>Fibrinolytic Agents</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Thrombolytics</td>
      <td>4.0</td>
      <td>urokinase</td>
      <td>['U-plasminogen activator', 'uPA', 'Urokinase-...</td>
      <td>Urokinase-Type Plasminogen Activator</td>
      <td>D08.811.277.656.300.760.910\nD08.811.277.656.9...</td>
      <td>bleeding gums, coughing up blood, dizziness, h...</td>
      <td>1/4,000,000U/iv</td>
      <td>10min</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Thrombolytics</td>
      <td>5.0</td>
      <td>tpa</td>
      <td>['Alteplasa', 'Alteplase (genetical recombinat...</td>
      <td>Tissue Plasminogen Activator</td>
      <td>D08.811.277.656.300.760.875\nD08.811.277.656.9...</td>
      <td>NaN</td>
      <td>1/0.9mg/kg/iv</td>
      <td>60min</td>
      <td>Fibrinolytic Agents</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 1025 columns</p>
</div>




```python
es = Elasticsearch(timeout=300)

```

## Find PMIDs associated with every combination of drugs and chemicals via elastic search
- output file saved to `output/chem_drug_pair_matches.csv`


```python
# All combinations of drugs and chemicals
drug_chemicals = product(
    drug_list_df['Name'].dropna().unique(),
    chemical_list_df['Molecule/Enzyme/Protein'].dropna().unique()
)

matches = pd.DataFrame()
for (drug, chemical) in progressbar.progressbar(drug_chemicals):

    mol_matches = {
            'PMID': [],
            'title': [],
            'Year': [],
            'Month': []
        }    
    # Match drug and chemical
    q = Q("match_phrase", abstract=drug.lower().strip()) & Q("match_phrase", abstract=chemical.lower().strip())
    
    # Search
    hits = Search(
        using=es,
        index="pubmed"
    ).params(
        request_timeout=300
    ).query(q)
    for h in hits.scan():
        date_dict = json.loads(h.date.replace("'", '"'))        
        mol_matches['PMID'].append(h.pmid)
        mol_matches['title'].append(h.title)
        mol_matches['Year'].append(date_dict['Year'])
        mol_matches['Month'].append(date_dict['Month'])
        
    match_df = pd.DataFrame.from_dict(mol_matches)
    match_df['drug'] = drug.lower().strip()
    match_df['chemical'] = chemical.lower().strip()
    matches = matches.append(match_df)

matches.head()
```

    | |                   #                           | 21059 Elapsed Time: 0:13:02





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
      <th>PMID</th>
      <th>title</th>
      <th>Year</th>
      <th>Month</th>
      <th>drug</th>
      <th>chemical</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>8376590</td>
      <td>Homocysteine, a thrombogenic agent, suppresses...</td>
      <td>1993</td>
      <td>Sep</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25037421</td>
      <td>Degradation of fucoidans from Sargassum fulvel...</td>
      <td>2014</td>
      <td>Oct</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
    </tr>
    <tr>
      <th>2</th>
      <td>14561655</td>
      <td>Role of hydrogen peroxide in sperm capacitatio...</td>
      <td>2004</td>
      <td>Feb</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
    </tr>
    <tr>
      <th>3</th>
      <td>9040037</td>
      <td>Protective effect of dextran sulfate and hepar...</td>
      <td>1997</td>
      <td>Jan</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10547607</td>
      <td>Heparin-binding EGF-like growth factor is expr...</td>
      <td>1999</td>
      <td>Nov</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
    </tr>
  </tbody>
</table>
</div>




```python
matches.to_csv('output/chem_drug_pair_matches.csv', index=False)
```


```python
matches = pd.read_csv('output/chem_drug_pair_matches.csv')
```

### Plotting Drug, chemical co-occurence heatmap
- colors used to show category of drug, and oxidative stress category association of chemicals


```python
chem_name_cats = chemical_list_df[['Molecule/Enzyme/Protein', 'Biological Events of Oxidative Stress']]\
    .drop_duplicates().rename(columns={
        'Molecule/Enzyme/Protein': 'chemical',
        'Biological Events of Oxidative Stress':'chem_cat'
    }).dropna()
drug_name_cats = drug_list_df[['Name', 'Drug Category']]\
    .drop_duplicates().rename(columns={
        'Name': 'drug',
        'Drug Category': 'drug_cat'
    }).dropna()
with_cats_matches = matches.merge(
    chem_name_cats,
    how='left',
    validate='m:m'
)
with_cats_matches = with_cats_matches.merge(
    drug_name_cats,
    how='left',
    validate='m:m'
)
print(matches.dropna().shape, with_cats_matches.dropna().shape)
with_cats_matches.head()
```

    (199479, 6) (204265, 8)





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
      <th>PMID</th>
      <th>title</th>
      <th>Year</th>
      <th>Month</th>
      <th>drug</th>
      <th>chemical</th>
      <th>chem_cat</th>
      <th>drug_cat</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>8376590</td>
      <td>Homocysteine, a thrombogenic agent, suppresses...</td>
      <td>1993.0</td>
      <td>Sep</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
      <td>Initiation of Oxidative</td>
      <td>Anticoagulants</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25037421</td>
      <td>Degradation of fucoidans from Sargassum fulvel...</td>
      <td>2014.0</td>
      <td>Oct</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
      <td>Initiation of Oxidative</td>
      <td>Anticoagulants</td>
    </tr>
    <tr>
      <th>2</th>
      <td>14561655</td>
      <td>Role of hydrogen peroxide in sperm capacitatio...</td>
      <td>2004.0</td>
      <td>Feb</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
      <td>Initiation of Oxidative</td>
      <td>Anticoagulants</td>
    </tr>
    <tr>
      <th>3</th>
      <td>9040037</td>
      <td>Protective effect of dextran sulfate and hepar...</td>
      <td>1997.0</td>
      <td>Jan</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
      <td>Initiation of Oxidative</td>
      <td>Anticoagulants</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10547607</td>
      <td>Heparin-binding EGF-like growth factor is expr...</td>
      <td>1999.0</td>
      <td>Nov</td>
      <td>heparin</td>
      <td>hydrogen peroxide</td>
      <td>Initiation of Oxidative</td>
      <td>Anticoagulants</td>
    </tr>
  </tbody>
</table>
</div>




```python
with_cats_matches.chem_cat.unique()
```




    array(['Initiation of Oxidative', 'Outcomes of Oxidative Stress',
           'Regulation of Oxidative Stress'], dtype=object)




```python
# Creating color palettes to label drug and chemical categories
chems = with_cats_matches.chem_cat.unique()
chem_pal = sns.color_palette("hls", n_colors=with_cats_matches.chem_cat.nunique())
chem_pal_dict = dict(zip(chems, chem_pal))

drugs = with_cats_matches.drug_cat.unique()
drug_pal = sns.color_palette("tab20c", n_colors=with_cats_matches.drug_cat.nunique())
drug_pal[-5:] = sns.color_palette("tab20b", n_colors=5)
drug_pal_dict = dict(zip(drugs, drug_pal))

with_cats_matches['chem_color'] = with_cats_matches.chem_cat.map(chem_pal_dict)
with_cats_matches['drug_color'] = with_cats_matches.drug_cat.map(drug_pal_dict)

sns.palplot(chem_pal)
plt.gca().set_xticklabels(chems)
plt.xticks(rotation=60)
sns.palplot(drug_pal)
plt.gca().set_xticklabels(drugs)
plt.xticks(rotation=90);
```


![png](output_14_0.png)



![png](output_14_1.png)



```python
## Save Color Palettes
with open('drug_cat_palette.json', 'w') as fp:
    json.dump(drug_pal_dict, fp)

with open('chem_cat_palette.json', 'w') as fp:
    json.dump(chem_pal_dict, fp)
```


```python
# Set NaN category color to white
with_cats_matches.loc[with_cats_matches.drug_color.isna(), 'drug_color'] = "white"
```


```python
# Count articles per drug-chemical co-occurrence
article_count = pd.DataFrame(
    with_cats_matches.groupby(['drug', 'chemical', 'drug_cat', 'chem_cat']).PMID.nunique()
).reset_index().rename(columns={'PMID': 'Article Count'})
article_count['log_count'] = np.log10(article_count['Article Count'])

chem_colors_df = with_cats_matches[['chemical', 'chem_color']].drop_duplicates()
chem_colors = [chem_colors_df[chem_colors_df.chemical == chem].chem_color.unique()[0] for chem in piv_count.index]

drug_colors_df = with_cats_matches[['drug', 'drug_color']].drop_duplicates()
drug_colors = [drug_colors_df[drug_colors_df.drug == drug].drug_color.unique()[0] for drug in piv_count.columns]

piv_count = article_count.pivot_table(
    index='chemical',
    columns='drug',
    values='log_count',
    fill_value=0
)
sns.clustermap(
    piv_count,
    figsize=(22,13),
    cmap='viridis',
    row_colors=chem_colors,
    col_colors=drug_colors
)

```




    <seaborn.matrix.ClusterGrid at 0x7ff7afcb4898>




![png](output_17_1.png)


## Find PMIDS assocaited with drugs via elastic search
- Searches abstracts for drug names or synonyms of drug names
- Finds number of occurances of drug name or synonyms in abstract

Saves to `output/Drug_PMID_occurances.csv`


```python
drug_matches = pd.DataFrame()
tot = drug_list_df.Name.nunique()

for (drug, synonyms, category), m_df in progressbar.progressbar(drug_list_df.groupby(['Name', 'Synonyms', 'Drug Category'])):
    drug_match = {
            'PMID': [],
            'title': [],
            'MeSH': [],
            'count': [],
            'Year': [],
            'Month': []
        }
    synonyms = synonyms.split(', ')
    drug = drug.lower()
    q = Q('match_phrase', abstract=drug)

    if synonyms:
        synonyms = [s.lower() for s in synonyms]
        for s in synonyms:
            q = q | Q('match_phrase', abstract=s)
    
    hits = Search(
        using=es,
        index="pubmed"
    ).query(q)

    for h in hits.scan():
        date_dict = json.loads(h.date.replace("'", '"'))        
        drug_match['PMID'].append(h.pmid)
        drug_match['title'].append(h.title)
        drug_match['MeSH'].append(h.MeSH)
        drug_match['Year'].append(date_dict['Year'])
        drug_match['Month'].append(date_dict['Month'])
        
        entity_count = 0
        for phrase in [drug] + synonyms:
            entity_lower = phrase.lower().replace("-", " ")
            entity_count += abs_lower.count(entity_lower)
        
        drug_match['count'].append(entity_count)
    
    drug_match_df = pd.DataFrame.from_dict(drug_match)
    drug_match_df['drug'] = drug
    drug_match_df['category'] = category
    drug_matches = drug_matches.append(drug_match_df)


drug_matches.head()
```

    100% (161 of 161) |######################| Elapsed Time: 0:37:59 Time:  0:37:59





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
      <th>PMID</th>
      <th>title</th>
      <th>MeSH</th>
      <th>count</th>
      <th>Year</th>
      <th>Month</th>
      <th>drug</th>
      <th>category</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>24853116</td>
      <td>Acarbose monotherapy and weight loss in Easter...</td>
      <td>[Acarbose, therapeutic use, Asian Continental ...</td>
      <td>0</td>
      <td>2014</td>
      <td>Nov</td>
      <td>acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
    </tr>
    <tr>
      <th>1</th>
      <td>24863354</td>
      <td>Comparative evaluation of polysaccharides isol...</td>
      <td>[Asteraceae, chemistry, Astragalus Plant, chem...</td>
      <td>0</td>
      <td>2014</td>
      <td>Apr</td>
      <td>acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
    </tr>
    <tr>
      <th>2</th>
      <td>24866329</td>
      <td>Effects of sitagliptin or mitiglinide as an ad...</td>
      <td>[Acarbose, therapeutic use, Aged, Asian Contin...</td>
      <td>0</td>
      <td>2014</td>
      <td>Jul</td>
      <td>acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
    </tr>
    <tr>
      <th>3</th>
      <td>12918894</td>
      <td>Nateglinide (Starlix): update on a new antidia...</td>
      <td>[Blood Glucose, physiology, Cyclohexanes, phar...</td>
      <td>0</td>
      <td></td>
      <td></td>
      <td>acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
    </tr>
    <tr>
      <th>4</th>
      <td>20568489</td>
      <td>Digoxin: serious drug interactions.</td>
      <td>[Digoxin, adverse effects, blood, Drug Interac...</td>
      <td>0</td>
      <td>2010</td>
      <td>Apr</td>
      <td>acarbose</td>
      <td>Alpha-glucosidase Inhibitors</td>
    </tr>
  </tbody>
</table>
</div>




```python
drug_matches.to_csv('output/Drug_PMID_occurances.csv', index=False)
drug_matches.shape
```




    (2702853, 8)




```python
drug_category_PMID_count = pd.DataFrame(drug_matches.groupby('category').PMID.nunique()).reset_index()
drug_category_PMID_count
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
      <th>category</th>
      <th>PMID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>ACE Inhibitors</td>
      <td>70299</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Alpha-glucosidase Inhibitors</td>
      <td>2008</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Angiotensin II Antagonists</td>
      <td>16230</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Anticoagulants</td>
      <td>80628</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Antiplatelets</td>
      <td>82006</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Beta Blockers</td>
      <td>949488</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Bile Acid Resins</td>
      <td>2459</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Calcium Antagonist</td>
      <td>47735</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Calcium Channel Blockers</td>
      <td>25446</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Cholesteron Absorption Blocker</td>
      <td>2554</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Diuretics</td>
      <td>28829</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Fibrates</td>
      <td>12214</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Glucagon-like peptide-1 blockers</td>
      <td>5180</td>
    </tr>
    <tr>
      <th>13</th>
      <td>HMG-CoA Reductase inhibitors (Statins)</td>
      <td>22166</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Inotropes</td>
      <td>246382</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Insulin</td>
      <td>7386</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Metformin</td>
      <td>16564</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Na Channel Blockers</td>
      <td>42512</td>
    </tr>
    <tr>
      <th>18</th>
      <td>Other Anti Arrhythmics</td>
      <td>130224</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Phosphodiesterase Inhbitors</td>
      <td>21894</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Potassium Channel Blockers</td>
      <td>10292</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Sulfonylureas</td>
      <td>12331</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Thiazolidinediones</td>
      <td>5089</td>
    </tr>
    <tr>
      <th>23</th>
      <td>Thrombolytics</td>
      <td>309306</td>
    </tr>
    <tr>
      <th>24</th>
      <td>Vasodilators</td>
      <td>408803</td>
    </tr>
    <tr>
      <th>25</th>
      <td>Vasopressin Antagonists</td>
      <td>850</td>
    </tr>
  </tbody>
</table>
</div>



## Searching for PMIDs associated with each chemical
- If there is a MeSH id, searches pubmed index for abstract containing drug name OR MeSH terms containin mesh term 
- If there is no MeSH id, only searches for drug name in abstract

saves to `output/Chemical_PMID_occurances.csv`


```python
has_data_df = chemical_list_df[
    (~chemical_list_df['Molecule/Enzyme/Protein'].isnull()) |
    (~chemical_list_df['MeSH Heading'].isnull())
]
chem_matches_df = pd.DataFrame()

for (name, mesh, category), m_df in progressbar.progressbar(has_data_df.groupby(['Molecule/Enzyme/Protein', 'MeSH Heading', 'Biological Events of Oxidative Stress'])):
    hit_dict = {
        'PMID': [],
        'Article MeSH': [],
        'Year': [],
        'Month': [],
    }
    if mesh.lower() == 'none listed':
        q = Q('match_phrase', abstract=name.lower())
    else:
        q = Q('match_phrase', abstract=name.lower()) | Q('match_phrase', MeSH=mesh)
    
    hits = Search(
        using=es,
        index="pubmed"
    ).params(
        request_timeout=300
    ).query(q)
    for h in hits.scan():
        date_dict = json.loads(h.date.replace("'", '"'))
        hit_dict['PMID'].append(h.pmid)
        hit_dict['Article MeSH'].append(h.MeSH)
        hit_dict['Year'].append(date_dict['Year'])
        hit_dict['Month'].append(date_dict['Month'])
    
    
    hit_df = pd.DataFrame.from_dict(hit_dict)
    hit_df['category'] = category
    hit_df['chemical'] = name
    hit_df['MeSH'] = mesh
    chem_matches_df = chem_matches_df.append(hit_df)

chem_matches_df.head()
```

    100% (157 of 157) |######################| Elapsed Time: 0:44:52 Time:  0:44:52





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
      <th>PMID</th>
      <th>Article MeSH</th>
      <th>Year</th>
      <th>Month</th>
      <th>category</th>
      <th>chemical</th>
      <th>MeSH</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>24852702</td>
      <td>[Alcohols, metabolism, toxicity, Aldehydes, me...</td>
      <td>2014</td>
      <td>Sep</td>
      <td>Initiation of Oxidative</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>Aldehydes</td>
    </tr>
    <tr>
      <th>1</th>
      <td>24854020</td>
      <td>[Adult, Aldehydes, metabolism, Case-Control St...</td>
      <td>2015</td>
      <td>Apr</td>
      <td>Initiation of Oxidative</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>Aldehydes</td>
    </tr>
    <tr>
      <th>2</th>
      <td>24854122</td>
      <td>[Acetylcysteine, pharmacology, Aldehydes, phar...</td>
      <td>2014</td>
      <td>Nov</td>
      <td>Initiation of Oxidative</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>Aldehydes</td>
    </tr>
    <tr>
      <th>3</th>
      <td>24877583</td>
      <td>[4-Butyrolactone, chemistry, Aldehydes, chemis...</td>
      <td>2014</td>
      <td>Jun</td>
      <td>Initiation of Oxidative</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>Aldehydes</td>
    </tr>
    <tr>
      <th>4</th>
      <td>24878441</td>
      <td>[Absorption, Physicochemical, Acetonitriles, c...</td>
      <td>2014</td>
      <td>Nov</td>
      <td>Initiation of Oxidative</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
      <td>Aldehydes</td>
    </tr>
  </tbody>
</table>
</div>




```python
chem_matches_df.to_csv('output/Chemical_PMID_occurances.csv', index=False)
chem_matches_df.shape
```




    (3291433, 7)


