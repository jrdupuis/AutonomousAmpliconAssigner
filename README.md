# AutonomousAmpliconAssigner

AutonomousAmpliconAssigner (AAA) processes and calls consensus sequences from multiplexed amplicon sequence data. AAA takes as input demultiplexed, adapter-trimmed, and FLASh-joined FASTQ files, and calls consensus sequences based on read length distributions. It outputs aligned multi-FASTA formatted files (1 per gene), that can be used directly for gene-tree analysis or concatenated for an "all data" approach, and summary information. This code base is an updated version of HiMAP's part03, and more details on potential processing steps before using AAA (going from raw illumina reads to AAA's input) are provided [here](https://github.com/popphylotools/HiMAP#part03).

## Installation

#### Install Anaconda distribution of python

[Anaconda download page](https://www.continuum.io/downloads)

#### Download this git repo
Using git:
```
git https://github.com/jrdupuis/AutonomousAmpliconAssigner.git
cd AutonomousAmpliconAssigner
```
#### Create and activate Anaconda environment: 
(note, env is installed in the current dir in ./AAA_env)
```
bash ./create_envs.sh
conda activate ./AAA_env
```

#### Download Data
A toy dataset from [Dupuis et al. (2018)](https://onlinelibrary.wiley.com/doi/abs/10.1111/1755-0998.12783) is included in this repo ([here](https://github.com/jrdupuis/AutonomousAmpliconAssigner/tree/main/input_data)). It contains a single specimen's data for XXX loci.

## Usage
#### config.toml
The file `config.toml` contains a few editable parameters and data paths for running AAA. 

#### Running AAA
AAA runs through a single script, `AAA.py`. The input is adapter-trimmed and demultiplexed (by individual) FASTQ files in individual-specific directories with the naming scheme `[individual]/[individual].[ortholog].fastq`. E.g., `Btau_Nepal_2983_1.orth9941_1692-2530.fastq` where `Btau_Nepal_2983_1` is the individual identifier and `orth9941_1692-2530` is the locus identifier. More details on preparing these input directories/files using [cutadapt](http://cutadapt.readthedocs.io/en/stable/index.html) and [FLASh](https://ccb.jhu.edu/software/FLASH/) are provided [here](https://github.com/popphylotools/HiMAP#part03).

Executing AAA is as simple as running
```
./AAA.py
```

AAA.py finds the most prevalent read length for each individual per amplicon and calls a degenerate consensus sequence based on all reads of that length, using the rules of [Cavener 1987 Nucleic Acids Research 15:1353–1361](https://academic.oup.com/nar/article-lookup/doi/10.1093/nar/15.4.1353) (via [Bio.motifs](http://biopython.org/DIST/docs/tutorial/Tutorial.html) in BioPython). By default, any consensus reads <65 bp are removed, and if an individuals' consensus sequence length deviates >20 bp from the mean consensus sequence length for that amplicon, it is removed. These default values can be edited in `config.toml`. 

The output of `AAA.py` includes individual FASTA files for each consensus sequence, and a single multi-FASTA per amplicon containing all individuals' consensus sequences. 

Note, the length deviation filter of `AAA.py` can throw away potentially good consensus sequences in the case that an amplicon primer set was amplifying multiple products of much different length. For example, if half of the consensus sequences are 300 bp and the other half are 100 bp, the average would be 200 bp, and >20 bp different than every consensus sequence (thus throwing away all sequences). It is a good idea to check over the `summary.csv` file created by `AAA.py` for each individual. If an amplicon's consensus sequences are found in these files, but are not being written into the overall `summary.csv` or multi-FASTA files, this may be the culprit. The 20 bp threshold established in Dupuis et al. (2018) was based on a natural break in the data to remove very short sequences that were obviously not full amplicon sequences. In this scenario, a simple `cat` of all of the singel FASTA files for each individual can create an amplicon-specific mutli-FASTA.
