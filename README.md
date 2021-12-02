# AutonomousAmpliconAssigner

AutonomousAmpliconAssigner (AAA) processes multiplexed amplicon sequence data. AAA takes as input demultiplexed, adapter-trimmed, and FLASh-joined FASTQ files, and calls consensus sequences based on read length distributions. It outputs aligned multi-FASTA formatted files (1 per gene), that can be used directly for gene-tree analysis or concatenated for an "all data" approach, and summary information.


## Installation

#### Install Anaconda distribution of python

[Anaconda download page](https://www.continuum.io/downloads)

#### Download this git repo
Using git:
```
git https://github.com/jrdupuis/AutonomousAmpliconAssigner.git
cd AutonomousAmpliconAssigner
```
#### Create and activate Anaconda environment: (note, env is installed in the current dir in ./AAA_env)
```
./create_envs.sh
conda activate ./AAA_env
```

#### Download Data
A toy dataset from [Dupuis et al. (2018)](https://onlinelibrary.wiley.com/doi/abs/10.1111/1755-0998.12783) is included in this repo ([here](https://github.com/jrdupuis/AutonomousAmpliconAssigner/tree/main/input_data)). It contains a single specimen's data for XXX loci.

## Usage: Detailed
#### config.toml
The file `config.toml` contains various parameters and data paths for running all three parts of this pipeline. Notably, this includes 2 lists of species names (spelled as in the output of the ortholog prediction): one of all species, and one of species with "high quality annotations". The latter are used to predict exon/intron boundaries for the remaining species. This config file also contains paths to the various inputs/intermediates/outputs for each of the three parts. Only the inputs directories are required to run the scripts (intermediate & output dirs will be generated automatically). If using all steps of this pipeline, we suggest using the data structure that is used in this GitHub repo, which is:


Part03 handles the post-sequencing data processing, and calling consensus sequences. The input for this script is adapter-trimmed and demultiplexed (by individual) FASTQ files. We suggest using [cutadapt](http://cutadapt.readthedocs.io/en/stable/index.html) and [FLASh](https://ccb.jhu.edu/software/FLASH/) for these steps, and provide details of the filtering done for Dupuis et al. (2018) below:

First, reads need to be demultiplexed by individual. If sequencing is done on an Illumina MiSeq or HiSeq connected to BaseSpace, this demultiplexing may be done on BaseSpace. Alternatively, cutadapt can be used to demultiplex based on individual-specific barcodes.

Next, cutadapt can be used to remove any additional Illumina adapters. We will assume that the raw FASTQ files are in `./RawData/`, and cutadapt can be run like:
```
for x in `cat individuals`; do echo "$x" |  cutadapt -a agatcggaagagcacacgtctgaa -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o AdapterTrimmed/"$x"_R1_adaptertrimmed.fastq -p AdapterTrimmed/"$x"_R2_adaptertrimmed.fastq  RawData/"$x"_R1_combined.fastq.gz RawData/"$x"_R2_combined.fastq.gz > CutAdapt_logs/log."$x".log ; done
```

Then, FLASh can be used to join the paired reads for all data:
```
for x in `cat individuals` ; do flash AdapterTrimmed/"$x"_R1_adaptertrimmed.fastq AdapterTrimmed/"$x"_R2_adaptertrimmed.fastq -o Flash/"$x"_flash.fastq | tee Flash/"$x".log ; done
```

Finally, cutadapt can be used again, but this time to demultiplex each individual, FLASh-joined FASTQ file by amplicon. Dupuis et al. (2018) pooled 384 individuals and 878 amplicons into single sequencing lanes, so we used a job array on a cluster to speed this process up. The job file (using SGE) looked something like this, and would need to be modified for other job schedulers:
```
#!/bin/sh
#$-S /bin/sh
#$-o Demultiplexed_CutAdapt_Logs/$JOB_ID.out 
#$-e Demultiplexed_CutAdapt_Logs/$JOB_ID.err
#$-q all.q
#$-pe orte 1
#$-cwd
#$-V
#$-t 1-384
INFILE=`awk "NR==$SGE_TASK_ID" individuals`
echo "$INFILE"
module load cutadapt
cutadapt -O 10 -a orth10028_611-1041=TGCCCATCGCCTCCGAy...GAGGTGTACTTGGTGGGCG \
-a orth10034_825-2024=GGCACCACATTCTCACAm...TGTATCAACTGCAGGCGAG \
-a orth10118_500-1241=TGTGGCTACTCGTGTCGw...CCACATGAATGTAAAGTATGTGGACG \
... .... ...
-a orth9938_2137-2366=AGCsAAGGTGCAACAAGTCT...GGCGATCGTCGGGATCA \
-a orth9941_1692-2530=AAGAGAGCAACCCACCTr...AGCCATGGAACTCGCCAA \
-o Demultiplexed_CutAdapt/"$INFILE"/"$INFILE".{name}.fastq Flash/"$INFILE"_flash.fastq.extendedFrags.fastq | tee Demultiplexed_CutAdapt_Logs/"$INFILE".log
```
Here, the `-a` options specify the amplicon primer pairs, so this job file would contain 878 `-a` options. We also split the output into multiple directories, one per individual, which is required for the part03 script that calls consensus sequences. Splitting the files into multiple directories is also a good idea, as 384 individuals x 878 amplicons = 337,152 files at the end of this step (which could stall bash commands or general command line manipulation). This process could also be run through a for loop, with modified variables in the input/output paths of the cutadapt command.

Following these steps, the sequencing data is now demultiplexed by individual and amplicon, and the data structure should be one main directory (`Demultiplexed_CutAdapt`) containing a single directory for each individual; each individual directory contains one FASTQ file per amplicon. Part03 expects this data structure, and expects individual FASTQ file names to be structured as, e.g. `Btau_Nepal_2983_1.orth9941_1692-2530.fastq` where `Btau_Nepal_2983_1` is the individual identifier, and `orth9941_1692-2530` is the amplicon identifier. With this data structure and naming scheme, part03 can be ran through a single script:
```
./step05_make_consensus.py 
```

This script finds the most prevalent read length for each individual per amplicon and calls a degenerate consensus sequence based on all reads of that length, using the rules of [Cavener 1987 Nucleic Acids Research 15:1353–1361](https://academic.oup.com/nar/article-lookup/doi/10.1093/nar/15.4.1353) (via [Bio.motifs](http://biopython.org/DIST/docs/tutorial/Tutorial.html) in BioPython). By default, a minimum of five reads per consensus is required, any consensus reads <65 bp are removed, and if an individuals' consensus sequence length deviates >20 bp from the mean consensus sequence length for that amplicon, it is removed. These default values can be edited in `config.toml`. 

The output of part03 includes individual FASTA files for each consensus sequence, and a single multi-FASTA per amplicon containing all individuals' consensus sequences. This latter format is easily used directly to generate gene-trees, or concatenated using something like [catfasta2phyml.pl](https://github.com/nylander/catfasta2phyml) for concatenated phylogenetic analyses.

Note, the length deviation filter of part03 can throw away potentially good consensus sequences in the case that an amplicon primer set was amplifying multiple products of much different length. For example, if half of the consensus sequences are 300 bp and the other half are 100 bp, the average would be 200 bp, and >20 bp different than every consensus sequence (thus throwing away all sequences). It is a good idea to check over the summary .csv files created by part03 for each individual. If an amplicon's consensus sequences are found in these files, but are not being written into the overall summary.csv or multi-FASTA files, this may be the culprit. The 20 bp threshold in Dupuis et al. (2018) was based on a natural break in the data to remove very short sequences that were obviously not full amplicon sequences. In this scenario, a simple `cat` of all of the singel FASTA files for each individual can create an amplicon-specific mutli-FASTA.
