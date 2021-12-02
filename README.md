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
#### Create Anaconda environment:
```
./create_envs.sh
```


