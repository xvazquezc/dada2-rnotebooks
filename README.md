# dada2-rnotebooks

This repository contains R notebooks for processing amplicon data via [DADA2](https://github.com/benjjneb/dada2/) with some modifications.

This repository consists mainly of 3 `.Rmd` files:
- `dada2.Rmd`: core notebook. It performs most of the DADA2 functions with a few additions:
  - Cutadapt to remove priming regions and polyG tails.
  - Detect if the sequencing data uses a "classic" continuous Phred score or binned quality scores, and automatically apply the correct error modelling function. [See this issue for details](https://github.com/benjjneb/dada2/issues/1307).
  - Cluster identical sequences of different length with CD-HIT. Often derived from uneven primer trimming.
  - Renames the ASVs with `ASV_[:number:]` instead of having the sequence as identifier. [Based on AstroBioMike's DADA2 workflow](https://astrobiomike.github.io/amplicon/dada2_workflow_ex#assigning-taxonomy).
  - Export a `phyloseq` object to make it easier to continue downstream analysis.
- `report.Rmd`: for a quick overview of your dataset. Much of this script reuses code from [my old mothur-based pipeline OTUreporter](https://bitbucket.org/xvazquezc/otureporter/src). It generates a self-contained HTML with:
  - Read-tracking: plot and table of reads being left behind at each of the DADA2 (from running `dada2.Rmd`) and postprocessing steps (off-target filtering)
  - Rarefaction curve (interactive plot)
  - Alpha-diversity indexes (table).
  - Community composition:
    - Krona plots
    - Barplots at phylum, class, order and family levels as interactive and static plots.
  - Beta-diversity plots (both PCoA and NMDS):
    - Bray-Curtis
    - Unweighted UniFrac (if tree is provided)
    - Weighted UniFrac (if tree is provided)
  - Export a `phyloseq` object with the off-target taxa removed, and the reference tree, if included, too.
- `regex_filename_check.Rmd`: an accessory script to test the renaming mask for your files. The renaming step, i.e., cutting off the extra junk in the filenames, is quite in the middle of `dada2.Rmd` - and it's quite annoying to find at the end of the analysis that some of your samples ended up with the whole file name as the sample name. This script is provided to freely test your regEx before running `dada2.Rmd`.


## Requirements

### Software
The notebooks have only been tested in R under Linux (Ubuntu and Rocky Linux) with RStudio. The non-R requirements should work even if installed in a separate environment.

- R
  - `dada2` (>=1.36, or whatever includes the `makeBinnedQualErrfun()` function)
  - `ShortRead`
  - `magrittr`
  - `pacman`
  - `Biostrings`
  - `tibble`
  - `dplyr`
  - `tidyr`
  - `stringr`
  - `phylotools`
  - `phyloseq`
- [Cutadapt](https://github.com/marcelm/cutadapt/)
- [CD-HIT](https://github.com/weizhongli/cdhit)

To run the `report.Rmd`, you'll also need:
- R
  - `kableExtra`
  - `speedyseq`
  - `microbiome`
  - `ggplot2`
  - `plotly`
  - `vegan`
  - `purrr`
- [Krona](https://github.com/marbl/Krona/wiki)

### Databases

You should be able to use any DADA2-formatted databases as a reference. Below are my usual choices:

- 16S rRNA gene: [SILVA formatted for DADA2](https://benjjneb.github.io/dada2/training.html).
- 18S rRNA gene: [PR2 database](https://github.com/pr2database/pr2database/releases).
- ITS: UNITE releases 4 different resources with each version. I recommend to use the "All eukaryotes" that "Includes singletons set as RefS (in dynamic files)". [Download it from the UNITE resources page under "General FASTA release"](https://unite.ut.ee/repository.php).
  > **IMPORTANT:** Fungal ITS primers often amplify organisms other than Fungi. If you use the "Fungi" version, any non-fungal sequence will be misclassified as "Fungi; unknown phylum" even when they are not fungal, e.g. plant sequences. You definitely don't want that!


## Running

Before you start, you need to create a new R project and copy the required notebook in it. You also want to create a new folder in the project directory, e.g. `raw`, where you will deposit the `fastq.gz` files or symbolic links to them. Note that the files need to be directly located in the `raw` folder. Do not place the `fastq.gz` files within subfolders in `raw`.

Most of the important configurable parameters to run the notebooks are in the YAML headers, in the `params` subheading. Unless indicated otherwise, all the parameters must be quoted `""`:

- `raw_data:` project subfolder with the raw reads (default: `"raw"`).
- `fwd_seq:` forward primer sequence (e.g., `"AAACTYAAAKGAATTGRCGG"`).
- `rev_seq:` forward primer sequence (e.g., `"ACGGGCGGTGWGTRC"`).
- `fwd_pat:` forward file pattern, filename pattern to differentiated the forward reads from the reverse files (default: `"R1(_001){0,1}.fastq.gz"`). The default should match standard filenames from Illumina sequencing (e.g. `[sample]_L001_R1_001.fastq.gz`) as well as simplified names (e.g. `[sample]_R1.fastq.gz`).
- `rev_pat:` reverse file pattern, filename pattern to differentiated the reverse reads from the forward files (default: `"R2(_001){0,1}.fastq.gz"`). The default should match standard filenames from Illumina sequencing (e.g. `[sample]_L001_R2_001.fastq.gz`) as well as simplified names (e.g. `[sample]_R2.fastq.gz`).
- `fname_regex:` forward file name [regular expression (RegEx)](https://en.wikipedia.org/wiki/Regular_expression). RegEx for parsing the "Sample name" from the forward file name. Because the name extraction happens in the middle of the run and can be a bit tricky to get right, [I include an accessory Rmd to test with your files](regex_filename_check.Rmd) (e.g., `"-MEL17453A[:digit:]{1,}-AAHLCF3M5_S[:digit:]{1,}_L001_R1_001\\.fastq\\.gz"`).
  >Note: it's not a compulsory parameter and you can just indicate something like `"\\.fastq\\.gz"` to simply remove the file extension.
- `read_len:` raw amplicon read length, __do not quote the value__ (default: `300`).
- `isITS`: if you are processing ITS sequences, switch the value to `TRUE`. This will simply adjust the headers of the taxonomy table to values more appropriate to that taxonomy, __do not quote the value__  (default: `FALSE`).
- `cpus:` number of CPUs to use in the tasks that allow multithreading, __do not quote the value__ (default: `8`).
- `cutadapt_path:` path to Cutadapt's executable (e.g., `"/apps/z_install_tree/linux-rocky8-ivybridge/gcc-12.2.0/cutadapt-4.3-h7up35wj3xjo6pfjkkevxbkqd764obtw/bin/cutadapt"`).
- `cdhit_path:` path to the folder *containing* CD-HIT's binaries and accessory tools, __not the executable__ (e.g., `"/apps/z_install_tree/linux-rocky8-ivybridge/gcc-12.2.0/cdhit-4.8.1-d3fwnap7tmcfkp2h46bxgntfddriozt5/bin"`).
- `db_train:` path to the reference database file, (e.g., `"/srv/scratch/ferrari/utils/silva_nr99_v138.1_train_set.fa.gz"`).


Some of the most critical parameters when running DADA2 are those associated with the `filterAndTrim()` function. This function is in the `filterAndTrim_quality` code chunk. Here, I indicate some parameters that have worked well in the past:

- 16S rRNA gene:
    - **515-Y** (5\'-GTGYCAGCMGCCGCGGTAA-3\') and **806RB** (5\'-GGACTACNVGGGTWTCTAAT-3\'): usual Bacteria/Archaea 16S v4 primers. 2x250 bp. Params: `maxEE = c(3, 5), truncLen = c(225,200), truncQ = 2, minLen = 150`.
    - **926F** (5\'-AAACTYAAAKGAATTGRCGG-3\') and **1392wR** (5\'-ACGGGCGGTGWGTRC-3\'): Universal 16S/18S - v6-v8 (based on 16S). It requires 2x300 bp run. Params: `maxEE = c(3, 5), truncLen = c(275,230), truncQ = 2, minLen = 150`.
    - **"341F"** (5\'-CCTAYGGGRBGCASCAG-3\') and **"806R"** (5\'-GGACTACNNGGGTATCTAAT-3\'): Novogene's universal 16S v3-4 primers. Novogene does 2x250 bp runs for this set but a 2x300 bp would be more appropriate. Params (for 2x250): `maxEE = c(2, 3), truncLen = c(220,220), truncQ = 2, minLen = 150`.
- 18S rRNA gene:
    - **1391F** (5\'-GTACACACCGCCCGTC-3\') and **EukB** (5\'-TGATCCTTCTGCAGGTTCACCTAC-3\'): universal eukaryotic 18S v9 primers. Usually ~160-170 nt, [unusually long ones in some Archaeaplastida and Opisthokonta lineages](https://onlinelibrary.wiley.com/doi/10.1111/1755-0998.13465). Params (tested so far, `truncRight` might be better than `truncLen`): `maxEE = c(1, 1), truncLen = c(120,110), truncQ = 2, minLen = 150`.
- Internal Transcribed Spacers (ITS):
    - **fITS7** (5\'-GTGAATCATCGAATCTTTG-3\') and **ITS4** (5\'-TCCTCCGCTTATTGATATGC-3\'): Fungal ITS2. 2x250 bp run. Params: `maxEE = c(1, 3), truncQ = 2, minLen = 50` (don't use `truncLen` to avoid discarding reads from short ITS2 regions).
    - **ITS1 region** or **full-length ITS**: actual amplicons might be way too long for some to be merged after quality trimming and primer removal, albeit they are _bona fide_ sequences. If you don't want to possibly lose that part of the community, you can either use the `justConcatenate = TRUE` option in `mergePairs()` or just use the forward reads. If you only keep the forward reads (you need to change the code) these are some suitable params: `maxEE = 2, trimLeft = 25, truncQ = 5, minLen = 50`.