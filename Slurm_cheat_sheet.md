# Slurm tips
Slurm is a system that manages the jobs submitted by users to a High Performance Computing facility. See more explanations [here](https://help.cropdiversity.ac.uk/slurm-overview.html) and [here](https://slurm.schedmd.com/overview.html). There is a lot of information but you will benefit from taking some time to read it.

## Interactive mode  
Although most jobs will be run in the background, it can be useful to also use slurm to run short tasks. Typically, these are the tasks that you would usually just run on the front node of the HPC, without bothering to submit them as a job. However, even small and short tasks can be too much for a front node, especially if many users are using it, and depending on how it is configured. So it is good practice to also run these through slurm using the interactive mode, and specifying only a very low amount of memory and cpu (as appropriate for your task).  
To start an interactive job on the queue for short jobs with 2 CPUs and 2 GB of memory, use:
```
srsh --partition=short --cpus-per-task=2 --mem=2G
```
To detach from the job, use ctrl-D. You can also cancel the job as any other job.

You can also submit a command directly to slurm, instead of first opening an interactive window. The command will then be ran in the background, as it would be when using a script (see below).  
```
sbatch --wrap --partition=short "gunzip *.gz"
```

## Slurm script
Generally, you will want to put the commands to run your job in a script that will then be submitted to slurm using the following command:
```
sbatch slurm_script.sh
```

All the info needed for slurm will then be inside the script, on the lines starting with "#SBATCH", which will be recognised as instructions by slurm.  
Here is a heavily commented example of a slurm script to check the quality of sequencing reads for 6 samples using fastqc through a slurm array of 6 jobs.  

```
#!/bin/bash
#
#SBATCH --chdir=/PATH/directory           # This is the folder where you want slurm to look for the input files (it will also be where the output files are written, except if you specify something else in the script below)
#SBATCH --job-name=fastqc                 # This is just a name to give to the whole slurm job, it can be anything, it will appear when you check the job status with "squeue"
#SBATCH --partition=medium                # use short if you expect a single job to last <6 hours, medium if you expect it to last <24 hours, long otherwise. Partition names and time limits will change depending on your HPC!
#SBATCH --array=1-6                       # if you want to run the script below on 6 samples in parallel (i.e. to run 6 jobs in parallel), this is the option for it, change the number depending on your number of samples/jobs, delete the line if you do not want to run jobs in parallel. NB, you can decide to run 6 jobs but only 2 by 2 if you set it to --array=1-6%2 This is useful when you have many jobs so that you do not monopolise too many ressources at once (see also below my comments about the cpus and mem options)
#SBATCH --ntasks=1                        # this is the number of tasks per job
#SBATCH --cpus-per-task=2                 # this is the number of cpu's to dedicate for a given job (sample) so careful when running many in parrallel as an array: if you set this to 2 and you ask for 6 jobs in parallel, it will attribute 2*6=12 CPUs to you, preventing others to use them: so think/test on a few samples before requesting too much!
#SBATCH --mem=8G                          # this is the number of GB of RAM to dedicate for a given job (sample) so careful when running many in parrallel as an array: if you set this to 8 and you ask for 6 jobs in parallel, it will attribute 8*6=48 GB to you, preventing others to use them: so think/test on a few samples before requesting too much!
#SBATCH --mail-user=xxxx@zzz.org          # put your email address to receive updates about your slurm job
#SBATCH --mail-type=END,FAIL              # this is to get an email only when the whole slurm job is finished or failed (see documentation for different options)


# The 3 lines below are only needed when you run your script on many samples at once (array job, using the --array option, here ran on 6 samples)
# Normally when you do an array job, slurm uses the numbers given in the --array option as input. It sees it as a list of IDs, here it would be [1,2,3,4,5,6], meaning that it has to run the job 6 times (on 6 samples) in parallel
# In our case, the numbers 1 to 6 do not allow us to tell slurm what samples to run the job on, because our samples (and associated input files) are not numbered from 1 to 6, but instead they have species names
# so we use the 3 commands below to make the link between the job IDs and the input files needed for each job 1 to 6.
# The command "echo $SLURM_ARRAY_TASK_ID" prints the job number (ID) from the array list in the standard output file (slurm_jobID.out), which can be useful for debugging; in our case it will first print "1" in slurm_jobID_1.out, then "2" in slurm_jobID_2.out etc until "6", one by one. The *.out files will be printed in the folder set using --chdir above. They are output files generated by slurm in addition to the fastqc output (basically they capture what fastqc would normally print on your screen, including when there are errors).
# for each job number (ID) it reads, slurm will then do the next two commands:
# The command "name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' names_root.txt)" sets a variable $name equal to the line of the file name_root.txt that corresponds to the job ID, so for instance if job ID is "1", slurm will read the first line of the file names_root.txt and the variable $name will be that first line (for instance, $name for the first job can be "Genus-species-SBL56_3263-S41" or whatever you use as names for your samples)
# "names_root.txt" is an input file that you create yourself. It contains the names of the sample files that you want to process, either complete or just a part of the name, depending on what is most convenient depending on the next commands (in the example the name is not complete, it is just the "root", without the "R1.fastq" or "R2.fastq" part)
# The command "echo $name" just prints the content of $name in the slurm output file, which can be useful for debugging
# in our case it should print "Genus-species-SBL56_3263-S41" in in slurm_jobID_1.out, "Genus-OtherSpecies-PL35" in slurm_jobID_2.out etc

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' names_root_nctry.txt)

echo $name

# Below your own commands finally start, in this case a simple fastqc command (you need to adapt the path to your own case, either provide an absolute path (i.e. starting by /mnt...) or a path relative to the folder set using --chdir above - I suggest that you use absolute paths whenever possible, it will save you headaches in the future)
#"$name" is the variable set when you do an array job, for instance for the first job, it will run the command /path/fastqc Genus-species-SBL56_3263-S41*.fastq, so it will run fastqc on all files present in the directory set using --chdir that start with Genus-species-SBL56_3263-S41 and finish by "fastq". If there are two files (typically R1 and R2) satisfying this condition, it will run fastqc on both, one after the other. Meanwhile, in parallel, the same will happen for "genus-OtherSpecies-PL35", etc.
# if you do not do an array job, you can replace "$name" by your sample name, or by a *.fastqc (in the latter case, slurm will run fastqc on all *.fastqc files, but it will do it one after the other, not in parallel, which will take more time)

/path/fastqc "$name"*.fastq

# NB, for programs when you need to specify the name of the output, you can also use "$name", for instance with --output "$name"_blablaOutput.fasta in which case the output will have the same root name as the input, which is very convenient when processing multiple samples

```

**NB:** On the cluster we work, CPU sensu slurm are called threads, and there are 2 threads that can be allocated per CPU sensu our cluster.  
Slurm will not occupy a CPU with two threads from a different job or from a different task of a same job, so it is unefficient to specify an odd number in --cpu-per-task since the remaining thread with not be used.


## Obtain information on a job  
Pasted/adpted from [here](https://help.cropdiversity.ac.uk/slurm-overview.html).

Check if the job is running and what other jobs are running/pending:

```
squeue
```

Details on the job:
```
sacct -j <jobid> --units=G --format JobID,MaxVMSize,MaxRSS,NodeList,AllocCPUS,TotalCPU,State,Start,End
```
  
JobID - the ID of the job  
MaxVMSize - how much memory the job requested, but did not necessarily fill up (including any swap usage)  
MaxRSS - the maximum real memory used by the job  
NodeList - the compute node that ran the job  
AllocCPUS - how many CPUs were allocated  
TotalCPU - the total CPU time used by the job, which will often be less than the runtime, especially if the job spent time waiting on user interaction or disk I/O  
State - the job’s exit state (failed or completed, etc)  
Start - start time of the job  
End - end time of the job  
  
To see the node usage
```
snodes
```

To just see when the job is likely to start:
```
sbatch --test-only slurm_script.sh
```

To cancel a job:
```
scancel <jobid>
```


**Important!**  
- On the cluster we use, short and medium queues will automatically kill a job if it exceeds their limits (6 and 24 hours respectively). 
NB: Each job of an array job has its own time allocation, so it is possible to run a long array job on a short or medium queue if every single job lasts less than 6 or 12 hours.

- It is good practice to first check what is already running/pending on the cluster before you submit a job

- Out of courtesy, if your array job has many jobs and many people use the cluster, specify a maximum number of jobs (eg. 50) to be run at once. This can be done using the "%" separator, such as:
```
--array=1-1000%50 for an array of 1000 jobs where only 50 can be run simultaneously
```

## Examples of commands
These are examples of commands used to analyse target capture data through the use of array jobs or single jobs submitted to slurm.  
"PATH/PATH" has to be replaced by the true path in your situation.  
The commands will need to be adapted to your situation/dataset!  
The commands need to be preceded by the right instructions for slurm, and, if relevant, by the array job ID specification and $name variable setting as shown above. 

Example with all that for FASTQC:
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory
#SBATCH --job-name=fastqc
#SBATCH --partition=short
#SBATCH --array=1-6
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=8G
#SBATCH --mail-user=xxx@zzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/names_root.txt) 

echo $name

/PATH/PATH/apps/FastQC/fastqc "$name".fastq

```

Unzip files:
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory
#SBATCH --job-name=fastqc
#SBATCH --partition=short
#SBATCH --array=1-6
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=8G
#SBATCH --mail-user=xxx@zzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/names_root.txt)

echo $name

gunzip "$name".fastq.gz
```


Run Trimmomatic as an array job on 3 samples
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory
#SBATCH --job-name=trimmomatic
#SBATCH --partition=medium
#SBATCH --array=1-3
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=8G      # test and decrease if too much
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/names_root.txt)

echo $name

java -jar /PATH/PATH/apps/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 /PATH/PATH/"$name"___R1.fastq /PATH/PATH/"$name"___R2.fastq "$name"___R1_Tpaired.fastq "$name"___R1_Tunpaired.fastq "$name"___R2_Tpaired.fastq "$name"___R2_Tunpaired.fastq ILLUMINACLIP:../../../../../apps/Trimmomatic-0.39/adapters/TruSeq3-PE-2.fa:1:30:7:2:true SLIDINGWINDOW:4:30 LEADING:30 MINLEN:40

# IMPORTANT: It seems that the path to the adapter file in the command above has to be given relative to the working directory (i.e. the directory specified with --chdir in the script header), instead of being an absolute path.

```





Run Secapr on all R1 files to get a plot summarizing their quality (one can do the same with the R2 files in parallel)
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/R1 # folder containing only the R1 files
#SBATCH --job-name=secaprQC
#SBATCH --partition=medium
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=16G   # decrease/test as appropriate
#SBATCH --mail-user=xxxx@xxx.org
#SBATCH --mail-type=END,FAIL


source activate secapr_env
secapr quality_check --input /PATH/PATH/R1/ --output /PATH/PATH/R1_QC_postT/
conda deactivate

```



Run Hybpiper on 1 sample
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory/
#SBATCH --job-name=hybpiper
#SBATCH --partition=short       
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=8G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL


export PATH=$PATH:/PATH/PATH/apps/ncbi-blast-2.10.1+/bin/
export PATH=$PATH:PATH/PATH/apps/SPAdes-3.14.1-Linux/bin/
export PATH=$PATH:/PATH/PATH/apps/samtools-1.11/bin/bin/
export PATH=$PATH:/PATH/PATH/apps/exonerate-2.2.0-x86_64/bin/
export PATH=$PATH:/PATH/PATH/apps/bwa-0.7.17/

/PATH/PATH/apps/HybPiper/reads_first.py -b PATH/PATH/Reference_file.fasta -r Sample_R*_Tpaired.fastq --unpairedSample_Tunpaired.fastq --prefix Sample --bwa --cov_cutoff 5

python /PATH/PATH/apps/HybPiper/cleanup.py Sample
```

Run HybPiper on 1 sample using the SSD for temporary storage and moving the output back to the permanent directory
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory/
#SBATCH --job-name=hybpiper
#SBATCH --partition=short       
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=8G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

cd $TMPDIR     # IMPORTANT CHANGE

export PATH=$PATH:/PATH/PATH/apps/ncbi-blast-2.10.1+/bin/
export PATH=$PATH:PATH/PATH/apps/SPAdes-3.14.1-Linux/bin/
export PATH=$PATH:/PATH/PATH/apps/samtools-1.11/bin/bin/
export PATH=$PATH:/PATH/PATH/apps/exonerate-2.2.0-x86_64/bin/
export PATH=$PATH:/PATH/PATH/apps/bwa-0.7.17/

/PATH/PATH/apps/HybPiper/reads_first.py -b PATH/PATH/Reference_file.fasta -r Sample_R*_Tpaired.fastq --unpairedSample_Tunpaired.fastq --prefix Sample --bwa --cov_cutoff 5

python /PATH/PATH/apps/HybPiper/cleanup.py Sample

mv Sample /PATH/PATH/directory      # IMPORTANT CHANGE
```

Run HybPiper on many samples using an array job
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory/
#SBATCH --job-name=hybpiper
#SBATCH --partition=medium      
#SBATCH --array=1-207%50   
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=8G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/HP_hybseq_names.txt)

