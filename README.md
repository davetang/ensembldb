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

