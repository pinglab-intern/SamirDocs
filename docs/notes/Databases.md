# Databases used in this project

## UniProt
Two primary sections:
1. Swiss-Prot/ Reviewed: Manually annotated records
2. TrEMBL/ Unreviewed: Computationally annotated records
Subset of UniProtKB —> Proteomes data set

Allows search via:  

* BLAST (local alignment)
* Multiple sequence alignment
* ID Mapping
* ‘Peptide search’ for 3 or more residues

## Reactome Pathway Database
* Compilation of many sources of data including Kegg, UniProt, NCBI, Ensemble, etc.
    * Manually curated, open-source, peer-reviewed
* Neo4J representation exists. Integrated with Spring Data Neo4j
* Tutorial on extracting different data types/building reactome cypher queries: https://reactome.org/dev/graph-database/extract-participating-molecules

## [Drugbank](https://www.drugbank.ca)
* Comprehensive free database of drugs and drug targets
* Well maintained
* 13k+ drug entries, 5k+ protein sequences (drug targets/enzymes/etc)
* Data base can be downloaded [here](https://www.drugbank.ca/releases/latest)
    * Free for non-commercial use
    * Contains sub databases to download as well (structures, protein identifiers, target sequences, etc.)

