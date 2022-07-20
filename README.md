# RNASeq Workshop
This repository contains a walkthrough of how to analyze RNA-Seq data, using a sample dataset from Bernal et al. 2020 (*Science Advances*). This repository will be used in the RNA-Seq data analysis workshop held at UT Chattanooga from August 10-12, 2022.

## Contents
1. [Getting set up](https://github.com/kmeaton/RNASeq_Workshop#getting-set-up)
2. [Cleaning and mapping reads](https://github.com/kmeaton/RNASeq_Workshop#day-1-cleaning-and-mapping-reads)
3. [Counting reads and analyzing expression patterns](https://github.com/kmeaton/RNASeq_Workshop#day-2-generating-read-counts-and-testing-for-differential-gene-expression)

## Getting set up

If you don't already have an account on [Github](https://github.com), make one now.

1. Log in to your [Github](https://github.com) account.

2. Fork this repository by clicking the "Fork" button on the upper right of this page. After a few seconds, you should be looking at your own copy of this repository in your own Github account. 

3. Click the green "Code" button at the upper right of this page. Click the tab that says HTTPS, then copy the link that's shown below it. 

4. Log in to your UTC computing cluster account by typing the following code into the terminal, substituting your UTC username in where it says [user]. You'll be prompted to enter a password, which you'll type right into the terminal.
```shell
ssh [user]@epyc.simcenter.utc.edu
```

5. Once you're logged in, in your home directory, type the following to clone into the repository. Make sure you're cloning into __your__ fork of the repository, not my original one.
```shell
git clone the-url-you-copied-in-step-3
```

6. Next, move into this directory:
```shell
cd RNASeq_Workshop
```

7. At this point, you should be in your own local copy of the repository, which contains all the scripts you'll need to edit and run to analyze our practice dataset. 

## Day 1: Cleaning and mapping reads

Log in to your account on the UTC Computing Cluster, substituting your UTC username in where it says [user]. 

```shell
ssh [user]@epyc.simcenter.utc.edu
```

You'll be prompted to enter your password, and then you'll be logged in to the cluster. Move into the directory you cloned yesterday, which contains all the scripts you'll need to work with today.

```shell
cd RNASeq_Workshop
```

### Assessing raw read quality

For this tutorial, we are going to use a sample RNA-Seq dataset from [this paper](https://www.science.org/doi/full/10.1126/sciadv.aay3423). The authors examined how gene expression changed in several species of fish as they were exposed to unseasonably warm temperatures over several months. We will use RNA-Seq data from one of the species (the spiny chromis damselfish, *Acanthochromis polyacanthus*) and compare gene expression between two months (December, when temperatures were relatively normal, and February, when temperatures were far above average). 

In the shared class directory, there are 8 files with the extension ".fq". These fastq files contain "raw" RNA-Seq reads for 4 samples - 2 from December and 2 from February. Each sample will have data in two different files: one will have the extension \_mate1.fq, and the other will be \_mate2.fq. This is because these samples were sequenced using paired-end Illumina sequencing, which generates two sequences per input molecule of RNA (one sequence in the "forward" direction, and one in the "reverse" direction). These paired sequences are stored in two different files.

You'll need to copy these files from the class directory to your home directory before you can start working on them. Type the following code in your terminal:

```shell
cp /scr/south_east_comp/*.fq ~/
cd ~
ls
```

You should now see that in your home directory, you have your own copies of the raw data files. 

Before we can do any analysis on our raw data, we have to clean them to remove any low-quality reads or contamination. We'll start by examining the quality of the sequences, using the program FastQC. 

First, create a new directory for our FastQC results to go into:
```shell
mkdir ~/raw_reports
```

Then, move into the RNASeq_Workshop folder that you cloned from Github. This folder contains all the scripts you'll need to run today. Examine the code in the script ```fastqc.sh```, and then run the script by typing the following:
```shell
# First examine the code in the script
cat fastqc.sh
# Once you understand what the code is doing, run the script
sbatch fastqc.sh
```

When the script is done running, take a look at the output. You'll have to download the reports to your local machine. Open a terminal on your __local machine__ and type the following:
```shell
scp -r [user]@epyc.simcenter.utc.edu:~/raw_reports/*.html ~/Desktop/
```

You'll be prompted to enter your UTC password here, and then the reports will be copied from your fastqc run to the Desktop on your local computer. Click on the files that you just copied, open them up, and see what the sequencing data looks like. Is it high quality? How can you tell? 

### Removing adapters and low-quality sequences

Now that we know what our raw reads look like, we should trim the sequencing adapters and remove any low-quality reads. We can do this using the program Trimmomatic. Move into the RNASeq_Workshop folder and examine the Trimmomatic script we are going to run. For more information on the quality trimming parameters, examine [the manual for trimmomatic](http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf). Once you understand what the code is doing, run the script by typing the following:

```shell
# Examine the code in the script
cat trimmomatic.sh
# Run the script
sbatch trimmomatic.sh
```

This program takes paired-end sequencing data for each of our samples, trims any sequencing adapters that are present in the reads, and then removes any low-quality sequences from the dataset. You'll end up with four files for each sample:

* [sample]\_1\_paired.fq
* [sample]\_1\_unpaired.fq
* [sample]\_2\_paired.fq
* [sample]\_2\_unpaired.fq

We will continue our analyses only using the "paired" files. These contain sequences where both members of a "pair" of reads have passed the quality filtering step. Check out the sizes of the fastq files containing the paired and unpaired reads by running the following commands:

```shell
ls -lh *_paired.fq
ls -lh *_unpaired.fq
```

The fourth column in the output from this command will show the approximate size of the file. Since the majority of our reads were of high quality, the "paired" files (where both reads in a mate pair passed the quality filtering step) should be several times larger than the "unpaired" files. 

### Checking the quality of our trimmed and filtered reads

Let's make sure that the filtering steps worked well, and that the quality of our sequences increased after trimming and filtering. To do this, we'll run FastQC again, but this time on the \*\_paired.fq files, which contain the sequences we will proceed with for our analyses.

We'll make a new directory for these reports.

```shell
mkdir filtered_reports
```

Then, modify the ```fastqc.sh``` script in your folder so that it will run on your filtered files and send the output to the filtered_reports folder we just created, and save it as a new file called ```fastqc_filtered.sh```. You can do this in your favorite text editor, like nano or vim. 

Run your new script, fastqc_filtered.sh, and then download the reports to your local machine like we did before. Check out the sequence quality now - how has it improved?

### Mapping our trimmed reads to a reference

Now that we have high-quality, filtered reads for each of our samples, we need to map them to a reference genome. This allows us to quantify the number of sequences in our dataset that originated from each gene in our organism's genome. For our analyses, we will use the published genome of the spiny chromis damselfish, *A. polyacanthus* (publicly available on NCBI at accession number: GCF_002109545.1). 

## Day 2: Generating read counts and testing for differential gene expression


## Acknowledgements
This workshop was made possible by funding provided to Fernando Alda from the University of Tennessee at Chattanooga. 

The sample datasets used in this analysis are publicly available on NCBI. The version of the *A. polyacanthus* genome used is available at accession number GCF_002109545.1. The raw RNA-Seq reads are available under NCBI BioProject Number PRJNA489934 and SRA accession number SRP160415. We thank the following authors of the study that generated this dataset: Moises Bernal, Celia Schunter, Robert Lehmann, Damien Lightfoot, Bridie Allan, Heather Veilleux, Jodie Rummer, Philip Munday, and Timothy Ravasi, as they have graciously allowed us to use their data (Bernal et al. 2020, *Science Advances* 6(12): eaay3423). 

The formatting of this tutorial, as well as the "Getting set up" portion of this page, borrow heavily from the Scripting for Biologists python tutorials written by [Jamie Oaks](http://phyletica.org) available [here](https://github.com/joaks1). 
