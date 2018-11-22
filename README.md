# README

This repository contains an ITS metabarcode pipeline that can be used to process Illumina MiSeq reads.  SCVUIU is an acronym that stands for the names of the programs/algorithms/datasets used here: S = SEQPREP, C = CUTADAPT, V = VSEARCH, U = USEARCH-unoise3, I = ITSx-ITS extractor, U = UNITE ITS reference set used with the RDP classifier.  The dataflow and scripts for the most current release can be downloaded from https://github.com/terrimporter/SCVUIU_ITS_metabarcode_pipeline/releases

## Overview

[Part I - Link to raw files](#part-i---link-to-raw-files)  
[Part II - Forward and reverse read number check](#part-ii---forward-and-reverse-read-number-check)  
[Part III - Read pairing](#part-iii---read-pairing)  
[Part IV - Primer trimming](#part-iv---primer-trimming)  
[Part V - Dereplicate reads](#part-v---dereplicate-reads)  
[Part VI - Denoising](#part-vi---denoising)  
[Part VII - Extract ITS2 region](#part-vii---extract-its2-region)  
[Part VIII - Taxonomic assignment](#part-viii---taxonomic-assignment)  

[Implementation notes](#implementation-notes)  
[References](#references)  
[Acknowledgements](#acknowledgements)  

## Part I - Link to raw files

This pipeline is meant to process Illumina paired-end reads from COI metabarcoding. To save space in my directory, I create symbolic links to the raw .gz files. The command linkfiles calls the script link_files.sh

~~~linux
linkfiles
~~~

## Part II - Forward and reverse read number check

I make sure that the number of reads in the forward R1 files are the same as those in the reverwe R2 files. The command gz_stats calls the script run_bash_fastq_gz_stats.sh. Therein the stats2 command links to the fastq_gz_stats.plx script. The filename suffix that targets R1 and R2 files needs to be supplied as an argument.

~~~linux
gz_stats R1.fq.gz > R1.stats
gz_stats R2.fz.gz > R2.stats
~~~

## Part III - Read pairing

I use SEQPREP (available from https://github.com/jstjohn/SeqPrep ) to pair forward and reverse reads using the default settings except that we required a minimum Phred score of 20 at the ends and a minimum overlap of 25 bp.  The command pair runs the script runseqprep_gz.sh .   I check the read stats by running the gz_stats command described in Part II.

~~~linux
pair _R1.fq.gz _R2.fq.gz
gz_stats gz > paired.stats
~~~

## Part IV - Primer trimming

I used CUTADAPT v1.18 to sequentially remove primers with the default settings except that we required a minimum of 150 bp after primer removal, required a minimum Phred score of 20 at the ends, and allowing a maximum of 3 N's (Martin, 2011).  I run CUTADAPT with GNU parallel using as many cores as possible (Tang, 2011).  The -g flag trims the primer off the 5' end of paired reads. I use default settings but require a minimum length after trimming of at least 150 bp, minimum read quality of Phred 20 at the ends of the sequences, and I allow a maximum of 3 N's. I get read stats by running the gz_stats command described in Part II. The CUTADAPT -a flag trims the primer off the ' end of paired reads and the primer sequence should be reverse-complemented.  CUTADAPT will automatically detect compressed fastq.gz files for reading and will convert these to .fasta.gz files based on the file extensions provided. I get read stats by running the fasta_gz_stats command that calls the run_bash_fasta_gz_stats.sh script. Therein the stats3 command links to the fasta_gz_stats.plx script.

~~~linux
ls | grep .fastq.gz | parallel -j 20 "cutadapt -g <INSERT FORWARD PRIMER SEQ> -m 150 -q 20,20 --max-n=3 --discard-untrimmed -o {}.Ftrimmed.fastq.gz {}"
gz_stats gz > Ftrimmed.stats
ls | grep .Ftrimmed.fastq | parallel -j 20 "cutadapt -a <INSERT REVERSE COMPLEMENTED PRIMER SEQ> -m 150 -q 20,20 --max-n=3 --discard-untrimmed -o {}.Rtrimmed.fasta.gz {}"
fasta_gz_stats gz > Rtrimmed.stats
~~~

## Part V - Dereplicate reads

I prepare the files for dereplication by adding sample names parsed from the filenames to the fasta headers using the rename_all_fastas command that calls the run_rename_fasta.sh. Therein the rename_fasta command calls the rename_fasta_gzip.plx script. The results are concatenated and compressed. The outfile is cat.fasta.gz . I change all dashes with underscores in the fasta files using vi. This large file is dereplicated with VSEARCHv2.9.1 (Rognes et al., 2016). I use the default settings with the --sizein --sizeout flags to track the number of reads in each cluster. I get read stats on the unique sequences using the stats_uniques command that calls the run_fastastats_parallel_uniques.sh script. Therein the stats command links to fasta_stats_parallel.plx . I count the total number of reads that were processed using the read_count_uniques command that calls the get_read_counts_uniques.sh script.

~~~linux
rename_all_fastas Rtrimmed.fasta.gz
vi -c "%s/-/_/g" -c "wq" cat.fasta.gz
vsearch --threads 10 --derep_fulllength cat.fasta --output cat.uniques --sizein --sizeout
stats_uniques
read_count_uniques
~~~

## Part VI - Denoising

I denoise the reads using USEARCH v10.0.240 with the UNOISE3 algorithm (Edgar, 2016). With this program, denoising involves correcting sequences with putative sequencing errors, removing PhiX and putative chimeric sequences, as well as low frequency reads (just singletons and doubletons here). This step can take quite a while to run for large files and I like to submit as a job on its own or use linux screen when working interactively so that I can detach the screen. To account for a bug in USEARCH10, the automatically generated 'Zotu' in the FASTA header needs to be changed to 'Otu' for the ESV/OTU table to be generated correctly in the next step. I get ESV stats using stats_denoised that links to run_fastastats_parallel_denoised.sh. Therein the command stats links to fasta_stats_parallel.plx . I generate an ESV/OTU table using VSEARCH by mapping the primer-trimmed reads in cat.fasta to the ESVs in cat.denoised using an identity cutoff of 1.0 .

~~~linux
usearch10 -unoise3 cat.uniques -zotus cat.denoised -minsize 3 > log
vi -c "%s/>Zotu/>Otu/g" -c "wq" cat.denoised
stats_denoised
vsearch  --usearch_global cat.fasta.gz --db cat.denoised --id 1.0 --otutabout cat.fasta.table --threads 20
~~~

## Part VII - Extract ITS2 region

The leading and trailing regions of the ITS2 region are retrieved using the ITSx v1.0.11 program available from http://microbiology.se/software/itsx/ (Bengtsson-Palme et al., 2013).  Be sure to adjust the --cpu flag according to how many cpus you want to use.

~~~linux
ITSx -i cat.denoised -o cat.denoised --cpu 15
~~~

## Part VIII - Taxonomic assignment

Taxonomic assignments were performed using the Ribosomal Database Project (RDP) Classifier v2.12 (Wang et al., 2007).  The ITS reference set is available with the RDP Classifier and is called with the -g flag.  Read counts from the ESV x sample table were mapped to the RDP classifier taxonomic assignments using add_abundance_to_rdp_out4.plx .  I also append the marker/primer name to each GlobalESV id.  This is important if down the line you end up working with multiple markers.

You can filter for high confidence taxonomic assignments by using a 0.80 bootstrap support cutoff for long queries or a 0.50 cutoff for queries shorter than 250 bp as recommended on the RDP Classifier website https://rdp.cme.msu.edu/classifier/classifier.jsp .

~~~linux
#Classify the ITS sequences
java -Xmx8g -jar /path/to/rdp_classifier_2.12/dist/classifier.jar classify -g fungalits_unite -o rdp.out cat.denoised.ITS2.fasta

#Map read counts from OTU table to the RDP taxonomic assignments
perl add_abundance_to_rdp_out4.plx cat.denoised.table rdp.out

#Prefix each GlobalESV with the marker/primer name
vi -c "%s/^/Marker_/g" -c "wq" rdp.out
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

## References

Bengtsson-Palme et al. (2013) Improved software detection and extraction of ITS1 and ITS2 from ribosomal ITS sequences of fungi and other eukaryotes for analysis of environmental sequencing data.  Methods in Ecology and Evolution, 4: 914.  Available from http://microbiology.se/software/itsx/

Edgar, R. C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. BioRxiv. doi:10.1101/081257 .  Available from: https://www.drive5.com/usearch/

Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet. Journal, 17(1), pp–10.  Available from: http://cutadapt.readthedocs.io/en/stable/index.html

Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. PeerJ, 4, e2584. doi:10.7717/peerj.2584 .  Available from:  https://github.com/torognes/vsearch

Tange, O. (2011). GNU Parallel - The Command-Line Power Tool. ;;Login: The USENIX Magazine, February, 42–47.  Available from: https://www.gnu.org/software/parallel/

Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07 .  Available from: https://sourceforge.net/projects/rdp-classifier/

## Acknowledgements

I would like to acknowledge funding from the Government of Canada through the Genomics Research and Development Initiative, EcoBiomcis Project.
