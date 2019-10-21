
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
```

## Load Drug and Chemical lists, initialize Elastic Search

- Requires elastic search engine to be running on cluster. Must have PMID index


```python
chemical_list_df = pd.read_csv('Oxidative Stress Text Mining Targets 4.1 - Summary of Oxidative Stress.csv')
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
      <td>Initiation of Oxidative  1</td>
      <td>Reactive Oxygen Species (ROS)</td>
      <td>Superoxide (anion radical)</td>
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
      <td>2</td>
      <td>NaN</td>
      <td>Hydrogen Peroxide</td>
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
      <td>NaN</td>
      <td>NaN</td>
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
      <td>3</td>
      <td>NaN</td>
      <td>Hydroxyl (radical)</td>
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
      <td>4</td>
      <td>NaN</td>
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
drug_list_df = pd.read_csv('Drug list total 04.05.19   - Overview Drug list.csv')
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
      <th>Pham Action</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Anticoagulants</td>
      <td>1</td>
      <td>heparin</td>
      <td>['Calciparine', 'Eparina', 'heparina', 'Hepari...</td>
      <td>heparin</td>
      <td>D09.698.373.400</td>
      <td>Thrombocytopenia, Cerebral haemorrhage, Haemog...</td>
      <td>1/18U/kg/iv</td>
      <td>2 days</td>
      <td>Anticoagulants, \nFibrinolytic Agents</td>
    </tr>
    <tr>
      <th>1</th>
      <td>NaN</td>
      <td>2</td>
      <td>warfarin</td>
      <td>['4-Hydroxy-3-(3-oxo-1-phenylbutyl)coumarin', ...</td>
      <td>warfarin</td>
      <td>D03.383.663.283.446.520.914\nD03.633.100.150.4...</td>
      <td>Haemorrhage, Haematoma, anaemia, Epistaxis, hy...</td>
      <td>1/2-10mg/day/po</td>
      <td>As needed</td>
      <td>Anticoagulants, \nRodenticides</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Thrombolytics</td>
      <td>3</td>
      <td>streptokinase</td>
      <td>['Streptokinase C precursor']</td>
      <td>streptokinase</td>
      <td>D08.811.277.656.300.775\nD12.776.124.125.662.537</td>
      <td>blurred vision, confusion, dizziness, fever, s...</td>
      <td>1/1,500,000 IU/iv</td>
      <td>60min</td>
      <td>Fibrinolytic Agents</td>
    </tr>
    <tr>
      <th>3</th>
      <td>NaN</td>
      <td>4</td>
      <td>urokinase</td>
      <td>['U-plasminogen activator', 'uPA', 'Urokinase-...</td>
      <td>Urokinase-Type Plasminogen Activator</td>
      <td>D08.811.277.656.300.760.910\nD08.811.277.656.9...</td>
      <td>bleeding gums, coughing up blood, dizziness, h...</td>
      <td>1/4,000,000U/iv</td>
      <td>10min</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>NaN</td>
      <td>5</td>
      <td>tpa</td>
      <td>['Alteplasa', 'Alteplase (genetical recombinat...</td>
      <td>Tissue Plasminogen Activator</td>
      <td>D08.811.277.656.300.760.875\nD08.811.277.656.9...</td>
      <td>NaN</td>
      <td>1/0.9mg/kg/iv</td>
      <td>60min</td>
      <td>Fibrinolytic Agents</td>
    </tr>
  </tbody>
</table>
</div>




```python
es = Elasticsearch()

```

## Find PMIDs associated with every combination of drugs and chemicals via elastic search


```python
# All combinations of drugs and chemicals
drug_chemicals = product(
    drug_list_df['Name'].dropna().unique(),
    chemical_list_df['Molecule/Enzyme/Protein'].dropna().unique()
)


matches = pd.DataFrame()
for (drug, chemical) in drug_chemicals:

    mol_matches = {
            'PMID': [],
            'title': [],
        }    
    # Match drug and chemical
    q = Q("match_phrase", abstract=drug) & Q("match_phrase", abstract=chemical)
    
    # Search
    hits = Search(
        using=es,
        index="pubmed"
    ).params(
        request_timeout=300
    ).query(q)
    for h in hits:
        mol_matches['PMID'].append(h.pmid)
        mol_matches['title'].append(h.title)
    
    match_df = pd.DataFrame.from_dict(mol_matches)
    match_df['drug'] = drug
    match_df['chemical'] = chemical
    matches = matches.append(match_df)

matches.head()
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
      <th>PMID</th>
      <th>title</th>
      <th>drug</th>
      <th>chemical</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>9655178</td>
      <td>Association of myeloperoxidase with heparin: o...</td>
      <td>heparin</td>
      <td>Hydrogen Peroxide</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10187813</td>
      <td>Oxidation of methionine residues in antithromb...</td>
      <td>heparin</td>
      <td>Hydrogen Peroxide</td>
    </tr>
    <tr>
      <th>2</th>
      <td>30879129</td>
      <td>Heparin prevents oxidative stress-induced apop...</td>
      <td>heparin</td>
      <td>Hydrogen Peroxide</td>
    </tr>
    <tr>
      <th>3</th>
      <td>8181006</td>
      <td>Evidence of a selective free radical degradati...</td>
      <td>heparin</td>
      <td>Hydrogen Peroxide</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1321628</td>
      <td>Heparin: does it act as an antioxidant in vivo?</td>
      <td>heparin</td>
      <td>Hydrogen Peroxide</td>
    </tr>
  </tbody>
</table>
</div>




```python
article_count = pd.DataFrame(
    matches.groupby(['drug', 'chemical']).PMID.nunique()
).reset_index().rename(columns={'PMID': 'Article Count'})
piv_count = article_count.pivot_table(
    index='chemical',
    columns='drug',
    values='Article Count',
    fill_value=0
)
sns.clustermap(
    piv_count,
    figsize=(22,13),
    cmap='viridis'
)

```




    <seaborn.matrix.ClusterGrid at 0x7fe7382b6588>




![png](output_8_1.png)


## Find PMIDS assocaited with drugs and chemicals via elastic search


```python
drug_matches = pd.DataFrame()
for drug, m_df in drug_list_df.groupby('Name'):
    drug_match = {
            'PMID': [],
            'title': [],
            'abstract': [],
            'MeSH': []
        }
    hits = Search(
        using=es,
        index="pubmed"
        ).params(
            request_timeout=300
        ).query(
            "match_phrase",
            abstract=drug,
        )
    for h in hits:
        drug_match['PMID'].append(h.pmid)
        drug_match['title'].append(h.title)
        drug_match['abstract'].append(h.abstract)
        drug_match['MeSH'].append(h.MeSH)


    drug_match_df = pd.DataFrame.from_dict(drug_match)
    drug_match_df['drug'] = drug
    drug_matches = drug_matches.append(drug_match_df)

drug_matches.head()
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
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[Actinomycetales, chemistry, enzymology, Adeno...</td>
      <td>8784428</td>
      <td>a phosphotransferase which modifies the alpha ...</td>
      <td>Acarbose 7-phosphotransferase from Actinoplane...</td>
      <td>Acarbose</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[Acarbose, Adult, Blood Glucose, metabolism, C...</td>
      <td>6350115</td>
      <td>in a double blind study we have compared the e...</td>
      <td>Effect of acarbose, pectin, a combination of a...</td>
      <td>Acarbose</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[Acarbose, Adult, Aged, Blood Glucose, metabol...</td>
      <td>9663365</td>
      <td>acarbose is an alpha glucosidase inhibitor app...</td>
      <td>Effects of beano on the tolerability and pharm...</td>
      <td>Acarbose</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[Acarbose, administration &amp; dosage, Animals, B...</td>
      <td>11779583</td>
      <td>as alpha glucosidase inhibitor, the antidiabet...</td>
      <td>Chronic acarbose-feeding increases GLUT1 prote...</td>
      <td>Acarbose</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[Acarbose, Aged, Blood Glucose, metabolism, Di...</td>
      <td>9428831</td>
      <td>to compare the therapeutic potential of acarbo...</td>
      <td>Efficacy of 24-week monotherapy with acarbose,...</td>
      <td>Acarbose</td>
    </tr>
  </tbody>
</table>
</div>




```python
drug_matches.to_csv('Drug_PMID_occurances.csv', index=False)
```


```python
chem_matches = pd.DataFrame()
for chemical, m_df in chemical_list_df.groupby('Molecule/Enzyme/Protein'):
    chem_match = {
            'PMID': [],
            'title': [],
            'abstract': [],
            'MeSH': []
        }
    hits = Search(
        using=es,
        index="pubmed"
        ).params(
            request_timeout=300
        ).query(
            "match_phrase",
            abstract=chemical,
        )
    for h in hits:
        chem_match['PMID'].append(h.pmid)
        chem_match['title'].append(h.title)
        chem_match['abstract'].append(h.abstract)
        chem_match['MeSH'].append(h.MeSH)


    chem_match_df = pd.DataFrame.from_dict(chem_match)
    chem_match_df['chemical'] = chemical
    chem_matches = chem_matches.append(chem_match_df)

chem_matches.head()
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
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[]</td>
      <td>31368101</td>
      <td>coronary spasm plays an important role in the ...</td>
      <td>Association of East Asian Variant Aldehyde Deh...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[Acetylcholinesterase, metabolism, Aldehydes, ...</td>
      <td>10463393</td>
      <td>we have investigated the effect of soman induc...</td>
      <td>Increased levels of nitrogen oxides and lipid ...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[Aldehydes, chemistry, Amines, chemistry, Benz...</td>
      <td>8448343</td>
      <td>the reaction of trans 4 hydroxy 2 nonenal (4 h...</td>
      <td>Pyrrole formation from 4-hydroxynonenal and pr...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[Animals, Blood-Brain Barrier, metabolism, pat...</td>
      <td>29775963</td>
      <td>brain ischemic preconditioning (ipc) with mild...</td>
      <td>Brain ischemic preconditioning protects agains...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[Alzheimer Disease, drug therapy, enzymology, ...</td>
      <td>30218858</td>
      <td>excessive production of amyloid β (aβ) induced...</td>
      <td>Neuro-protective effects of aloperine in an Al...</td>
      <td>4-hydroxy-2-nonenal (4-HNE)</td>
    </tr>
  </tbody>
</table>
</div>




```python
chem_matches.to_csv('Chemical_PMID_occurances.csv', index=False)
```


```python
chem_matches.merge(drug_matches[['PMID', 'drug']])
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
      <th>drug</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[Anaphylaxis, metabolism, physiopathology, Ani...</td>
      <td>8576313</td>
      <td>nitric oxide, synthesized from the guanidino g...</td>
      <td>Role of nitric oxide in anaphylactic shock.</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[Animals, Blotting, Western, Denervation, Immu...</td>
      <td>10391455</td>
      <td>nitric oxide may be liberated as an inflammato...</td>
      <td>Evidence for nitric oxide and nitric oxide syn...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[Administration, Inhalation, Adult, Aged, Atri...</td>
      <td>15891329</td>
      <td>to compare hemodynamic and gasometric variable...</td>
      <td>Lack of alteration of endogenous nitric oxide ...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[Administration, Inhalation, Coronary Artery B...</td>
      <td>8614135</td>
      <td>increased pulmonary vascular resistance may gr...</td>
      <td>Effective control of pulmonary vascular resist...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[1-Methyl-3-isobutylxanthine, pharmacology, An...</td>
      <td>9722153</td>
      <td>the structures capable of synthesizing cyclic ...</td>
      <td>Distribution of nitric oxide synthase and nitr...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>5</th>
      <td>[Amino Acid Oxidoreductases, metabolism, Anima...</td>
      <td>7543936</td>
      <td>in the myenteric plexus of the guinea pig ileu...</td>
      <td>Analysis of connections between nitric oxide s...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>6</th>
      <td>[Animals, Humans, Hypertension, metabolism, ph...</td>
      <td>9914862</td>
      <td>nitric oxide is hypothesized to be an inhibito...</td>
      <td>Neural mechanisms in nitric-oxide-deficient hy...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>7</th>
      <td>[Amino Acid Oxidoreductases, biosynthesis, Ani...</td>
      <td>7532824</td>
      <td>nitric oxide is produced in the cns by both co...</td>
      <td>Modulation of inducible nitric oxide synthase ...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>8</th>
      <td>[Administration, Inhalation, Animals, Drug Adm...</td>
      <td>12594148</td>
      <td>pulsed administration of nitric oxide has prov...</td>
      <td>Administration of nitric oxide into open lung ...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
    <tr>
      <th>9</th>
      <td>[Adult, Female, Humans, Luminescent Measuremen...</td>
      <td>10367927</td>
      <td>nasal nitric oxide is present in high concentr...</td>
      <td>Nitric oxide accumulation in the nonventilated...</td>
      <td>Nitric oxide</td>
      <td>nitric oxide</td>
    </tr>
  </tbody>
</table>
</div>


