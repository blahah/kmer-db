# Kmer-db

Kmer-db is a fast and memory-efficient tool for estimating evolutionary distances.

## REQUIREMENTS

Kmer-db requires a CPU supporting AVX extensions (AVX2 is used optionally).

## INSTALLATION AND CONFIGURATION

Kmer-db comes with a set of precompiled binaries for Windows and Linux. 
The software can be also built from the sources distributed as:

* MAKE project (G++ 4.8 required) for Linux and OS X.
* Visual Studio 2015 solution for Windows,


### *zlib* linking

Kmer-db uses *zlib* for handling gzipped inputs. Under Linux, the software is by default linked against system-installed *zlib*. Due to issues with some library versions, precompiled *zlib* is also present the repository. In order to use it, one needs to modify variable INTERNAL_ZLIB at the top of the makefile. Under Windows, the repository library is always used.

### AVX2 support

Kmer-db by default takes advantage of AVX and AVX2 CPU extensions. Pre-built binary detetermines supported instructions at runtime, thus it is multiplatform. However, one may encounter a problem when building default Kmer-db version on a CPU without AVX2. To prevent from using AVX2, the program must be compiled with NO_AVX2 symbolic constant defined. When building under Linux or OS X, there is a NO_AVX2 switch at the top of the makefile which does the job.

## USAGE
`kmer-db <mode> [options] <positional arguments>`

Kmer-db operates in one of the following modes:

* `build` - building a database from samples,
* `minhash` - minhashing k-mers,
* `all2all` - counting common k-mers - all samples in the database,
* `new2all` - counting common k-mers - set of new samples versus database,
* `one2all` - counting common k-mers - single sample versus database,
* `distance` - calculating similarities/distances.
    
Common options:
* `-t <threads>` - number of threads (default: number of available cores),
    
The meaning of other options and positional arguments depends on the selected mode.
    
### Building the database
Construction of k-mers database is an obligatory step for further analyses. The procedure accepts several input types:
* compressed or uncompressed genomes:

    ```kmer-db build [-f <filter> -k <kmer-length>] <sample_list> <database>```
