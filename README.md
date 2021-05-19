KrakenUniq: confident and fast metagenomics classification using unique k-mer counts
===============================================

This fork suppresses reverse-complement k-mers from being added to the index.

False-positive identifications are a significant problem in metagenomics classification. KrakenUniq (formerly KrakenHLL) is a novel metagenomics classifier that combines the fast k-mer-based classification of [Kraken](https://github.com/DerrickWood/kraken) with an efficient algorithm for assessing the coverage of unique k-mers found in each species in a dataset. On various test datasets, KrakenUniq gives better recall and precision than other methods and effectively classifies and distinguishes pathogens with low abundance from false positives in infectious disease samples. By using the probabilistic cardinality estimator HyperLogLog, KrakenUniq runs as fast as Kraken and requires little additional memory. 

**If you use KrakenUniq in your research, please cite our publication:** [KrakenUniq: confident and fast metagenomics classification using unique k-mer counts. Breitwieser FP, Baker DN, Salzberg SL. Genome Biology, Dec 2018. https://doi.org/10.1186/s13059-018-1568-0](https://doi.org/10.1186/s13059-018-1568-0)

## Installation
[![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg?style=flat-square)](http://bioconda.github.io/recipes/krakenuniq/README.html)
[![Anaconda-Server Badge](https://anaconda.org/bioconda/krakenuniq/badges/latest_release_date.svg)](https://anaconda.org/bioconda/krakenuniq)
[![Anaconda-Server Badge](https://anaconda.org/bioconda/krakenuniq/badges/platforms.svg)](https://anaconda.org/bioconda/krakenuniq)

KrakenUniq is available in the Anaconda cloud. To install, type:

```
conda install krakenuniq
```

Installation from source from GitHub:
```
git clone https://github.com/fbreitwieser/krakenuniq
cd krakenuniq
./install_krakenuniq /PATH/TO/INSTALL_DIR
```

Note that KrakenUniq requires Jellyfish v1 to be installed for the database building step (`krakenuniq-build`). To install Jellyfish alongside KrakenUniq, use the `-j` flag for the `install_krakenhll.sh` script. Alternatively, you can specify the Jellyfish path to `krakenuniq-build` with `krakenuniq-build --jellyfish-bin /usr/bin/jellyfish1`.

OSX by default links `g++` to `clang` without OpenMP support. When using clang, you may get the error `clang: fatal error: unsupported option '-fopenmp'`. To fix this, install `g++` with HomeBrew and use the `-c` option of `krakenuniq_install.sh` to specify the HomeBrew version of `g++`, which is accessible with `g++-8`: 
``` 
brew install gcc
./install_krakenuniq -c g++-8 /PATH/TO/INSTALL_DIR
```

## Database building

Note that KrakenUniq natively supports Kraken 1 databases (however not Kraken 2). If you have existing Kraken databases, you may run KrakenUniq directly on them, though for support of taxon nodes for genomes and sequences (see below) you will need to rebuild them with KrakenUniq. For building a custom database, there are three requirements:

1. Sequence files (FASTA format)
2. Mapping files (tab separated format, `sequence header<tab>taxID`
3. NCBI taxonomy files (though a custom taoxnomies may be used, too)

While you may supply this information yourself, `krakenuniq-download` supports a variety of data sources to download the taxonomy, sequence and mapping files. Please find examples below on how to download different sequence sets:

```
## Download the taxonomy
krakenuniq-download --db DBDIR taxonomy

## All complete bacterial and archaeal genomes genomes in RefSeq using 10 threads, and masking low-complexity sequences in the genomes
krakenuniq-download --db DBDIR --threads 10 --dust refseq/bacteria refseq/archaea

## Contaminant sequences from UniVec and EmVec, plus the human reference genome
krakenuniq-download --db DBDIR refseq/vertebrate_mammalian/Chromosome/species_taxid=9606

## All viral genomes from RefSeq plus viral 'neighbors' in NCBI Nucleotide
krakenuniq-download --db DBDIR refseq/viral/Any viral-neighbors

## All microbial (including eukaryotes) sequences in the NCBI nt database
krakenuniq-download --db DBDIR --dust microbial-nt
```

To build the database indices on the downloaded files, run `krakenuniq-build --db DBDIR`.  To build a database with a *k*-mer length of 31 (the default), adding virtual taxonomy nodes for genomes and sequences (off by default), run `krakenuniq-build` with the following parameters:
```
krakenuniq-build --db DBDIR --kmer-len 31 --threads 10 --taxids-for-genomes --taxids-for-sequences

```

For more information on taxids for genomes and sequences, look at the [manual](MANUAL.md). The building step may take up to a couple of days on large sequence sets such as nt.

## Classification

To run classification on a pair of FASTQ files, use `krakenuniq`.

```
krakenuniq --db DBDIR --threads 10 --report-file REPORTFILE.tsv > READCLASSIFICATION.tsv
```

It can be advantegeous to preload the database prior to the first run. KrakenUniq uses mmap to map the database files into memory, which reads the file on demand. `krakenuniq --preload` reads the full database into memory, so that subsequent runs can benefit from the mapped pages. You do not need to specify preload before every run, but only after restarting the machine or when using a new database.

```
krakenuniq --db DBDIR --preload --threads 10
krakenuniq --db DBDIR --threads 10 --report-file REPORTFILE.tsv > READCLASSIFICATION.tsv
...
```


## FAQ

### Memory requirements

KrakenUniq requires a lot of RAM - ideally 128GB - 512GB. For more memory efficient classification consider using [centrifuge](https://github.com/infphilo/centrifuge).

### KrakenUniq vs Kraken vs Kraken 2

KrakenUniq was built on top of Kraken, and supports Kraken 1 databases natively. Kraken 2 is a new development that has a different database format, which is not supported by KrakenUniq.

### Differences to `kraken`
 - Use `krakenuniq --report-file FILENAME ...` to write the kraken report to `FILENAME`.
 - Use `krakenuniq --db DB1 --db DB2 --db DB3 ...` to first attempt, for each k-mer, to assign it based on DB1, then DB2, then DB3. You can use this to prefer identifications based on DB1 (e.g. human and contaminant sequences), then DB2 (e.g. completed bacterial genomes), then DB3, etc. Note that this option is incompatible with `krakenuniq-build --taxids-for-genomes --taxids-for-sequences` since the taxDB between the databases has to be absolutely the same.
 - Add a suffix `.gz` to output files to generate gzipped output files

### Differences to `kraken-build`
 - Use `krakenuniq-build --taxids-for-genomes --taxids-for-sequences ...` to add pseudo-taxonomy IDs for each sequence header and genome assembly (when using `krakenuniq-download`). 
 - `seqid2taxid.map` mapping sequence IDs to taxonomy IDs does NOT parse or require `>gi|`, but rather the sequence ID is the header up to just before the first space
 
### Building a microbial nt database

KrakenUniq supports building databases on subsets of the NCBI nucleotide collection nr/nt, which is most prominently the standard database for BLASTn. On the command line, you can specify to extract all bacterial, viral, archaeal, protozoan, fungal and helminth sequences. The list of protozoan taxa is based on [Kaiju's](https://raw.githubusercontent.com/bioinformatics-centre/kaiju/master/util/taxonlist.tsv).

Example command line:
```
krakenuniq-download --db DB --taxa "archaea,bacteria,viral,fungi,protozoa,helminths" --dust --exclude-environmental-taxa microbial-nt
```


### Custom databases with NCBI taxonomy
To build a custom database with the NCBI taxonomy, first download the taxonomy files with
```
krakenuniq-download --db DBDIR taxonomy
```
Then you can add the desired sequence files to the `DBDIR/library` directory:
```
cp SEQ1.fa SEQ2.fa DBDIR/library
```
KrakenUniq needs a _sequence ID to taxonomy ID mapping_ for each sequence. This mappings can be provided in the `DBDIR/library/*.map` - KrakenUniq pools all `.map` files inside of the `library/` folder prior to database building. Format: three tab-separated fields that are, in order, the sequence ID (i. e. the sequence header without '>' up to the first space), the taxonomy ID and the genome or assembly name:
```
Strain1_Chr1_Seq     <tab> 562 <tab> E. Coli Strain Foo
Strain1_Chr2_Seq     <tab> 562 <tab> E. Coli Strain Foo
Strain1_Plasmid1_Seq <tab> 562 <tab> E. Coli Strain Foo
Strain2_Chr1_Seq     <tab> 621 <tab> S. boydii Strain Bar
Strain2_Plasmid1_Seq <tab> 621 <tab> S. boydii Strain Bar
```
The third column is optional, and used by KrakenUniq only when `--taxids-for-genomes` is specified for `krakenuniq-build` to add new nodes in the taxonomy tree for the genome. If you'd like to have the sequences identifier in the taxonomy report, too, specifiy `--taxids-for-sequences` for `krakenuniq-build` as well.

Finally, run `krakenuniq-build`:
```
krakenuniq-build --db DBDIR --taxids-for-genomes --taxids-for-sequences
```

Note that for custom databases with fewer sequences you might want to choose a smaller k (default: `--kmer-len 31`) and minimizer length (default: `--minimizer-len 15`).

### Custom databases with custom taxonomies

When using custom taxonomies, please provide `DBDIR/taxonomy/nodes.dmp` and `DBDIR/taxonomy/names.dmp` according to the format of NCBI taxonomy dumps.

## License

The code adpated from Kraken 1 is licensed under GPL 3.0. All code added in this project (such as the
HyperLogLog algorithm code) is dual-licensed under MIT and GPL 3.0 (or any later version), unless stated otherwise.
You can choose between one of them if you use that work.
