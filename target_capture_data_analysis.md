# **Analyse target capture data to obtain regions of interest**
    
[Working environment, pre-requisites and best practices](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#1-working-environment-pre-requisites-and-best-practices)  
[Checking data quality](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#2-checking-data-quality)  
[Cleaning data](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#3-cleaning-data)  
[Assembly strategy and reference file](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#4-choosing-a-strategy-to-assemble-the-data-and-formatting-the-reference-file)  
[Hybpiper tips](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#5-tips-when-running-hybpiper)  
- [Name list](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#name-list)  
- [PATH](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#path)  
- [Hybpiper command](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#typical-command)  
- [Retrieving the splash zone](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#retrieving-the-splash-zone)  
- [Gathering homologous sequences from all samples](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#gathering-supposedly-homologous-regions-from-all-samples)  
- [Running long or memory-consuming jobs](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#running-long-or-heavy-jobs)  
- [Checking for paralogs](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#checking-for-paralogs)  
  
[General advice](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#general-advice)  
[Target capture efficiency](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#6-assess-target-capture-efficiency)  
[Improving target recovery](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/target_capture_data_analysis.md#7-improving-target-recovery)  
    
    
## **1. Working environment, pre-requisites, and best practices**

Processing sequencing data is much easier, faster and customizable using command line. Many bioinformatic tools are free and designed to work in a unix-like environment, so people usually analyse NGS data on OS X or Linux computers.  
  
There are already many free tools, pipelines and tutorials available so most often you just need to know enough basic commands to link these tools together and to parse results, so **don’t panic**, and don’t hesitate to google your problem!  

Tutorials to **learn basic linux commands** can be found on the Internet. Knowing basic commands will be necessary. You should at least be able to create a directory, move from a directory to another, copy or move files, look into the beginning or the end of a file, find a pattern inside a file, and get information about a particular command.
  
More important than knowing many commands is to **understand the principles behind the tool that you use**, so that you can correctly analyze the data and properly interpret the results. But this does not require high bioinformatic skills, it requires to read the manual or article associated with the tool. More on this topic is provided [here](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/cluster/cluster-tips/).  
  
When you cannot find the tool you need or when you have difficulties to understand what is happening, ask a (bio)informatician!  
  
Kew has computing facilities that have been set up to enable the analysis of large amounts of target capture sequencing data by multiple users. Before using them, please ensure to:  
  
- **make yourself known** to Michael Chester or Pepijn Kooij for the general usage machines and to James Crowe for the cluster.
- **get familiar with the [facilities](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/Machines/)** 
- **take note of the general best practices described [here](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/best-practice/Best_practice/), and of the best usage practices described [here](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/utils/logging-jobs/) for the general usage machines, and [here](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/cluster/__intro/) for the cluster.**

Below we provide information on our current strategies to analyse target capture data to obtain multiple sequence alignments for individual genes.  
We also list useful software and give some default commands to use them, as well as some tips to use these software on multiple samples at once.  
The commands provided here are not meant to be copied and pasted directly (and they probably won't work as such), but they are given as a reference and starting point for beginners. You should adapt the commands and options to your dataset after reading the software documentation, probably try various options, and think about which steps are necessary or not to answer your research questions.

## **2. Checking data quality**
  
Usually Illumina sequencing data come back to you already **demultiplexed**: the sequencing machine created sets of files for each sample based on a sample sheet indicating which indexes corresponded to which sample. If the data are not demultiplexed, there are scripts online to do it.  

If the sequencing was **paired-end**, the data come in two files usually labelled R1 and R2. Each sequencing **read** in file R1 has a mate at the same position in file R2. The two mates correspond to the two extremities of a DNA fragment. Knowing that read 1 and read 2 correspond to a same fragment is very helpful when trying to assemble reads together, because we know that read 1 and read 2 are from two genomic regions that are separated by a distance inferior to the library size. More on paired-end sequencing and the resulting data can be found [here](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/images/Adapters.pdf) and in documents cited therein.
  
The format of the data is [**fastq**](https://en.wikipedia.org/wiki/FASTQ_format), which means that you have the sequencing reads and the quality of each **base** (nucleotide) of the read inside a single file. The quality of each base is expressed as the probability of the base to have been wrongly identified ("called") by the Illumina software. This is the [**phred score**](https://en.wikipedia.org/wiki/Phred_quality_score). Instead of being directly written in the file, each phred score is represented by a single character, following a code. This reduces the space taken by the fastq file. There are different codes that are used by different sequencing technologies. Current (2018) Illumina software (>v.1.8) use **phred33**. This is important to know because some data processing tools ask the user to specify it. 
  
The first thing to do when you get your data is to check if the reads are of sufficient quality, and to decide what to do to improve the quality of the individual reads and of the whole dataset. This is also the step where you can spot contamination by completely unrelated organisms (for instance fungi or bacteria when sequencing plants).
  
A great software to do this is [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) (S. Andrews, Babraham Bioinformatics), but there are other programs available.

To understand the graphical output of FastQC please follow this [link](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/Help/3%20Analysis%20Modules/).

Looking at the FASTQC results should help you decide which quality score threshold you want to accept as sufficient for each read base or each read. If your data is not of very good quality you may prefer to set a lower quality threshold so that you can still have enough data to perform your analyses, but if you do this you will have to keep in mind that there may still be many errors in the reads. This may be acceptable depending on your question or depending on how many reads of a base of interest you have.

An alternative to look at the quality of all the samples could be to use the script “clean_reads.py” of the pipeline [SEQCAPR](https://peerj.com/preprints/26477/) (Andermann et al., 2018). This script will trim the reads (see below) and produce a graphical summary of their quality for all species together.

### CODE TIPS

The **wild card** is very useful to perform simple operations on multiple inputs.  
For instance, to move all files with the "html" extension to a directory called "dir", you can use a wild card like this:
```
mv *.html dir
```
If you want to unzip files, you can use the command 
```
gunzip file
```
But ask yourself if it is really necessary before, because you will use more space.  
The program fastqc works on zipped (.gz) files as well as on fastq files.

**Loops** are used to perform a same operation (or suite of operations) on multiple inputs, one input after the other.  
The basic structure of the loop is:  
**for each_input; do (the operation); done**  
For instance, you can run the fastqc command on all files finishing by ".fastq" by typing:
```
for f in *.fastq; do (fastqc $f); done
```
"for f in \*.fastq" sets the loop: the loop will iterate as many times as there are objects of name finishing by ".fastq" in the current directory.  
At each iteration, a variable named f is created, and the object of name finishing by ".fastq" on which the loop is currently iterating is assigned to f, overwriting the previous value of f.  
"do ()" indicates an action to perform at each iteration of the loop (for instance run fastqc with the variable f as input).   
$f indicates that f is a variable.  
  
To copy files from your laptop/desktop to the remote server and vice versa, you can use the command **scp**:
```
scp path/to/file/on/local/computer user@server_address:/path/to/directory/on/server

scp user@server_address:/path/to/file/on/server path/to/directory/on/local/computer
```
The first command copies a file from your computer to the remote server (you need to know the address of the server and to have a user account on it).  
The second command copies a file from the server to your computer.  
**Both commands are run in the terminal of the local computer.**  
  
You can use loops or wild cards to copy many files at once.  
For instance the following command copies all fastq files from the directory called miseq_run located on the local computer to the directory called raw_data located in the account user in the remote server:
```
scp miseq_run/*.fastq user@server_address:raw_data/
```

## **3. Cleaning data**

Once you know how you should clean your data, you can use a tool such as [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) to do it.  
  
Trimming tools perform various operations, depending on the tool and on the options you choose.  
- They trim, i.e they cut, the end of the reads if they match adapter sequences (indicated by the user).  
- They trim the reads at the end and sometimes at the beginning, based on the quality of the base(s) or arbitrarily.  
- They remove completely some reads, based on their quality and/or their length.  
  
If the data was paired-end, and one of the reads of the pair has been removed but not the other, the remaining read is either removed too, or written in a separate file containing **unpaired** reads. This allows to keep two files of trimmed paired reads where reads at the same line number in both files correspond to the two reads of a pair.  
In Trimmomatic, this results in the creation of four output files: two files containing respectively read 1 and read 2 of each read pair, and two files containing respectively read 1 of the pairs for which read 2 has been deleted, and read 2 of the pairs for which read 1 has been deleted.

If you use Trimmomatic, you should read the [manual](http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf).  
  
As a general rule, we found that the MAXINFO setting is complicated to understand but gives good results. It takes a bit of trial and error to find the right parameters but it seems to be possible to keep the quality very high and dropping very few reads altogether, compared to SLIDINGWINDOW which seems to drop more reads. Apparently MAXINFO should only be used instead of SLIDINGWINDOW, not both together.  
Setting the "targetLength" parameter too low does result in lots of reads being dropped but the "strictness" parameter can be turned up to the maximum of 1 and still just trims without completely dropping reads, on the datasets that we tested.  
We did not find necessary to use TRAILING or LEADING but it might be necessary to use CROP on the long reads.
  
If you performed paired-end sequencing and used the NEBNext® Multiplex Oligos for Illumina®, the adapter file **TruSeq3-PE-2.fa** in "Trimmomatic-0.36/adapters/" should be the one you need (as of August 2018). We use the PALINDROME setting rather than SIMPLECLIP - this is automatically selected by using the PE-2 adapter file. The parameter set to "true" tells it to keep both reads even if redundant. 
  
After you cleaned the data, you should ideally check if your clean data are as you expect, and you should check the output of the trimming program, to know how many reads you lost, and how many you trimmed.  
In general, trimming a new dataset requires multiple trials. But after a while, it becomes easier to know what will work on which kind of dataset.  
  
### CODE TIPS

You can customize loops to create outputs with informative names:
```
for i in *.fastq; do (command $i > ${i/.fastq}-paired.fastq); done
```
In this example, we write the output in a new file using ">", and the name of this output file will be the same name as the name of the input file, after removal of ".fastq" and replacement of it by "-paired.fastq", to indicate that the output will contain reads belonging to intact pairs.

In the same way you can construct a loop that will be able to pick additional inputs based on the first input name that you specify:
```
for i in *R1.fastq; do (command $i ${i/R1.fastq}R2.fastq > ${i/.fastq}_paired.fastq ${i/R1.fastq}R2_paired.fastq); done
```
This loop iterates on each file finishing by "R1.fastq", and uses this file as input as well as the file with the same name but finishing by "R2.fastq" instead.
Using the same logic, it also creates two output files with names finishing by R1_paired.fastq and R2_paired.fastq

The Trimmomatic command can be designed the same way, carefully. For instance:
```
for f in *R1_001.fastq; do (java -jar ~/software/Trimmomatic-0.36/trimmomatic-0.36.jar PE -phred33 $f ${f/R1_001.fastq}R2_001.fastq ${f/R1_001.fastq}R1_001_Tpaired.fastq ${f/R1_001.fastq}R1_001_Tunpaired.fastq ${f/R1_001.fastq}R2_001_Tpaired.fastq ${f/R1_001.fastq}R2_001_Tunpaired.fastq ILLUMINACLIP:~/software/Trimmomatic-0.36/adapters/TruSeq3-PE-2.fa:1:30:7:2:true MAXINFO:40:0.85 MINLEN:36); done
```
**BE CAREFUL!**  
You need to understand the above command and adapt it to our needs/input names!  
THE ORDER OF THE OUTPUT FILES MATTERS!


## **4. Choosing a strategy to assemble the data, and formatting the reference file**

### Assembly strategy
There are multiple pipelines to analyze target capture sequencing data and to produce phylogenies from them.  
At Kew, we often use **HybPiper** or a homemade pipeline inspired by it: **PAFTOOLS**. However, there may be more adequate pipelines depending on your needs, or, more likely, you will need to customize an existing pipeline.  
  
HybPiper has the advantage to be relatively easy to customize, his author [Matt Johnson](https://github.com/mossmatters) is very helpful, and the pipeline allows us to retrieve the [splash-zone](https://github.com/sidonieB/bioinfo-utils/blob/master/docs/advice/images/Fig3_splash_zone.jpg), which is of interest to many people working at Kew.  
What follows is based on Hybpiper. If you want to use PAFTOOLS, talk to Jan Kim.  
  
You should read about HybPiper [here](https://github.com/mossmatters/HybPiper) and [here](https://github.com/mossmatters/KewHybSeqWorkshop) before trying it by yourself, because the original tutorial is very detailed and useful, and because it may contain updates that we overlooked (please let us know!). Below we only give some additional tips.

HybPiper and PAFTOOLS both:
- align (map) the reads on the reference sequences of the genes that you want to retrieve, 
- keep the reads that align well to the target genes, as well as their mates, 
- assemble separately the groups of reads corresponding to each gene to generate a consensus for each gene.  
  
However, even if you do target capture, it may be a good idea to map the reads on a full genome, where your targets are annotated. This may improve the read mapping, avoid artefacts created by wrong mappings, and make easier the recovery of the regions flanking the genes of interest. Let us know if you test it!
  
  
### Reference files
For HybPiper, you need to provide a **reference file** containing the sequences of the genes that you are targetting in a fasta format.  
Ideally the file should contain the regions on which you created baits, and only them.  
In general, people give protein coding sequences, i.e. exons concatenated together.  
The format should be **exactly** as following (in the same order, with the hyphen):  
\>ReferenceSpecies-geneName  
AAAAAAATTTTTTTTTGGGGGGGGGCCCCCCCC  
This allows to provide reference sequences of a same gene from different species, and to align homologous genes to each other later.  
  
You can ask HybPiper to retrieve other things than your target. For instance you can provide plastid genes, or even complete plastid genomes, formatted the same way.  
  
**To create a reference file for plastid regions**, there are multiple approaches.
You can download a plastome in Genbank, in format .gb, open it in Geneious, use the annotation tool to extract all regions that you are interested in, and rename them as required by HybPiper using the command **sed**, or a smart text editor ([BBEdit](https://www.barebones.com/products/bbedit/), [Notepad++](https://notepad-plus-plus.org/), etc.) that allows you to use regular expressions to perform complex find-replace operations.    
When you want to do it for multiple plastomes, or if you don't have access to Geneious, you can use/customize one of our [scripts](https://github.com/sidonieB/bioinfo-utils/tree/master/docs/advice/scripts), to extract only the regions you want based on the annotations, and rename them as you wish.
  
Depending on your input it will be relevant or not to use the "intronerate" option without modifications (see below).
If you choose to use blast, your reference file will have to provide amino-acid sequences, so it may not make sense to use blast for something else than coding sequences.
However, we found out empirically that blast allows to recover less reads, but longer genes, than with bwa, regardless if sequences were coding or not.


## **5. Tips when running Hybpiper**

### Name list
Hybpiper relies on a namelist, which is a list of all the samples on which you want to run hybpiper.  
The names in the list should contain **all** the part of the read file names that is **unique to each sample**.  
To make the list, you can use a loop such as:
```
for f in *_R1_001_Tpaired.fastq; do (echo ${f/_R1_001_Tpaired.fastq} >> namelist.txt); done
```
Look inside the namelist.txt file using **less**, and check that each line contains only one name and that this name contains the unique identifier of each species (and all of it). For instance, in the case of three samples:  
Genus_speciesA_library12  
Genus_speciesB_library13  
Genus_speciesC_library14  
or  
Genus_speciesA_library12_L001  
Genus_speciesB_library13_L001  
Genus_speciesC_library14_L001  
but NOT:  
Genus_speciesA_library12_L001_R1  
Genus_speciesA_library12_L001_R2  
Genus_speciesB_library13_L001_R1  
Genus_speciesB_library13_L001_R2  
Genus_speciesC_library14_L001_R1  
Genus_speciesC_library14_L001_R2  
(Because of the way HybPiper works, see command below)  
and NOT:  
Genus_speciesA_  
Genus_speciesB_  
Genus_speciesC_  
(Because you miss a part of the unique identifier)  
  
    
### PATH
Depending on where you run your analyses, you may need to put some programs in your **[PATH](https://kb.iu.edu/d/acar)**, so that HybPiper can find the program.  
To check what directories are your PATH (and thus findable by HybPiper), type:
```
echo $PATH
```
If you don't see the path to the directory containing the program you are interested in, you can use the **export** command to put the directory in the PATH:
```
export PATH=$PATH:/path/to/directory/containing/the/program
```
This creates a new variable called PATH which contains what was before in the variable PATH, and the path to the program of interest.
  
You have to run the export command each time you open a new terminal, or you have to modify a particular file in the server to make it permanent. Ask the informatician in charge.
  
**Before you panic** next time that you get an error from HybPiper, please check that:
- your reference file is correctly formatted
- your namelist is correct
- the programs used by HybPiper (bwa, blast, spades...) are in your PATH  
And try to understand what Hybpiper is complaining about.  
Typically if it does not find a file, it is either one of the two latter problems, and/or your HybPiper command is wrong, usually in the way you specified the input.  
  
### Typical command
Example of HybPiper command (adapt to your needs, and read the HybPiper documentation to understand the options!):
```
while read name 
do ~/software/HybPiper/reads_first.py -b ReferenceTargets.fasta -r "$name"R*Tpaired.fastq --prefix $name --bwa
done < namelist.txt
```

This is also a loop, and the input is provided at the end using "< namelist.txt".  
For each line in "namelist.txt", we create a variable $name which contains the line (in our case the identifier of each sample), and we apply a command (in our case the hybpiper command) on an input whose name is created using the variable $name + what is needed to find the input files corresponding to the sample (in our case "R\*Tpaired.fastq", note the * which allows to pick the file with the reads 1 and the file with the reads 2 at once)

If you want to use unpaired reads in hybpiper, you need to have all unpaired reads of one sample in one file, so if you used Trimmomatic and you have two files of unpaired reads for each sample, you can use a loop to concatenate them, such as:
```
for f in *R1_001_Tunpaired.fastq; do (cat $f ${f/R1_001_Tunpaired.fastq}R2_001_Tunpaired.fastq > ${f/R1_001_Tunpaired.fastq}TunpairedAll.fastq); done
```
  
### Retrieving the splash zone
You can retrieve the flanking regions of the target regions using Matt Johnson's intronerate script from the HybPiper directory.
```
while read name
do python intronerate.py --prefix $name
done < namelist.txt
```

### Gathering (supposedly) homologous regions from all samples
After HybPiper has run, you can make a file for each target region provided in the reference file, using Matt Johnson's retrieve_sequences.py script from the HybPiper directory.  
Each file will contain supposedly homologous regions from all samples.  
  
```
python retrieve_sequences.py ReferenceTargets.fasta . dna
```
The dot is the current directory.    
The "dna" option tells the script to retrieve only the sequences corresponding to the targets, not the flanking regions.  
  
To make files with only the flanking regions for all samples, you can use the "intron" option:
```
python retrieve_sequences.py ReferenceTargets.fasta . intron
```
To make files with the target and its flanking regions for all samples, you can use the "supercontig" option:
```
python retrieve_sequences.py targetsProbeAlignmentBaitOnly.fasta . supercontig
```

Using the "intron" and "supercontig" options will work only if you run the intronerate.py script before!  

The intronerate.py script generates:  
- a sequence corresponding to the concatenation of the flanking regions of the target (the "intron" in HybPiper's language)
- a sequence corresponding to the flanking region(s) + the target (the "supercontig" in HybPiper's language)

Example:  
  
Original sequence (only the xxx parts were baited: they are the target regions):  
---------xxxxxxxxxxxx----------xxxxxxxxxxxxxxxxxxxxx------------
  
dna:  
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  

intron:  
\-------------------------------  
  
supercontig:  
---------xxxxxxxxxxxx----------xxxxxxxxxxxxxxxxxxxxx------------  
  
  
This is an ideal case when one manages to retrieve the complete target and the complete flanking region, but you may just retrieve fragments of them, or you may retrieve longer sequences than what you expect if there was an insertion in the sequenced species compared to the reference.


### Running long or heavy jobs
Running hybpiper on many samples takes time! You should first try it on two samples to see if your commands work before submitting a long job.  
  
The [HybPiper HPC wiki](https://github.com/mossmatters/HybPiper/wiki/HPC) has information on running on a cluster, ask your informatician.  
If you are running HybPiper on lots of samples/genes and you have a decent size cluster available, it seems to make more sense to parallelise it as an array job over as many cpus as you have samples. This has worked well for us.  
  
The --cpu option (which takes a number of cpus to use) is useful if you don’t want to gobble all your resources: you should set it to less than the full number of cores your machine has, and ideally leave some cores for other users (see [best practices](http://rbg-kew-bioinformatics-utils.readthedocs.io/en/latest/cluster/__intro/)).
  
If you are running jobs on servers that don't have a system of job submission, you can use some tool to be able to close your connection to the server while the job is still running.   
For instance, you can use screen (official documentation [here](https://www.gnu.org/software/screen/manual/html_node/index.html)):  
```
#Launch screen:
screen
#Create a new window:
ctrl-A c
#Go to the next or the previous window:
ctrl-A n 
ctrl-A p 
#Detach from the screen session:
ctrl-A d
#Reattach to the screen session (you may have to indicate the session id, see error message if any):
screen -r
#Close a single window (you will lose what is running inside):
exit
#Know the screen session(s) id(s):
screen -ls
#Kill the whole screen session (you will lose what is running in all windows):
screen -X -S session-id quit
```
Ensure you keep control of your screen windows, delete the windows and kill the sessions when you don't use them anymore.

### Checking for paralogs
You can check for paralogs using Matt Johnson's paralog_investigator.py script in the HybPiper directory, in the same kind of loop as for the main HybPiper command:
```
while read i
do
echo $i
python ~/software/HybPiper/paralog_investigator.py $i
done < namelist.txt
```
Depending on the result you may want to discard a gene for a species or for all species.  
This can also be a first step to identify recent whole genome duplications, or ancient gene duplications.  

The following command produces a summary table with the number of copies of each gene:
```
parallel "python paralog_retriever.py namelist.txt {} > {}.paralogs.fasta" ::: < gene_list.txt 2> paralog_summary.txt
```
where gene_list.txt would be the list of target genes, one per line.

### General advice: 

#### Be up to date
Ensure that you don't miss an important update of HybPiper by consulting the website regularly.  
Consult also this website for new solutions. And please don't hesitate to contribute!  
  
#### Organize yourself  
Make folders corresponding to the different steps, and move the relevant files in them.  
For instance you may have a folder with the raw data, a folder with the trimmed data, a folder with the output of a given HybPiper run, a folder with the final gene files...
  
#### Tidy up  
HybPiper provides a cleaning script cleanup.py that removes unnecessary files. In addition, after a few trials you should be able to know what files are really necessary and what files can be deleted or at least moved to an external, long storage place.  
  
#### Work in a reproducible way, for you and for others  
At least, keep track of your commands. You can consult/save the output of the **history** command, or just take notes of your commands.  
Putting all commands in a file will allow you to run them all at once, or at least to come back to them and copy paste them quickly in the terminal following your needs. This is the first step towards building your own pipeline.  

## 6. Assess target capture efficiency

After you retrieved all the target regions (including or not flanking regions) for all the samples, you may want to know how much you recovered for each sample.  
There are different measures that can give you information about the success of the whole process, from the hybridization to the sequence recovery.  
Depending on what you look at, you may be able to tell if a low recovery comes from a problem during the DNA capture, unsufficient sequencing and/or unefficient read mapping or assembly.  


### Number of regions recovered for each sample, and length of the recovered regions

See Matt Johnson's script get_seq_lengths.py  
Example of command, to run from the directory where you put the HybPiper output directory(ies):
```
python ~/software/HybPiper/get_seq_lengths.py ReferenceTargets.fasta namelist.txt dna > seq_lengths.txt
```
Replace dna by aa if the reference file contains amino-acids.

The table seq_lengths.txt can then be visualized as a heatmap using Matt Johnson's script gene.recovery.heatmap.R, also available in the HybPiper directory.  
Just open the script in R and follow the instructions at the beginning of the script.

As an alternative to estimate the capture success, the **max_overlap** R script calculates three statistics using the output of the get_seq_lengths.py script:  
- representedness = proportion of species and genes for which sequences were obtained
- completeness = proportion of each target sequence obtained for each species
- evenness = measure of how evenly the sequence lengths are distributed across species and genes. This is adapted from a measure of species evenness described in [Pielou, 1966](https://www.sciencedirect.com/science/article/pii/0022519366900130).  
  
This script also generates a csv file which you can open in a spreadsheet programme of your choice to create a heatmap with appropriate conditonal formatting. This also allows you to easily zoom in on any sample or gene that you have a particular interest in.

### Number of reads ON/OFF target
See Matt Johnson's script hybpiper_stats.py  
Example of command:
```
python ~/software/HybPiper/hybpiper_stats.py seq_lengths.txt namelist.txt > stats.txt
```
You will need the result of the previous script (get_seq_lengths.py) first.    
**WARNING:** if you get 50% for everything, you are probably using a wrong, outdated version of the script. Get the newest one from the HybPiper github.

### Coverage of the recovered regions

The **coverage** of a given region for a given sample can be expressed in different ways:
- Read number: Number of reads used to assemble the region (not very informative, but easy to get)
- **Average coverage**: (number of reads used to assemble the region\*average read length)/region length
- **Read depth** of a particular nucleotidic position: number of reads that cover this position
- Average (or minimal, or median, or maximal...) read depth: average (or minimal, or median, or maximal...) of all read depths for the region.   
Depending if one considers the total region to be the region recovered or to be the reference region, these numbers will change.

The sequencing depth, or genome coverage, is usually understood as: (total read number\*read length)/ haploid genome size. It is usually expressed as "...x", where x means "times the haploid genome size". For instance, "a coverage of 3x", means that you generated 3 times as many bp as there are in the haploid genome.  
Because the sequencing is random, because there are organelle genomes in multiple copies, and because AT rich and AT poor regions are more difficult to sequence, you cannot expect the whole genome to be covered by 3 reads when you plan a 3x coverage. But it gives you an idea.  

Below are tips to access coverage information.  
Ideally you should familiarize with SAM and BAM files before trying the commands below. Read the documentation [here](https://samtools.github.io/hts-specs/).  

To calculate read depth, you need to have reads mapped on your reference region (or recovered region). To do this, you can use [bwa](http://bio-bwa.sourceforge.net/) or [bowtie](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml), for instance.  
Below are some examples using bwa, but they should not be reproduced as such, you need to adapt following your needs.
You first have to index the reference:
```
for f in *supercontig.fasta; do (bwa index $f); done
```
Then you do the mapping (play with parameters!):
```
for f in *supercontig.fasta; do (bwa mem -t 6 -a $f ${f/supercontig.fasta}interleaved.fasta > ${f/.fasta}\_BWA.sam); done
```
These steps can be memory and space consuming, you may need some strategy to parallelize the job!

Once you have your sam or bam file with the mapping information, you can do various operations to get read depth information, using [samtools](http://www.htslib.org/):

Use samtools to convert the sam to bam and to sort the bam
```
for f in *BWA.sam; do (samtools view -b $f -o ${f/.sam}.bam); done
for f in *BWA.bam; do (samtools sort $f -o ${f/.bam}_sorted.bam); done
```
Use samtools to get some basic stats for each region, put all stats in one single file for each species
```
for f in *BWA_sorted.bam; do (samtools index $f); done
for f in *BWA_sorted.bam; do (samtools idxstats $f >> All_basic_stats.txt); done
```
Use samtools to get some coverage stats for each gene, one file per gene
```
for f in *BWA_sorted.bam; do (samtools stats -c 0,1000,1 $f > ${f/.bam}_covstat.txt); done
```
Use samtools to get read depth at each position for each gene, one file per gene
```
for f in *BWA_sorted.bam; do (samtools depth -a $f > ${f/.bam}_depth.txt); done
```
This may give wrong results if your mapping is not good, for instance if many reads map where they should not map (frequent if you map reads from genome sequencing on a particular region instead of the whole genome).  
To avoid this, you have to tweak the read mapping parameters, and you can also play with some filtering options of the depth command.  
Alternatively, you can filter out the wrong matches using custom scripts.
  
You can use samtools tview to see the sorted.bam alignment in the console.  
You can use samtools phase to phase reads.  

You can use [picard](https://broadinstitute.github.io/picard/) markduplicates if you want to remove duplicates for unpaired reads (or reads where the paired-end information is not used).  

Use samtools to create a **pileup**, i.e a file with information about how many reads match each base and how well they match.
Use -ff to filter only the reads following a particular condition (see the samtools mpileup documentation):
```
for f in *BWA_sorted.bam; do (samtools mpileup -A -ff UNMAP -f $f -o ${f/.bam}_pileup.txt); done
```

  
## 7. Improving target recovery

The approach followed by HybPiper may not always be optimal, or it may only become optimal after fine tuning.  
Here are some strategies that may be worth trying when the default approach does not work well:  
- Run HybPiper on a sample for which you have good data, build a better reference file based on the results, and run HybPiper again on the other samples, using the new reference file
- Use the blast option instead of bwa
- Tune the bwa parameters (you probably need to tweak the HybPiper script, or see PAFTOOLS)
- Use the --cov_cutoff option to allow SPADES to assemble contigs with less coverage than the default (8)
- Try a more or less stringent trimming strategy
