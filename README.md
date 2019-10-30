# README

This repository contains an ITS metabarcode pipeline that can be used to process Illumina MiSeq reads.  SCVUIU is an acronym that stands for the names of the programs/algorithms/datasets used here: S = SEQPREP, C = CUTADAPT, V = VSEARCH, U = USEARCH-unoise3, I = ITSx-ITS extractor, U = UNITE ITS reference set used with the RDP classifier.  

This data flow has been developed using a conda environment and snakemake pipeline for improved reproducibility. It will be updated on a regular basis so check for the latest version at https://github.com/terrimporter/SCVUIU_ITS_metabarcode_pipeline/releases

## Overview

[Standard pipeline](#standard-pipeline)  

[Alternate pipeline](#alternate-pipeline)  

[Implementation notes](#implementation-notes)  

[References](#references)  

[Acknowledgements](#acknowledgements)  

## Standard pipeline

### Overview of the standard pipeline

If you are comfortable reading code, read through the snakefile to see how the pipeline runs, and which programs and versions are used.

#### A brief overview:

Raw paired-end reads are merged using SEQPREP v1.3.2 from bioconda (St. John, 2016).  This step looks for a minimum Phred quality score of 20 in the overlap region, requires at least 25bp overlap.

Primers are trimmed in two steps using CUTADAPT v2.6 from bioconda (Martin, 2011).  This step looks for a minimum Phred quality score of at least 20 at the ends, forward primer is trimmed first, no more than 3 N's allowed, trimmed reads need to be at least 150 bp, untrimmed reads are discarded.  The output from the first step, is used as in put for the second step.  This step looks for a minimum Phred quality score of at least 20 at the ends, the reverse primer is trimmed, no more than 3 N's allowed, trimmed reads need to be at least 150 bp, untrimmed reads are discarded.

Files are reformatted and samples are combined for a global analysis.

Reads are dereplicated (only unique sequences are retained) using VSEARCH v2.13.6 from bioconda (Rognes et al., 2016).

Denoised exact sequence variants (ESVs) are generated using USEARCH v11.0.667 with the unoise3 algorithm (Edgar, 2016).  This step removes any PhiX contamination, putative chimeric sequences, sequences with predicted errors, and rare sequences.  This step produces zero-radius OTUs (Zotus) also referred to commonly as amplicon sequence variants (ASVs), ESVs, or 100% operational taxonomic unit (OTU) clusters.  Here, we define rare sequences to be sequence clusters containing only one or two reads (singletons and doubletons) and these are removed as 'noise'.

An ESV table that tracks read number for each ESV in each sample is generated with VSEARCH.

Conserved rRNA gene regions (LSU, 5.8S, or SSU) are removed using the ITSx extractor v1.1b, isolating the internal transcribed DNA spacer regions (ITS1 and/or ITS2) for subsequent taxonomic assignment (Bengtsson-Palme et al., 2013).

ITS taxonomic assignments are made using the Ribosomal Database classifier v2.12 (RDP classifier) available from https://sourceforge.net/projects/rdp-classifier/ (Wang et al., 2007) using the ITS-UNITE reference dataset that comes with the classifier.

The final output is reformatted to add read numbers for each sample, and column headers to improve readability, and reformats the ESV ids so they match those in the ESV table.

Read and ESV statistics are provided for various steps of the program are also provided.

### Prepare your environment to run the pipeline

1. This pipeline includes a conda environment that provides most of the programs needed to run this pipeline (SNAKEMAKE, SEQPREP, CUTADAPT, VSEARCH, etc.).

```linux
# Create the environment from the provided environment.yml file
conda env create -f environment.yml

# Activate the environment
conda activate myenv
```
2. The pipeline requires commercial software for the denoising step.  A free 32-bit version of USEARCH v11.0.667 can be obtained from https://drive5.com/usearch/download.html .  Be sure to put the program in your PATH, ex. ~/bin .  Make it executable and rename it to simply usearch11.

```linux
mv usearch11.0.667_i86linux32 ~/bin/.
cd ~/bin
chmod 755 usearch11.0.667_i86linux32
mv usearch11.0.667_i86linux32 usearch11
```

3. The pipeline also requires the RDP classifier for the taxonomic assignment step.  Although the RDP classifier v2.2 is available through conda, a newer v2.12 is available form SourceForge at https://sourceforge.net/projects/rdp-classifier/ .  Download it and take note of where the classifier.jar file is as this needs to be added to config.yaml .

The RDP classifier comes with the training sets to classify fungal ITS sequences.

```linux
RDP:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"
    g: "fungalits_unite"
```

4. In most cases, your raw paired-end Illumina reads can go into a directory called 'data' which should be placed in the same directory as the other files that come with this pipeline.

```linux
# Create a new directory to hold your raw data
mkdir data
```

5. Please go through the config.yaml file and edit directory names, filename patterns, etc. as necessary to work with your filenames.

6. Be sure to edit the first line of each Perl script (shebang) in the perl_scripts directory to point to where Perl is installed.

```linux
# The usual shebang if you already have Perl installed
#!/usr/bin/perl

# Alternate shebang if you want to run perl using the conda environment (edit this)
#!/path/to/miniconda3/envs/myenv/bin/perl
```

### Run the standard pipeline

Run snakemake by indicating the number of jobs or cores that are available to run the whole pipeline.  

```linux
snakemake --jobs 24 --snakefile snakefile --configfile config.yaml
```

When you are done, deactivate the conda environment:

```linux
conda deactivate
```

## Alternate pipeline

This section describes modification to the standard pipeline described above when you get a message from 32-bit USEARCH that you have exceeded memory availble.  Instead of processing all the reads in one go, you can denoise each run on its own to keep file sizes small.

1. Instead of putting all raw read files in a directory called 'data', put them in their own directories according to run, ex. run1.  Edit the 'dir' variable in the config_alt_1.yaml file as follows:

```linux
raw: "run1"
```

2. The output directory also needs to be edited in the config_alt_1.yaml file:

```linux
dir: "run1_out"
```

3. Please go through the config_alt_1.yaml file and edit directory names, filename patterns, etc. as necessary to work with your filenames.

4. Run snakemake with the first alternate snakefile as follows, be sure to indicate the number of jobs/cores available to run the whole pipeline.

```linux
snakemake --jobs 24 --snakefile snakefile_alt_1 --configfile config_alt.yaml
```

5. Run steps 1-4 for each run directory, ex. run1, run2, run3, etc.

6. Combine and dereplicate the denoised ESVs from each run and put them in a directory named after the amplicon, for example:

```linux
# Make new directory
mkdir ITS2

# Add version number to denoised sequence headers to keep them unique after denoised data from each run is combined
sed 's/>Zotu[[:digit:]]\{1,6\}/&.1/g' run1_out/cat.denoised > run1_out/cat.denoised1
sed 's/>Zotu[[:digit:]]\{1,6\}/&.2/g' run2_out/cat.denoised > run2_out/cat.denoised2
sed 's/>Zotu[[:digit:]]\{1,6\}/&.3/g' run3_out/cat.denoised > run3_out/cat.denoised3

# Combine the denoised ESVs from each run
cat run1_out/cat.denoised run2_out/cat.denoised run3_out/cat.denoised > ITS2/cat.denoised.tmp

# Dereplicate the denoised ESVs
vsearch --derep_fulllength ITS2/cat.denoised.tmp --output ITS2/cat.denoised --sizein --sizeout --log ITS2/derep.log
```

7. Combine the primer trimmed reads frmo each run and put them in a directory named after the amplicon, for example:

```linux
# Combine the primer trimmed reads from each run
cat run1_out/cat.fasta.gz run2_out/cat.fasta.gz run3_out/cat.fasta.gz > ITS2/cat.fasta.gz
```

7. Edit the config.yaml 'dir' variable and the 'SED' variable, leave the rest of the variables as is (most of them won't be used here anyways):

```linux
dir: "ITS2"
...
SED: 's/^/ITS2_/g'
```

8. Continue with the second alternate snakelake pipeline, be sure to edit the number of jobs/cores available to run the whole pipeline.

```linux
snakemake --jobs 24 --snakefile snakefile_alt_2 --configfile config.yaml
```

9. When you are done, deactivate the conda environment:

```linux
conda deactivate
```

## Implementation notes

Shell scripts are written for Bash. Other scripts are written in Perl and may require additional libraries that are indicated near the top of the script when needed and these can be obtained from CPAN.

To keep the dataflow here as clear as possible, I have ommitted file renaming and clean-up steps.  I also use shortcuts to link to scripts as described above in numerous places.  This is only helpful if you will be running this pipeline often.  I describe, in general, how I like to do this here:

### File clean-up

At every step, I place outfiles into their own directory, then cd into that directory.  I also delete any extraneous outfiles that may have been generated but are not used in subsequent steps to save disk space.

### Symbolic links

Instead of continually traversing nested directories to get to files, I create symbolic links to target directories in a top level directory.  Symbolic links can also be placed in your ~/bin directory that point to scripts that reside elsewhere on your system.  So long as those scripts are executable (e.x. chmod 755 script.plx) then the shortcut will also be executable without having to type out the complete path or copy and pasting the script into the current directory.

~~~linux
ln -s /path/to/target/directory shortcutName
ln -s /path/to/script/script.sh commandName
~~~

## References

Bengtsson-Palme et al. (2013) Improved software detection and extraction of ITS1 and ITS2 from ribosomal ITS sequences of fungi and other eukaryotes for analysis of environmental sequencing data.  Methods in Ecology and Evolution, 4: 914.  Available from http://microbiology.se/software/itsx/

Edgar, R. C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. BioRxiv. doi:10.1101/081257 .  Available from: https://www.drive5.com/usearch/

Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet. Journal, 17(1), pp–10.  Available from: http://cutadapt.readthedocs.io/en/stable/index.html

Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. PeerJ, 4, e2584. doi:10.7717/peerj.2584 .  Available from:  https://github.com/torognes/vsearch

Tange, O. (2011). GNU Parallel - The Command-Line Power Tool. ;;Login: The USENIX Magazine, February, 42–47.  Available from: https://www.gnu.org/software/parallel/

Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07 .  Available from: https://sourceforge.net/projects/rdp-classifier/

## Acknowledgements

I would like to acknowledge funding from the Government of Canada through the Genomics Research and Development Initiative, Ecobiomcis Project.
