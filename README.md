# Natter-Pipeline 

Natter is an open-source bioinformatics pipeline for the preprocessing of raw sequencing data.
The need for a scalable, reproducible workflow for the processing of environmental amplicon data led to the development of Natter. It is divided into quality assessment, read assembly, dereplication, chimera detection, split-sample merging, OTU-generation and taxonomic assessment. The pipeline is written in [Snakemake](https://snakemake.readthedocs.io), a workflow management engine for the development of data analysis workflows. Snakemake ensures reproducibility of a workflow by automatically deploying dependencies of workflow steps (rules) and scales seamlessly to different computing environments like servers, computer clusters or cloud services. While it was only tested with 16S and 18S amplicon data, it should also work for different kinds of sequencing data. The pipeline contains seperate rules for each step of the pipeline and each rule that has additional dependencies has a seperate [conda](https://conda.io/) environment that will be automatically created when starting the pipeline for the first time. The encapsulation of rules and their dependencies allows for hassle-free sharing of rules between workflows.

![DAG of an example workflow](documentation/images/example_dag.png)
*DAG of an example workflow: each node represents a rule instance to be executed. The direction of each edge represents the order in which the rules are executed. Disjoint paths in the DAG can be executed in parallel. Below is a schematic representation of the main steps of the pipeline, the color coding represents which rules belong to which main step.*



*So far, Natter was only tested on Ubuntu 17.10, 18.04 and Arch Linux, feedback if it runs on other operating systems and distributions would be greatly appreciated.*

---

## Getting Started

To install Natter, you'll need the open-source package management system conda and, if you want to try Natter using the accompanying `pipeline.sh` script you'll need [GNU screen](https://www.gnu.org/software/screen/) (in most Linux distributions GNU screen is already preinstalled).
After cloning this repository to a folder of your choice, it is recommended to create a general snakemake conda environment with the accompanying `snakemake.yaml`. In the main folder of the cloned repository, execute the following command:

```shell
$ conda env create -f snakemake.yaml
```
This will create a conda environment containing all dependencies for Snakemake itself. To try out Natter using the example data, type in the following command:

```shell
$ ./pipeline.sh
Please enter the name of the project
$ example_data
```
The pipeline will then start a screen session using the project name (here, *example_data*) as session name and will beginn downloading dependencies for the rules. To detach from the screen session, press **Ctrl+a, d** (*first press Ctrl+a and then d*). To reattach to a running screen session, type in:

```shell
screen -r
```

When the workflow has finished, you can press **Ctrl+a, k** (*first press Ctrl+a and then k*). This will kill any processes still running in this screen session.

**NOW LINK TO THE EXPLAINATION OF THE RESULTS AND TO THE IN-DEPTH EXPLAINATION HOW TO USE THE PIPELINE.**

---

## Tutorial

**BLABLABLA FILENAMES, CONFIG, PRIMERTABLE ETC**

---

## Output

**HIERARCHY OF THE OUTPUT FOLDERS, EXPLAINATION OF ALL OUTPUT FILES**

# Steps of the Pipeline **NOBODY READS ALL THIS, PUT IT IN THE WIKI**
## Initial demultiplexing
The sorting of reads accoring to their barcode is known as demultiplexing. In Natter, the demultiplexing step is implemented in a separate script, independent from the rest
of the pipeline, as it is often already done by the sequencing company and therefore in most cases not necessary.

## Prerequisites: primer table and configuration file
Besides the FASTQ data from the sequencing process Natter needs a primer table
containing the sample names and, if they exists in the data, the length of the poly-N tails, the sequence of the primers and the barcodes used for each sample and direction. Besides the sample names all other information can be omitted if the data was already preprocessed or did not contain the corresponding subsequence. An example primer table is shown in section **PRIMERTABLE SECTION**. Natter also needs a configuration file in YAML format, specifying parameter values for tools used in the pipeline. An example configuration file with a description and default value
for each parameter is shown in **CONFIG SECTION**.

## Quality control
For quality control the pipeline uses the programs FastQC (Andrews 2010), MultiQC (Ewels
et al. 2016) and PRINSEQ (Schmieder and Edwards 2011). **ALL NEED REFERENCES**

### FastQC 
FastQC generates a quality report for each FASTQ file, containing information such
as the per base and average sequence quality (using the Phred quality score), overrepresented
sequences, GC content, adapter and the k-mer content of the FASTQ file.

### MultiQC 
MultiQC aggregates the FastQC reports for a given set of FASTQ files into a
single report, allowing reviews of all FASTQ files at once.
PRINSEQ PRINSEQ is used to filter out sequences with an average quality score below
the threshold that can be defined in the configuration file of the pipeline.

## Read assembly
### Define primer
The define_primer rule specifies the subsequences to be removed by the
assembly rule, specified by entries of the configuration file and a primer table that contains information about the primer and barcode sequences used and the length of the poly-N subsequences. Besides removing the subsequences based on their nucleotide sequence, it is also possible to remove them based solely on their length using an offset. Using an offset can be useful if the sequence has many uncalled bases in the primer region, which could otherwise hinder matches between the target sequence defined in the primer table and the sequence read.

### Assembly 
To assemble paired-end reads and remove the subsequences described in the pre-
vious section PANDAseq (Masella et al. 2012) **REFERENCE** is used, which uses a probabilistic error correction to assemble overlapping forward- and reverse-reads. After assembly and sequence trimming, it will remove sequences that do not meet a minimal or maximal length threshold, have an assembly quality score below a threshold that can be defined in the configuration file and sequences whose forward- and reverse-read do not have an sufficiently long overlap. The thresholds for each of these procedures can be adjusted in the configuration file. If the reads are single-end, the subsequences (poly-N, barcode and the primer) that are defined in the define_primer rule are removed, followed by the removal of sequences that do not meet a minimal or maximal length threshold as defined in the configuration file.
## Similarity clustering
### Conversion of FASTQ files to FASTA files 
The rule copy_to_fasta converts the FASTQ files to FASTA files to reduce the disc space occupied by the files and to allow the usage of CD-HIT, which requires FASTA formated sequencing files. Clustering of similar sequences The CD-HIT-EST algorithm (Fu et al. 2012) **REFERENCE** clusters sequences together if they are either identical or if a sequence is a subsequence of another. This clustering approach is known as dereplication. Beginning with the longest sequence of the dataset as the first representative sequence, it iterates through the dataset in order of
decreasing sequence length, comparing at each iteration the current query sequence to all representative sequences. If the sequence identity threshold defined in the configuration file is met for a representative sequence, the counter of the representative sequence is increased by one. If the threshold could not be met for any of the existing representative sequences, the query sequence is added to the pool of representative sequences. Cluster sorting The cluster_sorting rule uses the output of the cdhit rule to count the amount of sequences represented by each cluster, followed by sorting the representative sequences in descending order according to the cluster size and adds a specific header to each
sequence as required by the UCHIME chimera detection algorithm.
## Chimera detection
### VSEARCH
VSEARCH is a open-source alternative to the USEARCH toolkit, which aims to functionally replicate the algorithms used by USEARCH for which the source code is not openly available and which are often only rudimentarily described (Rognes et al. 2016) **REFERENCE**. Natter uses as an alternative to UCHIME the VSEARCH uchime3_denovo algorithm to detect chimeric sequences (further referred to as VSEARCH3). The VSEARCH3 algorithm is a replication of the UCHIME2 algorithm with optimized standard parameters. The UCHIME2 algorithm is described by R. Edgar 2016 **REFERENCE** as follows:
"Given a query sequence *Q*, UCHIME2 uses the UCHIME algorithm to construct a model
(*M*), then makes a multiple alignment of *Q* with the model and top hit (*T*, the most similar reference sequence). The following metrics are calculated from the alignment: number of differences d<sub>QT</sub> between Q and T and d<sub>QM</sub> between *Q* and *M*, the alignment score (*H*) using eq. 2 in R. C. Edgar et al. 2011. The fractional divergence with respect to the top hit is calculated as div<sub>T</sub> = (d<sub>QT</sub> − d<sub>QM</sub>)/|Q|. If divT is large, the model is a much better match than the top hit and the query is more likely to be chimeric, and conversely if div<sub>T</sub> is small, the model is more likely to be a fake." The difference between the UCHIME2 and UCHIME3 algorithm is that to be selected as a potential parent, a sequence needs to have at least 16 times the abundance of the query sequence in the UCHIME3 algorithm, while it only needs double the abundance of the query sequence to be selected as a potential parent in the UCHIME2 algorithm.

## Table creation and filtering
### Merging of all FASTA files into a single table
For further processing the rule unfiltered_table merges all FASTA files into a single, nested dictionary, containing each sequence as the key with another dictionary as value, whose keys are all (split -) samples in which the sequence occurred in and as values the abundance of the sequence in the particular (split - ) sample. For further pipeline processing the dictionary is temporarily saved in JSON format. To ease statistical analysis of the data the dictionary is also exported as a comma separated table. Filtering In the filtering rule of the pipeline all sequences that do not occur in both splitsamples of at least one sample are filtered out. For single-sample data, the filtering rule uses an abundance cutoff value that can be specified in the configuration file to filter out all sequences which have abundances less
or equal the specified cutoff value. The filtered data and the filtered out data is subsequently exported as comma separated tables.

### Table conversion to FASTA files
As the swarm rule needs FASTA files as input, the resulting table of the filtering is converted to a FASTA file by the rule write_fasta.

## AmpliconDuo / Split-sample approach
The pipeline supports both single-sample and split-sample FASTQ amplicon data. The split-sample protocol (Lange et al. 2015) **REFERENCE** aims to reduce the amount of sequences that are the result of PCR or sequencing errors without the usage of stringent abundance cutoffs, which often lead to the loss of rare but naturally occurring sequences. To achieve this, extracted DNA from a single sample is divided into two split-samples, which are then separately amplified and sequenced. All sequences that do not occur in both split-samples are seen as erroneous sequences and filtered out. The method is therefore based on the idea that a sequence that
was created by PCR or sequencing errors does not occur in both samples. A schematic
representation of the split-sample method is shown below:

<p align="center"> 
<img src="documentation/images/splitsample.png" alt="split_sample" width="300"/>
</p>

*Schematic representation of the split-sample approach: Extracted DNA from a single en-
vironmental sample is splitted and separately amplified and sequenced. The filtering rule compares the resulting read sets between two split-samples, filtering out all sequences that do not occur in both split-samples. Image adapted from Lange et al. 2015.*