echo $name

export PATH=$PATH:/PATH/PATH/apps/ncbi-blast-2.10.1+/bin/
export PATH=$PATH:PATH/PATH/apps/SPAdes-3.14.1-Linux/bin/
export PATH=$PATH:/PATH/PATH/apps/samtools-1.11/bin/bin/
export PATH=$PATH:/PATH/PATH/apps/exonerate-2.2.0-x86_64/bin/
export PATH=$PATH:/PATH/PATH/apps/bwa-0.7.17/

/PATH/PATH/apps/HybPiper/reads_first.py -b PATH/PATH/Reference_file.fasta -r "$name"_R*_Tpaired.fastq --unpaired "$name"_Tunpaired.fastq --prefix $name --bwa --cov_cutoff 5

python /PATH/PATH/apps/HybPiper/cleanup.py $name

```

Run HybPiper on many samples using an array job using the SSD for temporary storage and moving the output back to the permanent directory:
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/directory/
#SBATCH --job-name=hybpiper
#SBATCH --partition=medium      
#SBATCH --array=1-207%50   
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=8G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/HP_hybseq_names.txt)

echo $name

export PATH=$PATH:/PATH/PATH/apps/ncbi-blast-2.10.1+/bin/
export PATH=$PATH:PATH/PATH/apps/SPAdes-3.14.1-Linux/bin/
export PATH=$PATH:/PATH/PATH/apps/samtools-1.11/bin/bin/
export PATH=$PATH:/PATH/PATH/apps/exonerate-2.2.0-x86_64/bin/
export PATH=$PATH:/PATH/PATH/apps/bwa-0.7.17/

cd $TMPDIR

/PATH/PATH/apps/HybPiper/reads_first.py -b PATH/PATH/Reference_file.fasta -r "$name"_R*_Tpaired.fastq --unpaired "$name"_Tunpaired.fastq --prefix $name --bwa --cov_cutoff 5

python /PATH/PATH/apps/HybPiper/cleanup.py $name

mv $name /PATH/PATH/HP_out

```


