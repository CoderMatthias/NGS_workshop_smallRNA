# Small RNA analysis
Practical for the smallRNA section of the NGS workshop.

While useful for those interested in small RNA analysis, this section will provide an example of what goes into the running of a typical bioinformatics pipeline.

____

## Before we get started...

+ \* -- wild card
+ # -- comment out
+ . -- current working directory
+ && -- chains commands together; do this command, then this one
+ Tab-completion
+ Relative paths versus absolute paths
+ On command line notation
+ On /proj/seq/data
+ On annotation files


## Setup

#### Log in

Login to Killdevil with your onyen.

#### Change to your scratch directory

Switch from the home directory to your scratch directory, by typing `$ cd /netscr/ONYEN`, where ONYEN is your UNC onyen.

#### Copy scripts and data to your scratch directory

The necessary miRquant scripts, resource files, and data have been compressed and can be copied from /netscr/mattkank/miRquant.tar.gz.  To copy the files over and un-compress them, we will submit the job to LSF by typing:

```
$ cp /netscr/mattkank/miRquant.tar.gz . && tar -zxvf miRquant.tar.gz  && cd miRquant
```

#### Load pipeline requirements

The clusters (Killdevil and Kure) have many programs and applications already installed.  To call on them, you simply need to call on them.  Let's take a look at which applications are available to us, type:

```
$ module avail
```
miRquant requires several applications.

The required applications are:  
cutadapt  
blast  
bedtools  
SHRiMP  
R

While modules can be loaded individually by typing:
```
$ module load blast
$ module load bedtools
ect...
```
this can be tedious when you have many modules required for a pipeline.  One alternative is to create a file containing the environmental variables (modules, paths, ect.) we want to use for the pipeline.  The uncENV.sh has all the necessary variables we need for the pipeline!  Lets take a look at this file.
```
$ cat uncENV.sh

# Adds locations to system path, so the system knows where to look for programs
export PATH=/netscr/mattkank/miRquant/02_software/bin:./:$PATH
export PATH=$PATH:/netscr/mattkank/miRquant/02_software/SHRiMP_2_2_2/bin
export PYTHONPATH=/netscr/mattkank/miRquant/02_software/python/

# Sets location of required components to variables
export SHRIMP_FOLDER=/netscr/mattkank/miRquant/02_software/SHRiMP_2_2_2
export CHROMO_SIZE=/netscr/mattkank/miRquant/02_software/resources/mm9.chromSizes
export JB_GENOME_DIR=/proj/seq/data/mm9/bowtie_path/base/
export MMU_GENOME=/proj/seq/data/MM9_UCSC/Sequence/WholeGenomeFasta/genome.fa

# Loads the modules necessary for the pipeline
module load r/2.15.1
module load bowtie/1.1.0
module load samtools/1.3
module load bedtools/2.25.0
module load bamtools/1.0.2
module load perl/5.12.0
```
uncENV.sh contains the location of programs used by the pipeline, the location of various files called on by the pipeline, and the commands to load the necessary modules.  We can execute the code within this file by typing `source uncENV.sh`.  If we type `module list` we should see the following output, confirming the necessary modules were loaded.

```
Currently Loaded Modulefiles:
  1) null              2) r/2.15.1          3) bowtie/1.1.0      4) samtools/1.3      5) bedtools/2.25.0   6) bamtools/1.0.2    7) perl/5.12.0
```

OK, we should now be setup to run the pipeline on non-descript SAMPLE_A.fastq.

## Running miRquant

Let's make sure first that we are in the correct location on the file system.

```
$ pwd
/netscr/ONYEN/miRquant
```

Take a look at what the directory structure is.

```
$ ls
00_documentation  02_software  05_scripts        chainSubmission.sh  process_all_summary2tab.pl  runC.sh
01_logs           03_samples   06_final_scripts  post_runC.sh        resources                   uncENV.sh

where:
01_logs is where the pipeline logs will be stored as they are produced (currently empty)
02_software is where some of the programs necessary for the pipeline are stored
03_samples is where the sequencing data and pipeline outputs will be stored
04_resources is where miRNA resource files required for the pipeline are stored
05_scripts is where various scripts called on by the pipeline are stored
06_final_output_scripts is where scripts used for the final assembly of the data is stored
```

Next we'll take a look within our 03_samples folder.

```
$ cd 03_samples
$ ls
SampleA.adaptor  SampleA.fastq
$ head SampleA.adaptor
TGGAATTCTCGGGTGCCAAGGAACTCCAGTCACATCACGATCTCGTATGCCGTCTTCTGCTTG
$ head SampleA.fastq
@D1317JN1:309:C88CWACXX:1:1101:2446:2123 1:N:0:ATCACG
TGGAGTGTGACAATGGTGTTTTGGAATTCTCGGGTGCCAAGGAACTCCAGT
+
BB@FFDEDHHHFHIHIGHIJJJJJGHIJJJJJJJJIJJHHIJJJJJJJJHF
@D1317JN1:309:C88CWACXX:1:1101:2294:2127 1:N:0:ATCACG
GTCTACGGCCATACCACCCTGTGGAATTATCGGGTGCCAAGGAACTCCAGT
+
?=?DD=DD1C?ADE3CBEFI>AE3@??D*?DD?DDDDDDD9CDEDED7A@=
@D1317JN1:309:C88CWACXX:1:1101:2353:2170 1:N:0:ATCACG
AATACCGGGTGCTGTAGGCTTTTGGAATTCTCGGGTGCCAAGGAACTCCAG
```
For the fastq output, this is the information for our first two (and half of the third) reads.  For each read, the first line starts with a '@' and is followed by a read identifier containing various bits of information (instrument name, flowcell id, ect).  The second line contains the raw sequence letters.  The third line is a '+'.  The fourth line encodes the quality value for the sequence in the second line.  Each read contains a portion of the 3' adapter sequnce, which will have to be trimmed.

Change back to the main directory by typing `$ cd ..`.

#### Section 1: chainSubmission.sh

The first part of miRquant trims the adapter, aligns RNAs perfectly matching the genome with Bowtie, creates genomic 'windows' for the SHRiMP alignment using bedtools, and aligns small RNAs containing mismatches to these windows using SHRiMP.  This is something you'll come across often as you use various pipelines, the stringing together of well-written programs to achieve the goal (if a toilet is available, why dig a hole in the woods).  All these programs brought together and connected in a 'wrapper' script, in this case chainSubmission.sh.  To run chainSubmission.sh, enter:

```
$ bsub -o 01_logs/chainSubmission.log bash chainSubmission.sh mmu /netscr/ONYEN/miRquant/03_Samples
```

Pause for ~10 seconds, then confirm your job is running.

```
$ bjobs
JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME
186010  mattkan RUN   day        killdevil-l donor_pool2 *leA.fastq May 13 15:19
```
Continue to check until the job finishes, the type `$ more 01_logs/chainSubmission.log`.  This is the log file for the chainSubmission.sh run of the script.  In this file you'll see the output for the various programs which were 'wrapped' together in chainSubmission.sh.  Logs should be checked after every run, as this is where you are likely to catch errors which may have occured while running a pipeline.

#### Section 2 & 3: runC.sh & post_runC.sh

The second part of miRquant processes the results from the Bowtie alignment and SHRiMP alignment in the first stage and separates these based on length, internal nucleotide edits, and 3' additions.  This part also determines if a read corresponds to a known miRNA.  The third part combines the results from the second part into a single file.  To run this second part, be very mindful of tab-completion and enter:

```
$ bash runC.sh /netscr/ONYEN/miRquant/03_samples/SampleA./IntermediateFiles/g1Results/CHR*.results
```


