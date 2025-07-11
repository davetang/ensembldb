## Table of Contents

- [Ensembl public databases](#ensembl-public-databases)
- [Setup](#setup)
- [Rattus norvegicus](#rattus-norvegicus)
  - [Getting started](#getting-started)
  - [Tables](#tables)

# Ensembl public databases

Typically one wouldn't need to delve into the innards of the [Ensembl public MySQL Servers](https://www.ensembl.org/info/data/mysql.html?redirect=no) because it's much easier to use Biomart. It also requires understanding the [database schema](https://www.ensembl.org/info/docs/api/core/core_schema.html), which is not that straightforward. However, Biomart is sometimes unreachable and/or does not work as expected. A deeper understanding of the Ensembl public databases can help in those cases since we can directly retrieve the data we want from the databases. Given my reliance on Ensembl, it is a worthwhile endeavour to try to understand the backend of Ensembl.

# Setup

On Debian 12, I wasn't sure which package to install so I made an initial search.

```console
sudo apt update
sudo apt search mysql > blah

# after looking through blah
sudo apt install -y default-mysql-client
mysql --version
```
```
mysql  Ver 15.1 Distrib 10.11.13-MariaDB, for debian-linux-gnu (x86_64) using  EditLine wrapper
```

In case you were wondering about the compatibility between MariaDB and MySQL, at least the [client protocol](https://mariadb.com/docs/release-notes/compatibility-and-differences/mariadb-vs-mysql-compatibility) is binary compatible.

Get all the databases as a test.

```console
mysql --user=anonymous --host=ensembldb.ensembl.org -A -e "show databases;" > ensembldb_databases.txt
cat ensembldb_databases.txt| wc -l
```
```
24290
```

# Rattus norvegicus

## Getting started

This work was motivated by the fact that the Ensembl version 113 archive does not contain rat references, which means I can't use Biomart. Looking through `ensembldb_databases.txt`, the database I need is `rattus_norvegicus_core_113_72`. I need a single lookup table that connects Ensembl gene IDs with their associated Gene Ontology (GO) terms. Looking through the schema, it seems the table that I need is [ontology_xref](https://www.ensembl.org/info/docs/api/core/core_schema.html#ontology_xref).

* This table associates Evidence Tags to the relationship between EnsEMBL objects and ontology accessions (primarily GO accessions).
* The relationship to GO that is stored in the database is actually derived through the relationship of EnsEMBL peptides to SwissProt peptides, i.e. the relationship is derived like this: ENSP -> SWISSPROT -> GO And the evidence tag describes the relationship between the SwissProt Peptide and the GO entry.
* In reality, however, we store this in the database like this: ENSP -> SWISSPROT ENSP -> GO and the evidence tag hangs off of the relationship between the ENSP and the GO identifier.
* Some ENSPs are associated with multiple closely related Swissprot entries which may both be associated with the same GO identifier but with different evidence tags.
* For this reason a single Ensembl - external db object relationship in the object_xref table can be associated with multiple evidence tags in the ontology_xref table.

```console
mysql --user=anonymous --host=ensembldb.ensembl.org -A
```
```mysql
use rattus_norvegicus_core_113_72;
select * from ontology_xref limit 5;
```
```
+----------------+----------------+--------------+
| object_xref_id | source_xref_id | linkage_type |
+----------------+----------------+--------------+
|       11895259 |        4650749 | IDA          |
|       11895260 |        4650749 | IEA          |
|       11895261 |        4646135 | IEA          |
|       11895262 |        4646135 | IEA          |
|       11895263 |        4653274 | IEA          |
+----------------+----------------+--------------+
5 rows in set (0.868 sec)
```

* `object_xref_id` - Composite key. Foreign key references to the `object_xref` table.
* `source_xref_id` - Composite key. Foreign key references to the `xref` table.

`object_xref`:

> Describes links between EnsEMBL objects and objects held in external databases. The EnsEMBL object can be one of several types; the type is held in the ensembl_object_type column. The ID of the particular EnsEMBL gene, translation or whatever is given in the ensembl_id column. The xref_id points to the entry in the xref table that holds data about the external object. Each EnsEMBL object can be associated with zero or more xrefs. An xref object can be associated with one or more EnsEMBL objects.

```mysql
select * from object_xref limit 5;
```
```
+----------------+------------+---------------------+---------+--------------------+-------------+
| object_xref_id | ensembl_id | ensembl_object_type | xref_id | linkage_annotation | analysis_id |
+----------------+------------+---------------------+---------+--------------------+-------------+
|        4089743 |      30316 | Gene                | 3859187 | NULL               |          72 |
|        4088975 |      18247 | Gene                | 3859187 | NULL               |          72 |
|        4088616 |      17440 | Gene                | 3859187 | NULL               |          72 |
|        4088163 |      11380 | Gene                | 3859187 | NULL               |          72 |
|        4088491 |       3301 | Gene                | 3859187 | NULL               |          72 |
+----------------+------------+---------------------+---------+--------------------+-------------+
5 rows in set (0.254 sec)
```

`xref`:

> Holds data about objects which are external to EnsEMBL, but need to be associated with EnsEMBL objects. Information about the database that the external object is stored in is held in the external_db table entry referred to by the external_db column.

```mysql
select * from xref limit 5;
```
```
+---------+----------------+---------------+----------------+---------+-------------------------------------------------------+----------------+-----------+
| xref_id | external_db_id | dbprimary_acc | display_label  | version | description                                           | info_type      | info_text |
+---------+----------------+---------------+----------------+---------+-------------------------------------------------------+----------------+-----------+
| 4637553 |           1810 | NP_001388970  | NP_001388970.1 | 1       | UPF0450 protein C17orf58 homolog isoform 2 precursor  | SEQUENCE_MATCH |           |
|       2 |           2250 | P18088        | P18088         | 0       | NULL                                                  | DEPENDENT      |           |
|       3 |           2250 | C9E895        | C9E895         | 0       | NULL                                                  | DEPENDENT      |           |
|       5 |           2250 | D3ZEM1        | D3ZEM1         | 0       | NULL                                                  | DEPENDENT      |           |
|       7 |           2250 | D3ZEL3        | D3ZEL3         | 0       | NULL                                                  | DEPENDENT      |           |
+---------+----------------+---------------+----------------+---------+-------------------------------------------------------+----------------+-----------+
```

Join with `object_xref`.

```mysql
select * from ontology_xref join object_xref on ontology_xref.object_xref_id = object_xref.object_xref_id limit 5;
```
```
+----------------+----------------+--------------+----------------+------------+---------------------+---------+--------------------+-------------+
| object_xref_id | source_xref_id | linkage_type | object_xref_id | ensembl_id | ensembl_object_type | xref_id | linkage_annotation | analysis_id |
+----------------+----------------+--------------+----------------+------------+---------------------+---------+--------------------+-------------+
|       11895259 |        4650749 | IDA          |       11895259 |      36056 | Transcript          |  861201 | IDA                |         153 |
|       11895260 |        4650749 | IEA          |       11895260 |      36056 | Transcript          | 1711926 | IEA                |         153 |
|       11895261 |        4646135 | IEA          |       11895261 |       1754 | Transcript          | 1711880 | IEA                |         153 |
|       11895262 |        4646135 | IEA          |       11895262 |       1754 | Transcript          | 1711881 | IEA                |         153 |
|       11895263 |        4653274 | IEA          |       11895263 |      54956 | Transcript          | 1711882 | IEA                |         153 |
+----------------+----------------+--------------+----------------+------------+---------------------+---------+--------------------+-------------+
5 rows in set (0.282 sec)
```

Join with `xref`.

```mysql
select * from ontology_xref join xref on ontology_xref.source_xref_id = xref.xref_id limit 5;
```
```
+----------------+----------------+--------------+---------+----------------+---------------+---------------+---------+-------------+-----------+-----------+
| object_xref_id | source_xref_id | linkage_type | xref_id | external_db_id | dbprimary_acc | display_label | version | description | info_type | info_text |
+----------------+----------------+--------------+---------+----------------+---------------+---------------+---------+-------------+-----------+-----------+
|       11895259 |        4650749 | IDA          | 4650749 |          50817 | URS0000013967 | URS0000013967 | 1       | NULL        | CHECKSUM  |           |
|       11895260 |        4650749 | IEA          | 4650749 |          50817 | URS0000013967 | URS0000013967 | 1       | NULL        | CHECKSUM  |           |
|       11895261 |        4646135 | IEA          | 4646135 |          50817 | URS0000060CC3 | URS0000060CC3 | 1       | NULL        | CHECKSUM  |           |
|       11895262 |        4646135 | IEA          | 4646135 |          50817 | URS0000060CC3 | URS0000060CC3 | 1       | NULL        | CHECKSUM  |           |
|       11895263 |        4653274 | IEA          | 4653274 |          50817 | URS000007B8A8 | URS000007B8A8 | 1       | NULL        | CHECKSUM  |           |
+----------------+----------------+--------------+---------+----------------+---------------+---------------+---------+-------------+-----------+-----------+
5 rows in set (0.367 sec)
```

I'm not familiar with `URS` IDs, let's check the `external_db` table to see what they are.

```mysql
select * from external_db where external_db_id = 50817;
```
```
+----------------+------------+------------+-----------+----------+-----------------+--------------------+-------------------+--------------------+-------------------------------------+
| external_db_id | db_name    | db_release | status    | priority | db_display_name | type               | secondary_db_name | secondary_db_table | description                         |
+----------------+------------+------------+-----------+----------+-----------------+--------------------+-------------------+--------------------+-------------------------------------+
|          50817 | RNAcentral | NULL       | KNOWNXREF |        0 | RNAcentral      | PRIMARY_DB_SYNONYM | NULL              | NULL               | Sequence identity to RNAcentral ids |
+----------------+------------+------------+-----------+----------+-----------------+--------------------+-------------------+--------------------+-------------------------------------+
1 row in set (0.310 sec)
```

Looking online I found [RNAcentral: The non-coding RNA sequence database](https://rnacentral.org/); let's tally all the external databases.

First count all entries (as I'm not confident with my SQL skills).

```mysql
select count(*) as total from ontology_xref join xref on ontology_xref.source_xref_id = xref.xref_id;
```
```
+--------+
| total  |
+--------+
| 164031 |
+--------+
1 row in set (0.292 sec)
```

Tally.

```mysql
select xref.external_db_id, count(xref.external_db_id) as total from ontology_xref join xref on ontology_xref.source_xref_id = xref.xref_id group by xref.external_db_id;
```
```
+----------------+-------+
| external_db_id | total |
+----------------+-------+
|           2000 |  6109 |
|           2200 |  7996 |
|           2250 | 76238 |
|          50621 | 62300 |
|          50817 | 11244 |
|          50869 |   144 |
+----------------+-------+
6 rows in set (0.345 sec)
```

Double-check the total.

```console
echo $(( 6109+7996+76238+62300+11244+144 ))
```
```
164031
```

Join on `external_db` so we have the actual names of the databases.

```mysql
select external_db.db_display_name, count(external_db.db_display_name) as total from ontology_xref join xref on ontology_xref.source_xref_id = xref.xref_id join external_db on xref.external_db_id = external_db.external_db_id group by external_db.external_db_id;
```
```
+-----------------------------------------------------------------------------+-------+
| db_display_name                                                             | total |
+-----------------------------------------------------------------------------+-------+
| UniProtKB/TrEMBL                                                            |  6109 |
| UniProtKB/Swiss-Prot                                                        |  7996 |
| UniProtKB generic accession number (TrEMBL or SwissProt not differentiated) | 76238 |
| UniProtKB Gene Name                                                         | 62300 |
| RNAcentral                                                                  | 11244 |
| UniProtKB isoform                                                           |   144 |
+-----------------------------------------------------------------------------+-------+
6 rows in set (0.369 sec)
```

Let's see if we get similar results using version 114.

```mysql
use rattus_norvegicus_core_114_1;
select external_db.db_display_name, count(external_db.db_display_name) as total from ontology_xref join xref on ontology_xref.source_xref_id = xref.xref_id join external_db on xref.external_db_id = external_db.external_db_id group by external_db.external_db_id;
```
```
+-----------------------------------------------------------------------------+--------+
| db_display_name                                                             | total  |
+-----------------------------------------------------------------------------+--------+
| InterPro                                                                    | 167744 |
| UniProtKB/TrEMBL                                                            | 130525 |
| UniProtKB/Swiss-Prot                                                        |  32202 |
| UniProtKB generic accession number (TrEMBL or SwissProt not differentiated) | 123811 |
| RNAcentral                                                                  |  10972 |
| UniProtKB isoform                                                           |    146 |
+-----------------------------------------------------------------------------+--------+
6 rows in set (0.563 sec)
```

Looking at the [Ensembl 114 has been released! blog post](https://www.ensembl.info/2025/05/07/ensembl-114-has-been-released/) the large difference in numbers may be due to:

> Updated Rat reference (GRCr8 automated annotation)
>     Rattus norvegicus (GRCr8)

## Tables

Export `ontology_xref` joined with `object_xref` and `xref`.

```console
mysql \
   --user=anonymous \
   --host=ensembldb.ensembl.org \
   -A \
   -e "use rattus_norvegicus_core_113_72; select * from ontology_xref join object_xref on ontology_xref.object_xref_id = object_xref.object_xref_id join xref on ontology_xref.source_xref_id = xref.xref_id;" \
   > rattus_norvegicus_core_113_72_ontology_xref_join.txt
```

What I currently still don't understand is why there are no GO IDs in `rattus_norvegicus_core_113_72_ontology_xref_join.txt`.

The query below shows that in the `xref` table there are 205056 GO entries.

```mysql
select external_db.db_display_name, count(external_db.db_display_name) as total from xref join external_db on xref.external_db_id = external_db.external_db_id group by external_db.db_display_name;
```
```
+-----------------------------------------------------------------------------+--------+
| db_display_name                                                             | total  |
+-----------------------------------------------------------------------------+--------+
| BioGRID Interaction data, The General Repository for Interaction Datasets   |   1195 |
| ChEMBL                                                                      |    527 |
| EntrezGene transcript name                                                  |      2 |
| European Nucleotide Archive                                                 |  12244 |
| Expression Atlas                                                            |  30562 |
| GO                                                                          | 205056 |
| INSDC protein ID                                                            |  23530 |
| InterPro                                                                    |  45163 |
| MEROPS - the Peptidase Database                                             |    481 |
| MGI Symbol                                                                  |      7 |
| MGI transcript name                                                         |      8 |
| miRBase                                                                     |    424 |
| miRBase transcript name                                                     |      4 |
| NCBI gene (formerly Entrezgene)                                             |  21395 |
| PDB                                                                         |   1824 |
| Reactome                                                                    |   1694 |
| Reactome gene                                                               |   1731 |
| Reactome transcript                                                         |   1697 |
| RefSeq mRNA                                                                 |  18964 |
| RefSeq mRNA predicted                                                       |  53563 |
| RefSeq ncRNA                                                                |    519 |
| RefSeq ncRNA predicted                                                      |   6348 |
| RefSeq peptide                                                              |  20432 |
| RefSeq peptide predicted                                                    |  37932 |
| RFAM                                                                        |     54 |
| RFAM transcript name                                                        |    113 |
| RGD Symbol                                                                  |  26739 |
| RGD transcript name                                                         |  49208 |
| RNAcentral                                                                  |   7876 |
| UniParc                                                                     |  44049 |
| UniProt transcript name                                                     |     65 |
| UniProtKB Gene Name                                                         |  53527 |
| UniProtKB generic accession number (TrEMBL or SwissProt not differentiated) |  15375 |
| UniProtKB isoform                                                           |    833 |
| UniProtKB/Swiss-Prot                                                        |   4436 |
| UniProtKB/TrEMBL                                                            |  49957 |
| WikiGene                                                                    |  21395 |
+-----------------------------------------------------------------------------+--------+
```

I would imagine that `ontology_xref.source_xref_id` matches GO entries in `xref` but this is not the case; there are no GO IDs in the joined table. Recall `ontology_xref` looks like this:

```
+----------------+----------------+--------------+
| object_xref_id | source_xref_id | linkage_type |
+----------------+----------------+--------------+
|       11895259 |        4650749 | IDA          |
|       11895260 |        4650749 | IEA          |
|       11895261 |        4646135 | IEA          |
|       11895262 |        4646135 | IEA          |
|       11895263 |        4653274 | IEA          |
+----------------+----------------+--------------+
5 rows in set (0.868 sec)
```

Version 114 shows similar results where there are also no GO IDs in the joined table.

```console
mysql \
   --user=anonymous \
   --host=ensembldb.ensembl.org \
   -A \
   -e "use rattus_norvegicus_core_114_1; select * from ontology_xref join object_xref on ontology_xref.object_xref_id = object_xref.object_xref_id join xref on ontology_xref.source_xref_id = xref.xref_id;" \
   > rattus_norvegicus_core_114_1_ontology_xref_join.txt
```

I am literally missing a link.