Get sequences assembled by HybPiper from all samples, get sequence lengths from the HybPiper output to assess gene recovery and make a heatmap, and get some stats
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/HP_out
#SBATCH --job-name=HPgetSeqs
#SBATCH --partition=medium      ## may need medium if many species
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=4G
#SBATCH --mail-user=xxxx@zzzz.orgg
#SBATCH --mail-type=END,FAIL

python /PATH/PATH/apps/HybPiper/retrieve_sequences.py /PATH/PATH/Reference_file.fasta /PATH/PATH/HP_out dna

python /PATH/PATH/apps/HybPiper/get_seq_lengths.py /PATH/PATH/Reference_file.fasta /PATH/PATH/All_names_207.txt dna > /PATH/PATH/HP_207_seq_lengths.txt

python /PATH/PATH/apps/HybPiper/hybpiper_stats_MODIFsamtoolsCMD.py /PATH/PATH/HP_207_seq_lengths.txt /PATH/PATH/All_names_207.txt > /PATH/PATH/HP_207_stats.txt
```


Additional cleaning of HybPiper output files, more stringent. **CAREFUL, read [instructions](https://github.com/sidonieB/Workflows/blob/main/2_target_capture_data_analysis.md#cleaning-the-hybpiper-output) and understand the commands below before you try it!**
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/HP_out
#SBATCH --job-name=HPclean2
#SBATCH --partition=medium      ## may need medium if many species
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=4G
#SBATCH --mail-user=xxxx@zzzz.orgg
#SBATCH --mail-type=END,FAIL

while read name
do
rm -f $name/*/*_baits.fasta
rm -f $name/*/$name/exonerate_results.fasta
rm -f $name/*/$name/supercontig_exonerate.fasta
rm -f $name/*/$name/temp.contig.fa
rm -f $name/*/$name/temp.prot.fa
done < /PATH/PATH/All_names_207.txt

```

