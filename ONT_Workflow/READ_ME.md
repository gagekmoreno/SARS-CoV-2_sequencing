## Introduction
Here is a pipeline that will demultiplex ONT reads, clean them so there is no reagent and host reads, and generate sendsketch profiles for all the reads.  

## Methods
### Running Docker 
First, changed to the directory of fastq_pass reads: 
```
cd PATH_TO_DATA
```

Once I was in this folder, I started the viral-ont Docker container. All workflow was expected to be done within this container. 

```
# load Docker container
docker run \
--user $(id -u):$(id -g) \
-it \
-v $(pwd):/scratch \
-w /scratch \
dockerreg.chtc.wisc.edu/dhoconno/viral-ont:22664 \
/bin/bash
```

### Download bbtools minhash sketch for nt database
### Run sendsketch locally
Download the ntSketch profile to generate sendsketch profiles with.

```
# download
wget https://portal.nersc.gov/dna/microbial/assembly/bushnell/ntSketch.tar

# untar
tar -xvf ntSketch.tar
```

Make sure that the file path for nt sketch is 'nt/global/projectb/sandbox/gaag/bbtools/nt/current/*.sketch'

#### Start pipeline
Type this command in your terminal window to start workflow.

```
bash 22682.workflow.sh
```

This will start going through and running all of the attached bash and snakemakefiles.

What is actually in the workflow (note the few things you'll have to change in the file before you run it):

- BARCODE = barcode##
- HOST_RNA.fna.gz = for humans: GCF_000001405.39_GRCh38.p13_rna.fna.gz
- HOST_GENOMIC.fna.gz = for humans: GCF_000001405.39_GRCh38.p13_genomic.fna.gz

```
#! /bin/bash

# run through complete workflow using a combination of shell scripts and snakemake files

## Parameters ##
FASTQ_PASS_FOLDER='fastq_pass'
NUMBER_FASTQ_PARTITIONS='36'
MINIMUM_READ_LENGTH='500'

## partition sequences ##
bash ./22682-01.parition.sh  $FASTQ_PASS_FOLDER $NUMBER_FASTQ_PARTITIONS

## demultiplex sequences ##
snakemake \
--snakefile 22682-02.demultiplex.snakefile \
--config \
min_length=$MINIMUM_READ_LENGTH \
partitioned_fastq_folder=partitioned_fastq \
--cores $NUMBER_FASTQ_PARTITIONS

## merge demultiplexed sequences into one FASTQ per barcode ##
bash 22682-03.merge-demultiplex.sh \

## remove host and reagent reads ##
# this step is run separately on each barcode because the host databases may be different when multiple samples from multiple species are run in a single ONT run
preprocess () {
    snakemake \
    --snakefile 22682-04.remove-host-reagent.snakefile \
    --config \
    ont_fastq_gz=$1 \
    reagent_db=22592-reagent-db.fasta.gz \
    host_rna_db=$2 \
    host_dna_db=$3 \
    --cores 36
}

preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;
preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;
preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;
preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;
preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;
preprocess merged_demultiplexed/BARCODE.fastq.gz HOST_RNA.fna.gz HOST_GENOMIC.fna.gz;

## make minnashes from cleaned reads vs. nt database ##
snakemake --snakefile 22682-05.minhash.dataset.snakefile --config cleaned=cleaned --cores 36
snakemake --snakefile 22682-05.minhash.sequences.snakefile --config cleaned=cleaned --cores 36

## cleanup
# rm -rf demultiplexed merged_demultiplexed merged_fastq partitioned_fastq tmp
```