* [KMC-generated](https://github.com/refresh-bio/KMC) k-mers: 

    ```kmer-db build -from-kmers [-f <filter>] <sample_list> <database>```
  
* minhashed k-mers produced by `minhash` mode:

    ```kmer-db build -from-minhash <sample_list> <database>```

Parameters:
* `sample_list` (input) - file containing list of samples in the following format:
    ```
    sample1
    sample2
    sample3
    ...
    ```
    By default, the tool requires compressed (*.gz*/*.fna.gz*/*.fasta.gz*) or uncompressed (*.fna*/*.fasta*) genome files for each sample (extensions are added automatically). When `-from-kmers` switch is specified, corresponding [KMC-generated](https://github.com/refresh-bio/KMC) k-mer files (*.kmc_pre* and *.kmc_suf*) are required. If `-from-minhash` switch is present, minhashed k-mer files (*.minhash*) must be generated by `minhash` command prior to the database construction. Note, that minhashing may be also done during the database construction by specyfying `-f` option.
* `database` (output) - file with generated k-mer database. 
* `-f <filter>` - number from [0,1] interval determining a fraction of all k-mers to be accepted by the minhash filter during database construction (default: 1); ignored when `-from-minhash` switch is present.
* `-k <kmer-length>` - length of k-mers (default: 18); ignored when `-from-kmers` or `-from-minhash` switch is specified. 


### Minhashing k-mers
This is an optional analysis step which stores minhashed k-mers on the hard disk to be later consumed by `build`, `new2all`, or `one2all` modes with `-from-minhash` switch. It can be skipped if one wants to use all k-mers from samples for distance estimation or employs minhashing during database construction. Syntax:

`kmer-db minhash <sample_list> <filter>`

Parameters:
 * `sample_list` (input) - file containing list of [KMC-generated](https://github.com/refresh-bio/KMC) k-mer files for samples (a pair of *.pre* and *.suf* files per sample is needed). 
 * `filter` (input) - number from [0,1] interval determining a fraction of all k-mers to be accepted by the filter.
 
  For each sample from the list, a binary file with *.minhash* extension containing filtered k-mers is created.

 
 ### Counting common k-mers ###
Counting common k-mers for all the samples in the database:
 
 `kmer_db all2all [-buffer <size_mb>] <database> <common_table>`
 
Parameters:
* `-buffer <size_mb>` - size of cache buffer in megabytes; use L3 size for Intel CPUs and L2 for AMD to maximize performance; default: 8
* `database` (input) - k-mer database file created by `build` mode,
* `common_table` (output) - file containing table with common k-mer counts.

Counting common k-mers between set of new samples and all the samples in the database:

`kmer_db new2all [-from-kmers|-from-minhash] <database> <sample_list> <common_table>`

Parameters:
* `-from-kmers`/`-from-minhash` - by default, the query samples are given as compressed or uncompressed genomes; use these switches for different sample formats (see `build` mode for details).
 * `database` (input) - k-mer database file created by `build` mode,
 * `sample_list` (input) - file containing list of samples in the same format as in the `build` mode; if samples are given as genomes (default) or k-mers (`-from-kmers` switch), the minhashing is done automatically with the same filter as in the database,
 * `common_table` (output) - file containing table with common k-mer counts.
 
 
Counting common k-mers between single sample and all the samples in the database:

`kmer_db one2all [-from-kmers|-from-minhash] <database> <sample> <common_table>`

The meaning of the parameters is the same as in `new2all` mode, but instead of specifying file with sample list, a single sample is used as a query.

Modes `all2all`, `new2all`, and `one2all` produce a comma-separated table with number of common k-mers. The table is in the following form:

| 									| 								| 					| 					|		|			|	
| :---: 							| :---: 						| :---: 			| :---:				| :---:	|  :---:	| 
| kmer-length: *k* fraction: *f* 	| db-samples 					| *s<sub>1</sub>*									| *s<sub>2</sub>* 										| ... 	|  *s<sub>n</sub>* |
| query-samples 					| total-kmers 					| &#124;*s<sub>1</sub>*&#124;						| &#124;*s<sub>2</sub>*&#124; 							| ... 	|  &#124;*s<sub>n</sub>*&#124; |
| *q<sub>1</sub>* 					| &#124;*q<sub>1</sub>*&#124;	| &#124;*q<sub>1</sub> &cap; s<sub>1</sub>*&#124;	| &#124;*q<sub>1</sub> &cap; s<sub>2</sub>*&#124; 	| ... 	|  &#124;*q<sub>1</sub> &cap; s<sub>n</sub>*&#124; |
| *q<sub>2</sub>* 					| &#124;*q<sub>1</sub>*&#124;	| &#124;*q<sub>2</sub> &cap; s<sub>1</sub>*&#124;	| &#124;*q<sub>2</sub> &cap; s<sub>2</sub>*&#124; 	| ... 	|  &#124;*q<sub>2</sub> &cap; s<sub>n</sub>*&#124; |
| ... 								| ...							| ...												| ...													| ...	|	...												 |
| *q<sub>m</sub>* 					| &#124;*q<sub>m</sub>*&#124;	| &#124;*q<sub>m</sub> &cap; s<sub>1</sub>*&#124;	| &#124;*q<sub>m</sub> &cap; s<sub>2</sub>*&#124; 	| ... 	|  &#124;*q<sub>m</sub> &cap; s<sub>n</sub>*&#124; |

where:
* *k* - k-mer length,
* *f* - minhash fraction (1, when minhashing is disabled), 
* *s<sub>1</sub>*, *s<sub>2</sub>*,  ...,   *s<sub>n</sub>* - database sample names,
* *q<sub>1</sub>*, *q<sub>2</sub>*,  ...,   *q<sub>m</sub>* - query sample names,
* &#124;*a*&#124; - number of k-mers in sample *a*,
* &#124;*a &cap; b*&#124; - number of k-mers common for samples *a* and *b*.


 
 ### Calculating similarities/distances
 
`kmer_db distance <common_table>`

Parameters:
* `common_table` (input) - file containing table with numbers of common k-mers produced by `all2all`, `new2all`, or `one2all` mode.

This mode generates a set of files containing tables with different similarity/distance measures:
* *<common_table>.jaccard*
* *<common_table>.min*
* *<common_table>.max*
* *<common_table>.cosine*
* *<common_table>.mash*

### Toy examples

Let *pathogens.list* be the file containing names of samples (there exist *.gz* or *.fasta* genome file for each sample):
```
acinetobacter
klebsiella
e.coli
...
```

Calculating similarities/distances between all samples listed in *pathogens.list* using all 20-mers: 
```
kmer-db build -k 20 pathogens.list pathogens.db
kmer-db all2all pathogens.db matrix.csv
kmer-db distance matrix.csv
```

Same as above, but using only 10% of 20-mers:
```
kmer-db build -k 20 -f 0.1 pathogens.list pathogens.db
kmer-db all2all pathogens.db matrix.csv
kmer-db distance matrix.csv
```

Calculating similarities/distances between samples listed in *pathogens.list* and *salmonella* using all 20-mers: 
```
kmer-db build -k 20 pathogens.list pathogens.db
kmer-db one2all pathogens.db salmonella vector.csv
kmer-db distance vector.csv
```

Same as above, but using only 10% of 20-mers:
```
kmer-db build -k 20 -f 0.1 pathogens.list pathogens.db
kmer-db one2all pathogens.db salmonella vector.csv
kmer-db distance vector.csv
```

## Datasets
List of the pathogens investigated in Kmer-db study can be found [here](https://github.com/refresh-bio/kmer-db/tree/master/data)

## Citing
[Deorowicz, S., Gudyś, A., Długosz, M., Kokot, M., Danek, A. (2018) Kmer-db: instant evolutionary distance estimation, Bioinformatics](https://academic.oup.com/bioinformatics/advance-article-abstract/doi/10.1093/bioinformatics/bty610/5050791?redirectedFrom=fulltext)