Run MAFFT to align sequences
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/not_aligned
#SBATCH --job-name=mafft
#SBATCH --partition=medium      ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --array=1-1211%32       ### change by your number of jobs!!!
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=4G
#SBATCH --mail-user=zzz@xxx.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/Gene_names.txt)

echo $name

mafft --thread 8 --genafpair --adjustdirectionaccurately --maxiterate 1000 "$name".fasta > "$name"_alM.fasta
```

Run TAPER to clean the alignments
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/alignments
#SBATCH --job-name=taper
#SBATCH --partition=short       ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --array=1-1211%32       ### change by your number of jobs!!!
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=2G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/Gene_names_L.txt)

echo $name

/PATH/PATH/apps/julia-1.6.2/bin/julia /PATH/PATH/apps/TAPER-master/correction_multi.jl -m N -a N "$name"_alM_r.fasta > "$name"_alM_r_cT.fasta

```

Basic script to run any custom python script as a loop, for instance to further clean/filter alignments. Could also do it in an array.
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/alignments
#SBATCH --job-name=cleaning
#SBATCH --partition=short       ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=2G
#SBATCH --mail-user=xxxx@zzzz.org
#SBATCH --mail-type=END,FAIL

for f in *.fasta; do (python /PATH/PATH/apps/scripts/Script.py $f ${f/.fasta}_clean.fasta xxxx options as required); done

