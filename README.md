# README

This repository contains an ITS metabarcode pipeline that can be used to process Illumina MiSeq reads.  SCVUIU is an acronym that stands for the names of the programs/algorithms/datasets used here: S = SEQPREP, C = CUTADAPT, V = VSEARCH, U = USEARCH-unoise3, I = ITSx-ITS extractor, U = UNITE ITS reference set used with the RDP classifier.  The dataflow and scripts for the most current release can be downloaded from https://github.com/terrimporter/SCVUIU_ITS_metabarcode_pipeline/releases

## Overview

[Part 1 - Pair forward and reverse reads](#part-1---pair-forward-and-reverse-reads)  
[Part 2 - Trim primers](#part-2---trim-primers)  
[Part 3 - Dereplicate reads](#part-3---dereplicate-reads)  
[Part 4 - Denoise reads](#part-4---denoise-reads)  
[Part 5 - Generate ESV table](#part-5---generate-esv-table)  
[Part 6 Extract ITS2 region](#part-6---extract-its2-region)  
[Part 7 - Taxonomic assignment](#part-7---taxonomic-assignment)  
[Implementation notes](#implementation-notes)  
[References](#references)  
[Acknowledgements](#acknowledgements)  

## Part 1 - Pair forward and reverse reads

I used SEQPREP (available from https://github.com/jstjohn/SeqPrep ) to pair forward and reverse reads using the default settings except that we required a minimum Phred score of 20 at the ends and a minimum overlap of 25 bp.  This can be automated to run on a batch of infiles using a shell script or GNU parallel.

~~~linux
seqprep -f R1.fq -r R2.fq -1 R1.out -2 R2.out -q 20 -s basename.paired.fastq.gz -E basename.paired.aln.gz -o 25
~~~

## Part 2 - Trim primers

I used CUTADAPT v1.10 to sequentially remove primers with the default settings except that we required a minimum of 150 bp after primer removal, required a minimum Phred score of 20 at the ends, and allowing a maximum of 3 N's.  The -g flag trims the primer off the 5' end of paired reads.

~~~linux
ls | grep .fastq | parallel -j 15 "cutadapt -g <INSERT FORWARD PRIMER SEQ> -m 150 -q 20,20 --max-n=3 --discard-untrimmed {} > {}.Ftrimmed.fastq"
~~~

The -a flag trims the primer off the 3' end of paired reads.

~~~linux
ls | grep .Ftrimmed.fastq | parallel -j 15 "cutadapt -a <INSERT REVERSE COMPLEMENTED PRIMER SEQ> -m 150 -q 20,20 --max-n=3 --discard-untrimmed {} > {}.Rtrimmed"
~~~

## Part 3 - Dereplicate reads

I add the sample name to FASTA headers using the rename_all_fastas command that links to the run_rename_fasta.sh script.  Therein, the rename_fasta command links to the rename_fasta.plx script.  This step is necessary for proper OTU table generation in USEARCH.  This command should be run in a directory of FASTA files.  This step also produces a single concatenated FASTA file to permit a GLOBAL sample analysis in VSEARCH and USEARCH.  Then I change any dashes in the FASTA headers to underscores so that the OTU table is generated properly in USEARCH using the program vi Editor.  Read dereplication is carried out with VSEARCH using the default parameters but tracking the number of clustered reads with the --sizein and --sizeout flags.

~~~linux
rename_all_fastas
vi -c "%s/-/_/g" -c "wq" cat.fasta
vsearch --threads 10 --derep_fulllength cat.fasta --output cat.uniques --sizein --sizeout
~~~

## Part 4 - Denoise reads

I used USEARCH v9.1.13 with the unoise2 algorithm with the default settings except that I specified a minimum OTU size of 3.  This basically removes singletons and doubletons from the OTU table.

~~~linux
usearch9 -unoise2 cat.uniques -fastaout cat.denoised -minampsize 3
~~~

## Part 5 - Generate ESV table

An OTU table is generated in USEARCH9 with the default settings and specifying an identity cutoff of 97% sequence similarity.

~~~linux
usearch9 -usearch_global cat.fasta -db cat.denoised.sort2.centroids3 -strand plus -id 0.97 -otuabout cat.fasta.table97
~~~

## Part 6 - Extract ITS2 region

The leading and trailing regions of the ITS2 region are retrieved using the ITSx program available from http://microbiology.se/software/itsx/ (Bengtsson-Palme et al., 2013).  Be sure to adjust the --cpu flag according to how many cpus you want to use.

~~~linux
ITSx -i cat.denoised.sort2.centroids3 -o cat.denoised.sort2.centroids3 --cpu 15
~~~

## Part 7 - Taxonomic assignment

Taxonomic assignments were performed using the Ribosomal Database Project (RDP) Classifier (Wang et al., 2007).  Read counts from the OTU table were mapped to the RDP classifier taxonomic assignments using add_abundance_to_rdp_out3.plx .  The ITS reference set is available with the RDP Classifier and is called with the -g flag.  The 18S v2.0 reference set is available at https://github.com/terrimporter/18SClassifier/releases.  The unmodified CO1 Classifier v1.0 reference database is available from https://github.com/terrimporter/CO1Classifier/releases .  The ammended CO1 Classifier v2.1 is available as a release from this repository at https://github.com/terrimporter/JesseHoage2018/releases

~~~linux
#Classify the ITS sequences
java -Xmx8g -jar /path/to/rdp_classifier_2.12/dist/classifier.jar classify -g fungalits_unite -o rdp.out cat.denoised.sort2.centroids3.ITS2.fasta

#Map read counts from OTU table to the RDP taxonomic assignments
#Do this individually for each marker
perl add_abundance_to_rdp_out3.plx cat.fasta.table97 rdp.out
~~~

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

### Summary statistics

At each major step of the data flow described above, fastq files are converted to FASTA files using MOTHUR v1.38.1.  A perl script is then used to calculate sequence statistics including total number of sequences and min/max/mean/median/mode sequence length.  The command stats_R1 runs the Bash script run_fastastats_parallel_R1.sh where the pattern to search for the infiles is hard-coded.  Therein, stats links to the Perl script fasta_stats_parallel.plx.  This script requires the Perl module Statistics::Lite that can be obtained from CPAN.  The command stats_R2 runs the Bash script run_fastastats_parallelR2.sh where the pattern to search for the infiles is hard-coded.  The command stats_fasta links to run_fastastats_parallel_fasta.sh .  The stats_uniques command links to the run_fastastats_parallel_uniques.sh script.  The command read_counts_uniques links to the get_read_counts_uniques.sh .  The stats_denoised command links to the run_fastastats_parallel_denoised.sh . The read_count_denoised command links to the get_read_counts_denoised.sh . The stats_centroids3 command links to the run_fastastats_parallel_centroids3.sh . 

~~~linux
#Get summary stats for fastq reads
ls | grep .fastq | parallel -j 23 "mothur '#fastq.info(fast1={})'"
stats_R1
stats_R2

#Get summary stats for FASTA files
stats_fasta

#Get summary stats for the dereplicated OTUs
stats_uniques

#Get the number of reads contained in the dereplicated OTUs
read_counts_uniques

#Get the denoised ESV summary stats and read counts
stats_denoised
read_count_denoised

#Get the ITS2 summary stats
stats_fasta
~~~

## References

Bengtsson-Palme et al. (2013) Improved software detection and extraction of ITS1 and ITS2 from ribosomal ITS sequences of fungi and other eukaryotes for analysis of environmental sequencing data.  Methods in Ecology and Evolution, 4: 914.  Available from http://microbiology.se/software/itsx/

Tange, O. (2011). GNU Parallel - The Command-Line Power Tool. ;;Login: The USENIX Magazine, February, 42–47.  Available from: https://www.gnu.org/software/parallel/

Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07 .  Available from: https://sourceforge.net/projects/rdp-classifier/

## Acknowledgements

I would like to acknowledge funding from the Government of Canada through the Genomics Research and Development Initiative, EcoBiomcis Project.
