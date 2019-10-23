Database
========

## Overview

The `database` command is an automated workflow for sampling reference genomes, downloading non-redundant protein sequences, and building local databases for sequence homology search. It provides various options for flexible customization of the database, to address specific research goals including HGT prediction or other general purposes.

```bash
hgtector database -o <output_dir> <parameters...>
```

### Default protocol

HGTector provides a default protocol for database building.

```bash
hgtector database -o <output_dir> --default
```

This will download all protein sequences of NCBI RefSeq genomes of bacteria, archaea, fungi and protozoa, keep one genome per species, plus all NCBI-defined reference and representative genomes. Finally it will attempt to compile the database using DIAMOND, if available. The command is equivalent to:

```bash
hgtector database --output <output_dir> --cats microbe --sample 1 --rank species --reference --representative --compile diamond
```

A pre-build default database as of 2019-10-21 is available for [download](https://www.dropbox.com/s/qdnfgzdcjadlm4i/hgtdb_20191021.tar.xz?dl=0). It needs to be [compiled](#Manual-compiling) using choice of aligner.

### Procedures

The workflow consists of the following steps:

1. Download NCBI taxonomy database (taxdump).
2. Download NCBI RefSeq assembly summary.
3. Sample genomes based on various properties and taxonomic information.
4. Download protein sequences associated with sampled genomes.
5. Compile local databases using DIAMOND and/or BLAST.

### Database files

File or directory | Description
--- | ---
`db.faa` | A single multi-Fasta file containing all protein sequences.
`genomes.tsv` | Metadata table for the sampled genomes.
`genome.map.gz` | Protein ID to genome ID mapping file.
`taxon.map.gz` | Protein ID to TaxID mapping file.
`lineages.txt` | Genome ID to Greengenes-style lineage mapping file.
`taxdump/` | A taxonomy database in the format of NCBI taxdump, but only contains TaxIDs relevant to the sampled genomes.
`diamond/` | (Optional) compiled DIAMOND database.
`blast/` | (Optional) compiled BLAST database.
`download/` | Original files downloaded from the NCBI server.

For a typical HGTector run, one will need either `diamond/` or `blast/` as the reference protein sequence database, plus `taxdump/` to inform taxonomic hierarchies. Example:

```bash
hgtector search -i sample.faa -o . -d <db_dir>/diamond/db -t <db_dir>/taxdump
hgtector analyze -i sample.tsv -o . -t <db_dir>/taxdump
```

The protein-to-TaxID map is already integrated into the compiled databases, so one does not need `taxon.map.gz` except for special, custom situations.

Feel free to delete (e.g., `download/`) or compress the intermediate files (e.g., `db.faa`) to save disk space.

### More examples

```bash
hgtector database -c bacteria,archaea -o . -t 1117,766 -e
```

This will include bacterial and archaeal genomes, but exclude ones classified as Cyanobacteria (TaxID: [1117](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?mode=Info&id=1117)) and Rickettsiales (TaxID: [766](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?mode=Info&id=780)). Useful for ruling out susceptible chloroplastic or mitochondrial signals.

```bash
hgtector database --genbank --sample 10 --rank phylum -o .
```

This will include GenBank genomes, and select up to 10 genomes per phylum. This small database will maximize the inclusion of the 100+ mostly uncultivated, understudied candidate phyla which has less curated genomes. Useful for exploring deep branches of life.

```bash
hgtector database -g gids.txt -o .
```

This will only download genomes specified in the file `gids.txt`. Useful for controlled tests.


### Break and resume

Should any of the download steps be interrupted by e.g., a network failure, one can resume the downloading process by re-executing the same command. The program will skip the already downloaded files in this new run. In some instances, one may need to manually remove the last file from the failed run (because that file may be corrupt), before re-running the program.

If one wants to overwrite downloaded files (e.g., upgrading), add `--overwrite` to the command.

### Manual compiling

The sampling & downloading steps (1-4) require extensive network traffics (usually several hours, if the bottleneck is not on the recipient side) but little local computing load; whereas the compling step (5) requires extensive local computing power, but no networking.

Therefore, it is a reasonable plan to only download database files without compiling. One can then manually compile databases using choice of programs.

For DIAMOND:

```bash
echo $'accession\taccession.version\ttaxid' > prot.accession2taxid
zcat taxon.map.gz | awk -v OFS='\t' '{split($1, a, "."); print a[1], $1, $2}' >> prot.accession2taxid

diamond makedb --threads 16 --in db.faa --taxonmap prot.accession2taxid --taxonnodes taxdump/nodes.dmp --taxonnames taxdump/names.dmp --db diamond/db

rm prot.accession2taxid
```

For BLAST:

```bash
gunzip -k taxon.map.gz

makeblastdb -dbtype prot -in db.faa -out blast/db -title db -parse_seqids -taxid_map taxon.map

rm taxon.map
```

### Other genome files

The `database` command only downloads protein sequences. If for other purposes you also want additional database files for the same genomes, for example, the GenBank files (those ending with `genomic.gbff.gz`), you can do this:

```bash
ext=genomic.gbff.gz
while IFS=$'\t' read -r -a a
do
    path=${a[@]: -1}
    wget -O ${a[0]}.${ext#*.} $path/${path##*/}_$ext
    break
done < <(tail -n+2 genomes.tsv)
```


## Command-line reference

### Default protocol

Option | Default | Description
--- | --- | ---
`--default` | - | Apply the default protocol for building the database (see above).

### Basic

Option | Default | Description
--- | --- | ---
`-o`, `--output` | - | Directory where the database will be stored.
`-c`, `--cats` | microbe | Categories of genomes to include. Adopted from NCBI [RefSeq](ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/). Options are: archaea, bacteria, fungi, invertebrate, plant, protozoa, vertebrate_mammalian, vertebrate_other, and viral. The default value "microbe" represents archaea, bacteria, fungi, and protozoa. Additionally, enter "all" to include all categories.

### Custom list

Option | Default | Description
--- | --- | ---
`-t`, `--taxids` | - | Provide a custom list of TaxIDs. Only genomes under these taxonomic groups will be included.
`-g`, `--genoids` | - | Provide a custom list of genome IDs. Only these genomes will be included. Valid genome IDs may be NCBI assembly accessions (e.g., "GCF_000123456.1"), with or without version number, or simply "G000123456".
`-e`, `--exclude` | - | Exclude instead of include taxonomic groups or genomes defined by the custom lists.

### Taxon sampling

Option | Default | Description
--- | --- | ---
`-s`, `--sample` | 0 | Sample up to this number of genomes per taxonomic group at the given rank. "0" is for all (disable sampling).
`-r`, `--rank` | species | Taxonomic rank at which subsampling will be performed.

### Genome sampling

Option | Default | Description
--- | --- | ---
`--genbank` | - | By default the program only downloads RefSeq genomes (`GCF`). This flag will let the program also download GenBank genomes (`GCA`). But RefSeq has higher priority than GenBank if the same genome is hosted by both catalogs.
`--complete` | - | Only include complete genomes, i.e., `assembly_level` is `Complete Genome` or `Chromosome`.
`--reference` | - | Add NCBI-defined reference genomes to selection (after taxon sampling).
`--representative` | - | Add NCBI-defined representative genomes to selection (after taxon sampling).

### Taxonomic filter

Option | Default | Description
--- | --- | ---
`--capital` | yes | Organism name must be capitalized.
`--block` | unknown,uncultured,<br>unidentified,unclassified,<br>unresolved,environmental,<br>plasmid,vector,<br>synthetic,phage | Organism name must not contain any of the following words (a comma-delimited string, or a file of one word per line)
`--latin` | no | Genomes must have Latinate species names. e.g., "_Vibrio cholerae_" is okay, but "_Vibrio_ sp. 123" is not.

### Download

Option | Default | Description
--- | --- | ---
`--overwrite` | - | Overwrite existing files with newly downloaded ones. Othewise use existing files whenever available
`--retries` | 3 | Maximum number of retries for downloading one file.
`--delay` | 10 | Seconds between two retries.
`--timeout` | 60 | Seconds before program gives up waiting.

### Compile

Option | Default | Description
--- | --- | ---
`--compile` | none | Compile downloaded protein sequences into a database using this program if available. Choices: diamond, blast, both, none
`--diamond` | diamond | diamond executable
`--makeblastdb` | makeblastdb | makeblastdb executable
`--threads` | 0 | Number of threads for diamond makedb (0 for all CPUs)
`--tmpdir` | - | Temporary directory for diamond makedb