```


Run IQtree to chose the best substitution model (later used to set up RAxML)
```

#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/alignments
#SBATCH --job-name=iqtree
#SBATCH --partition=short       ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --array=1-1161%50       ### change by your number of jobs!!!
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=2G
#SBATCH --mail-user=zzzz@xxxx.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/Gene_names.txt)

echo $name

/PATH/PATH/apps/iqtree-1.6.12-Linux/bin/iqtree -s "$name".fasta -m MF -AICc -mset JC69,HKY85,GTR,K80 -nt AUTO -ntmax 2
```

Run IQtree to get a tree and 500 standard bootstrap replicates (very long if using standard BP):  
To use the Ultrafast bootstrap, replace -b by -B (check command on the [IQtree website](http://www.iqtree.org/)!)

```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/alignments
#SBATCH --job-name=iqtree
#SBATCH --partition=short       ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --array=1-1161%50       ### change by your number of jobs!!!
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=2G
#SBATCH --mail-user=zzzz@xxxx.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/Gene_names.txt)

echo $name

/PATH/PATH/apps/iqtree-1.6.12-Linux/bin/iqtree -s "$name".fasta -m MFP -AICc -b 500 -nt AUTO -ntmax 8
```


Run RAxML to get gene trees with 500 bootstrap replicates:
```
#!/bin/bash
#
#SBATCH --chdir=/PATH/PATH/alignments
#SBATCH --job-name=raxml
#SBATCH --partition=medium      ## ok if one species takes less than 6 hours, otherwise use medium
#SBATCH --array=1-787%50        ### change by your number of jobs!!!
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8       ### This has to fit with the -T parameter in the command below!!!
#SBATCH --mem=4G
#SBATCH --mail-user=xxx@zzz.org
#SBATCH --mail-type=END,FAIL

echo $SLURM_ARRAY_TASK_ID

name=$(awk -v lineid=$SLURM_ARRAY_TASK_ID 'NR==lineid{print;exit}' /PATH/PATH/Gene_names.txt)

echo $name

raxmlHPC-PTHREADS -T 8 -m GTRGAMMA -f a -p 2345 -x 2345 -# 500 -k -s "$name".fasta -n "$name"_tree
```




Run ASTRAL (combine gene trees, collapse branches with less than 10% bootstrap support, run astral with and without annoations, root the species trees and format it for plotting in R):
```
while read name
do cat RAxML_bipartitions."$name"_alM_r_g_tree >> CDS_alM_r_g_c0all_trees.tre && echo "" >> CDS_alM_r_g_c0all_trees.tre
done < /PATH/PATH/c0all_names2.txt

/PATH/PATH/apps/newick-utils-1.6/src/nw_ed CDS_alM_r_g_c0all_trees.tre 'i & b<=10' o > CDS_alM_r_g_c0all_trees_BP10.tre
java -jar /PATH/PATH/apps/Astral/astral.5.7.5.jar -i CDS_alM_r_g_c0all_trees_BP10.tre -t 2 -o CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_annotQ.tre
java -jar /PATH/PATH/apps/Astral/astral.5.7.5.jar -i CDS_alM_r_g_c0all_trees_BP10.tre -t 0 -o CDS_alM_r_g_c0all_trees_BP10_SpeciesTree.tre
pxrr -t CDS_alM_r_g_c0all_trees_BP10_SpeciesTree.tre -g Dasypogon_bromeliifolius > CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_rooted.tre
pxrr -t CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_annotQ.tre -g Dasypogon_bromeliifolius > CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_annotQ_rooted.tre
sed 's/\;\n/\;\r\n/' CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_rooted.tre > CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_rooted2.tre
sed 's/\;\n/\;\r\n/' CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_annotQ_rooted.tre > CDS_alM_r_g_c0all_trees_BP10_SpeciesTree_annotQ_rooted2.tre

```






