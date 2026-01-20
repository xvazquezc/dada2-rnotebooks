# dada2-rnotebooks

This repository contains R notebooks for processing amplicon data via [DADA2](https://github.com/benjjneb/dada2/) with some modifications.



## Requirements

The notebooks have only been tested in R under Linux (Ubuntu and Rocky Linux) with RStudio.

- R
  - `dada2`
  - `ShortRead`
  - `magrittr`
  - `Biostrings`
  - `tibble`
  - `dplyr`
  - `tidyr`
  - `stringr`
  - `phylotools`
  - `phyloseq`
- [Cutadapt](https://github.com/marcelm/cutadapt/)
- [CD-HIT](https://github.com/weizhongli/cdhit)



## Running

Before you start, you need to create a new R project and copy the required notebook in it. You also want to create a new folder in the project directory, e.g. `raw`, where you will deposit the `fastq.gz` files or symbolic links to them. Note that the files need to be directly located in the `raw` folder. Do not place the `fastq.gz` files within subfolders in `raw`.

Most of the important configurable parameters to run the notebooks are in the YAML headers, in the `params` subheading. Unless indicated otherwise, all the parameters must be quoted `""`:

- `raw_data:` project subfolder with the raw reads (default: `"raw"`).
- `fwd_seq:` forward primer sequence (e.g., `"AAACTYAAAKGAATTGRCGG"`).
- `rev_seq:` forward primer sequence (e.g., `"ACGGGCGGTGWGTRC"`).
- `fwd_pat:` forward file pattern, filename pattern to differentiated the forward reads from the reverse files (default: `"_L001_R1_001.fastq.gz"`).
- `rev_pat:` reverse file pattern, filename pattern to differentiated the reverse reads from the forward files (default: `"_L001_R2_001.fastq.gz"`).
- `fname_regex:` forward file name [regular expression (RegEx)](https://en.wikipedia.org/wiki/Regular_expression). RegEx for parsing the "Sample name" from the forward file name. Because the name extraction happens in the middle of the run and can be a bit tricky to get right, [I include an accessory Rmd to test with your files](regex_filename_check.Rmd) (e.g., `"-MEL17453A[:digit:]{1,}-AAHLCF3M5_S[:digit:]{1,}_L001_R1_001\\.fastq\\.gz"`).
  >Note: it's not a compulsory parameter and you can just indicate something like `"\\.fastq\\.gz"` to simply remove the file extension.
- `read_len:` raw amplicon read length, __do not quote the value__ (default: `300`).
- `cpus:` number of CPUs to use in the tasks that allow multithreading, __do not quote the value__ (default: `8`).
- `cutadapt_path:` path to Cutadapt's executable (e.g., `"/apps/z_install_tree/linux-rocky8-ivybridge/gcc-12.2.0/cutadapt-4.3-h7up35wj3xjo6pfjkkevxbkqd764obtw/bin/cutadapt"`).
- `cdhit_path:` path to the folder *containing* CD-HIT's binaries and accessory tools, __not the executable__ (e.g., `"/apps/z_install_tree/linux-rocky8-ivybridge/gcc-12.2.0/cdhit-4.8.1-d3fwnap7tmcfkp2h46bxgntfddriozt5/bin"`).
- `db_train:` path to the reference database file, (e.g., `"/srv/scratch/ferrari/utils/silva_nr99_v138.1_train_set.fa.gz"`).


Some of the most critical parameters when running DADA2 are those associated with the `filterAndTrim()` function. This function is in the `filterAndTrim_quality` code chunk. Here, I indicate some parameters that have worked well in the past:

- **515-Y** (5\'-GTGYCAGCMGCCGCGGTAA-3\') and **806RB** (5\'-GGACTACNVGGGTWTCTAAT-3\'): usual 16S v4 primers. 2x250. Params: `maxEE = c(3, 5), truncLen = c(225,200), truncQ = 2, minLen = 150`.
- **926F** (5\'-AAACTYAAAKGAATTGRCGG-3\') and **1392wR** (5\'-ACGGGCGGTGWGTRC-3\'): Universal 16S/18S - v6-v8 (based on 16S). It requires 2x300 run. Params: `maxEE = c(3, 5), truncLen = c(275,230), truncQ = 2, minLen = 150`.
- **fITS7** (5\'-GTGAATCATCGAATCTTTG-3\') and **ITS4** (5\'-TCCTCCGCTTATTGATATGC-3\'): Fungal ITS2. 2x250 run. Params: `maxEE = c(1, 3), truncQ = 2, minLen = 50` (don't use `truncLen` to avoid discarding reads from short ITS2 regions).
- **ITS1 region** or **full-length ITS**: actual amplicons might be way too long for some to be merged after quality trimming and primer removal, albeit they are _bona fide_ sequences. If you don't want to possibly lose that part of the community, you can either use the `justConcatenate = TRUE` option in `mergePairs()` or just use the forward reads. If you only keep the forward reads (you need to change the code) these are some suitable params: `maxEE = 2, trimLeft = 25, truncQ = 5, minLen = 50`.