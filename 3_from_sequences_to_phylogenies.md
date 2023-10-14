# **Phylogenomics tips**

[Multiple sequence alignments](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#1-multiple-sequence-alignments)  
[Gene trees](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#2-gene-trees)  
[Spotting alignment problems by observing gene trees](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#3-spotting-alignment-problems)  
[Species tree estimation using ASTRAL](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#4-infer-species-tree-with-astral)  
[Rooting trees](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#5-rooting-trees)  
[Calculate clade support](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#6-calculate-support)  
[Visualize support](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#7-visualize-support-on-the-species-tree)  
[Dating divergence times](https://github.com/sidonieB/Workflows/blob/main/3_from_sequences_to_phylogenies.md#8-dating-divergence-times)  


## **1. Multiple Sequence Alignments**
There are many different programs to align sequences.  
None of them is guarantied to find the optimal alignment.  
Performance depends on your data and how you tune the parameters.
Below we give examples of use and remarks about the software that we tend to use the most.  
  
According to research conducted by people in [T. Warnow's lab](http://tandy.cs.illinois.edu/), [filtering out fragmentary sequences before aligning results in less gene tree estimation errors](https://www.ncbi.nlm.nih.gov/pubmed/29029241), and is thus generally beneficial since the impact of missing data on species tree estimation methods is usually [null or positive](https://www.ncbi.nlm.nih.gov/pubmed/29029338). 

### MAFFT
See the documentation [here](https://mafft.cbrc.jp/alignment/software/)  

Basic MAFFT command:  
```
mafft --thread 2 --genafpair --adjustdirectionaccurately --maxiterate 1000 "$name".fasta > "$name"_alM.fasta
```
This is just an example, look at the documentation to chose adequate options!
Note the "--adjustdirectionaccurately" option, which reverse complement sequences if necessary.  
  
Sometimes, you need to replace ```_R_``` by nothing in the aligned fasta files because if mafft reversed a sequence it appended ```_R_``` at the beginning of its name. Here is a line of code that can do that on all alignments and append ```_r``` to the name of the resulting alignment:  
```
for f in *.fasta; do (sed -e 's/_R_//g' $f > ${f/.fasta}_r.fasta); done
```

  
Example of command to run MAFFT on multiple files using GNU parallel (see Matt Johnson explanation [here](https://github.com/mossmatters/KewHybSeqWorkshop/blob/master/Alignment.md)):
This is not really needed if you run MAFFT on a HPC using an array job. See the [slurm cheat sheet](https://github.com/sidonieB/Workflows/blob/main/Slurm_cheat_sheet.md).
```
# Make a list of the gene names:
for f in *.fasta; do (echo $f >> genenames.txt); done
# look at the list:
less genenames.txt
# run MAFFT on all genes from the list:
parallel --eta "mafft --localpair --adjustdirectionaccurately --maxiterate 1000 {} > {}.aligned.fasta" :::: genenames.txt
```
    
For aligning big alignments MAFFt has a –thread option that can be set to the number of cores on your machine.   
Explained [here](https://mafft.cbrc.jp/alignment/software/multithreading.html).  
There is also a [MPI version](https://mafft.cbrc.jp/alignment/software/mpi.html).


### PASTA
See the documentation [here](https://github.com/smirarab/pasta).  
What follows needs testing and proofreading. Comments welcome!  
PASTA aligns sequences following an iterative approach that generates an initial alignment and tree, uses this tree to improve the previous alignment, and does it again a user-defined number of times (default == 3).  Alternatively, the user can provide an initial tree.
PASTA is able to deal with very large numbers of sequences to align because it splits the data in subsets of sequences, align the subsets, and then combine the alignments.  
PASTA works with different alignment software, including MAFFT.    
Using MAFFT through PASTA may give better results than using MAFFT alone, especially because PASTA tends to infer gaps instead of forcing alignment between very divergent sequences.  
Recent work by people in [T. Warnow's group](http://tandy.cs.illinois.edu/) suggest that a combination of PASTA and [BAli-Phy](http://www.bali-phy.org/) could [work well](https://bmcgenomics.biomedcentral.com/articles/10.1186/s12864-016-3101-8#Sec15) although more testing on real data is needed (pers. com. from Mike Nute, PhyloSynth symposium, Montpellier, France, August 2018).

### UPP
UPP produces more accurate alignments than MAFFT or PASTA in the case of fragmented sequences, see the documentation [here](https://github.com/smirarab/sepp/blob/master/README.UPP.md).  
UPP splits the data set into more fragmented and less fragmented sequences. It then produces a backbone alignment and Hidden Markov Models (HMM) from the less fragmented sequences and attempts to fit the more fragmented sequences into each HMM. The final alignment is then selected from the best supported HMM.  
If the range of sequence lengths is highly variable or most of your fragments are much shorter than the target reference sequence, you can use the UPP ```-M``` option, and you may need to specify something like "95th percentile of all sequence lengths in data set" as input for this option.

## **2. Concatenate alignments (if needed)**

We know of two command line tools: [FASconCAT-G](https://github.com/PatrickKueck/FASconCAT-G) and [AMAS](https://github.com/marekborowiec/AMAS). The latter seems more robust to big alignments.
The command to run AMAS is:
```
python3 AMAS.py concat -i input-file*.txt -f fasta -d dna
```
AMAS.py does some other useful things including giving a summary of an alignment with number of parsimony informative sites etc:
```
python3  AMAS.py summary -i alignment.fasta -f fasta -d dna
```

## **3. Spotting alignment problems and trimming alignments**
  
It is often a good idea to spend some time looking at your alignments.  
[FluentDNA](https://github.com/josiahseaman/FluentDNA) allows to look at many alignments and visually spot abnormalities, changes in nucleotide frequencies etc.  

Alignments can be ambiguous or problematic in many ways:  
- They can have sites (columns) with many gaps
- They can have sequences (rows) with many gaps
- They can have sequences that are mis-aligned with the rest, or that are very divergent
- They can have some regions that are mis-aligned or ambiguously aligned
  
Often, it is best to remove sites and sequences that are source of ambiguity or unreliable. The quality of the alignment will have a strong impact on the phylogenetic tree estimation!  
Below are some programs that we found useful to clean alignments.  
Ideally, you should use them on a few alignments representative of the issues found in our dataset, see how each program changes the alignments, and adapt the programs settings accordingly instead of just using defaults settings.  

#### Remove gappy sites and uninformative alignments with trimAl or optrimAl  
  
[trimAl](http://trimal.cgenomics.org/) can be used to trim out columns or sequences based on their gap content.  
For instance:
```
trimal -in concatenated.out -out concat_trimmalled.fas -automated -resoverlap 0.65 -seqoverlap 0.65
```

Another suite of tools to perform similar tasks and many others is [phyutility](https://github.com/blackrim/phyutility).  

Trimming can sometimes result in loss of informativeness. It may be worthwhile to check that the trimming parameters did not actually make things worse. For example, one can use instead [**optrimAl**](https://github.com/keblat/bioinfo-utils/blob/master/docs/advice/scripts/optrimAl.txt), which uses [AMAS](https://github.com/marekborowiec/AMAS) to explore the effect of different trimAl gap threshold values on the proportion of parsimony informative sites and amount of data loss.

##### Instructions to run [**optrimAl**](https://github.com/keblat/bioinfo-utils/blob/master/docs/advice/scripts/optrimAl.txt) on multiple alignments at once:  
  
- You need 3 files: PASTA_taster.sh, cutoff_trim.txt and optrimal.R (the scripts PASTA_taster.sh and optrimal.R can be found on the optrimAl webpage, you will need to save the two scripts provided on that page in two separate files named PASTA_taster.sh and optrimal.R. The cutoff_trim.txt is a text file that you have to make yourself, with desired trimming threshold values, one per line. Make sure you do not have \r end of line characters if you make the file in windows).
- Decide what your working directory will be in the cluster (e.g. the directory with the alignment files, or a directory with a copy of the alignments).  
- **Do not put anything else in the directory**, just the files to process and the scripts (see below)  
- Edit paths as needed in PASTA_taster.sh, depending on the working directory you chose
- Edit thresholds as needed in cutoff_trim.txt (0,0.1,0.3,0.5,0.7,0.9,1 could be good)  
- Edit the setwd path in optrimal.R, depending on the working directory you chose  
Make sure you do not change anything else and that there is no \r end of lines if you edited the files in Windows  
- Copy the three files in the chosen working directory  
- Run ```bash PASTA_taster.sh``` from that working directory (do it through slurm if on the HPC)  
If it fails at the Rscript step, run the last line: "Rscript PATH/optrimal.R" in an interactive slurm (but it should be fine)  
- Move the optimally trimmed alignments in a new folder for downstream analyses.  
There may be less alignments than before if the optimal trimming was to discard the alignment due to a lack of informativeness.  
The list of alignments lost is in overlost.txt. It is important to know because it will change the number of files to process in downstream analyses.

#### Remove spurious sequences (and sites) with CIAlign

[CIAlign](https://github.com/KatyBrown/CIAlign) can trim alignment columns and rows based on various criteria. It is especially a great tool to remove sequences that are completely spurious, which will be detected based on how different they are from the rest of the sequences. It is even possible to specify sequences to keep regardless of how different they are, for instance an outgroup sequence.  

Basic CIAlign command to remove spurious sequences:
```
CIAlign.py --infile alignment.fasta --outfile_stem alignment_prefix --remove_divergent --remove_divergent_minperc 0.85 --retain_str Outgroup_name --plot_input --plot_output --plot_markup
```

A CIAlign command to further clean the alignments using mostly default settings (partly redundant with optrimAl so best not use both together):
```
CIAlign.py --infile alignment.fasta --outfile_stem alignment_prefix --remove_divergent --remove_divergent_minperc 0.85 --remove_divergen_retain_str Outgroup_name --remove_insertions --remove_short --remove_gap_only --plot_input --plot_output --plot_markup
```

#### Remove spurious sequence stretches with TAPER

[TAPER](https://github.com/chaoszhang/TAPER) can remove spurious bits in individual sequences while keeping the rest of the sequence. As far as we know, only the default settings seem to work as expected.  

Basic TAPER command (it requires julia):
```
/PATH/julia-1.6.2/bin/julia /PATH/TAPER-master/correction_multi.jl -m N -a N alignment.fasta > clean_alignment.fasta
```
    
**We found that using OptrimAl, CIAlign, TAPER, and again OptrimAl gave satisfactorily clean alignments without excessive loss of data**  


#### Renaming sequences in all alignments
  
We have scripts for that, just ask!


  
## **4. Estimating Gene trees**

### Selecting an appropriate model of nucleotide substitution
  
This can be done efficiently in [IQtree](http://www.iqtree.org/) and the model chosen can then be indicated to [RAxML](https://cme.h-its.org/exelixis/web/software/raxml/), or in [raxml-ng](https://github.com/amkozlov/raxml-ng), which perform the "standard" (not ultrafast) bootstrap more quickly than IQtree.  
Alternatively, both model selection and gene tree estimation can be performed in IQtree, including bootstrap or ultrafast bootstrap support values estimations. The standard bootstrap estimation may take a very long time if done in IQtree.  

Basic IQtree command to select a nucleotide substitution model:
```
/PATH/iqtree-1.6.12-Linux/bin/iqtree -s alignment.fasta -m MF -AICc -nt AUTO -ntmax 2
```
Check the great documentation to make sure this fits your situation!  

  
### Editing and interpreting the output of IQtree to use it in raxml
  
This is a set of instructions to retrieve the best model of substitution selected by IQtree for a set of genes, and make a list of the gene names and a separate list of the corresponding model of substitution, in a format understood by raxml-ng.  
  
- Extract the best model from all IQtree output files: from the folder containing the IQtree outputs, run:
```
ls *.log > file_names.txt
while read f; do grep "Best-fit model" $f; done < file_names.txt >> models.txt
paste -d "\t" file_names.txt models.txt > All_models.txt
rm file_names.txt
rm models.txt
```
  
- Edit the model list to keep only the alignment file name and the model
```
sed -e 's/.fasta.log//g' -e 's/Best-fit model: //g' -e 's/ chosen according to AICc//g' All_models.txt > All_models_2.txt
```
  
- Check what models were found and if some names need to be edited to be recognised by raxml-ng:
```
cut -f2 All_models_2.txt > All_models_2_models.txt
sort -u All_models_2_models.txt > All_models_2_models_su.txt
```
  
- Download the All_models_2_models_su.txt file and open it in excel to see what models there are.  
For each model, look if the model name needs to be edited and write down what should be the new model name so that raxml-ng can understand it.   See [here](https://github.com/amkozlov/raxml-ng/wiki/Input-data#evolutionary-model) and [here](http://www.iqtree.org/doc/Substitution-Models) to find out correct model names.  
  
- Example of a command to edit the model names in the original list:
```
sed -e 's/GTR+/GTR+/g' -e 's/HKY+/HKY+/g' -e 's/K2P+/K80+/g' -e 's/K3P+/K81+/g' -e 's/K3Pu+/K81uf+/g' -e 's/SYM+/SYM+/g' -e 's/TIM2e+/TIM2+/g' -e 's/TIM2+/TIM2uf+/g' -e 's/TIM3e+/TIM3+/g' -e 's/TIM3+/TIM3uf+/g' -e 's/TIMe+/TIM1+/g' -e 's/TIM+/TIM1uf+/g' -e 's/TNe+/TN93ef+/g' -e 's/TN+/TN93+/g' -e 's/TPM2+/TPM2+/g' -e 's/TPM2u+/TPM2uf+/g' -e 's/TPM3+/TPM3+/g' -e 's/TPM3u+/TPM3uf+/g' -e 's/TVMe+/TVMef+/g' -e 's/TVM+/TVM+/g' All_models_2.txt > All_models_3.txt
```
Make sure you edit the above command to modify your own model names!  
You may need to edit the *_3.txt list locally by hand for the models such as K2P and K3P that do not have a + after the model name, because making a sed command for them like above may lead to unexpected results.  
  
- Save the edited file as All_models_4.txt (make sure that end of lines do not contain \r if you did this in Windows)

- Generate a list of models with their edited names, and a list of corresponding gene names:
```
cut -f2 All_models_4.txt > All_models_4_models.txt
cut -f1 All_models_4.txt > All_models_4_names.txt
```
The first model in All_models_4_models.txt is the model selected for the first gene in All_models_4_names.txt and so on...  
This will enable to run raxml-ng on all genes in an array job, picking the right model for the right gene.  


### Gene tree estimation using maximum likelihood

### RAxML  
  
We often use [RAxML](https://cme.h-its.org/exelixis/web/software/raxml/), see full documentation [here](https://cme.h-its.org/exelixis/resource/download/NewManual.pdf) to understand the options.  
Example of command to get gene trees with 100 bootstrap replicates, with branch lengths in the bootstrap files (option -k):
```
for f in *.fasta; do (raxmlHPC-PTHREADS -m GTRGAMMA -f a -p 12345 -x 12345 -# 100 -k -s $f -n ${f}_tree -T 4); done
```
Be careful with the -T option, which controls the number of threads to use!  
According to the manual the benfit of using -T will depend on the number of site patterns, so for an average gene it is not worth setting -T to more than 2 or at most 4, although this will depend on the model of evolution and if the sequences are nucleotides or amino-acids.  
The -p and -x options are important for reproducibility, the number does not matter but you should take note of it (see the manual).
  
If you have access to a HPC, gene trees can be produced in parallel using an array job.  
  
For concatenated alignments RaxML can also be run in MPI mode, or in HYBRID mode with parallelisation of “coarse grain” processes over nodes (e.g. building separate bootstrap trees) and “fine-grain” processes using multithreading of multiple processors on a single machine (e.g. working on a single tree). See [documentation](https://help.rc.ufl.edu/doc/RAxML).
  
The trees to be used for species tree estimation with ASTRAL (see below) are the RAxML_bipartitions.* trees, NOT the RAxML_bipartitionsBranchLabels.* trees.

### raxml-ng  

For genomic datasets, [raxml-ng](https://github.com/amkozlov/raxml-ng) may be a more efficient option.  It seems to be more accurate and quick than RAxML.  
Basic raxml-ng command:
```
raxml-ng --all --msa alignment.fasta --model model --bs-trees 100 --threads auto{8} --tree pars{30},rand{30} --seed 1 --prefix prefix
# Check that found only one tree (global optimum) instead of many (local optima)
raxml-ng --rfdist --tree prefix.raxml.mlTrees --prefix RF6_prefix
# Check convergence of bootstrap trees
raxml-ng --bsconverge --bs-trees prefix.raxml.bootstraps --prefix prefix --seed 1 --threads auto{8} --bs-cutoff 0.03

```
Above, "prefix" is for instance the name of the gene, and "model" is the desired model of nucleotide substitution (for instance the one selected by IQtree).  
  
Another accurate option is [IQ-Tree](http://www.iqtree.org/) but the ultra-fast bootstrap is less accurate and its standard bootstrap takes much more time to compute than when done in RAxML.

### IQtree  
  
[IQtree](http://www.iqtree.org/) is also a great option:  
Basic IQtree command to select a model of nucleotide substitution, and make a tree with 1000 ultrafast bootstrap replicates:  
```
/PATH/iqtree-1.6.12-Linux/bin/iqtree -s alignment.fasta -nt AUTO -ntmax 2 -bb 1000
```
  
## **5. Spotting alignment problems by observing gene trees**

When you have hundreds of alignments, looking at all of them to spot wrong alignments or weird sequences becomes difficult, so people are developping tools to spot problems automatically.  

Matt Johnson's [script](https://github.com/mossmatters/KewHybSeqWorkshop/blob/master/Alignment.md#identifying-poorly-aligned-sequences) to spot anormally long branches in trees. Blogpost about the script [here](http://blog.mossmatters.net/detecting-branch-length-outliers/).
  
U. Mai and S. Mirarab's [Treeshrink](https://github.com/uym2/TreeShrink) to do the same. We have a script to remove the outliers found by Treeshrink.

If you can have a look at your alignments don't hesitate though...  


## **6. Infer a species tree with ASTRAL**

Taxa names have to be the same in all trees (but you can have missing taxa).  

Combine all gene tree files in one file with the **cat** command: 
Example for raxml-ng:
```
while read name
do cat "$name".raxml.support >> all_trees.tre && echo "" >> all_trees.tre
done < gene_names.txt
```
Above, gene_names.txt contains the names of the genes you want to use to make the species tree. 

  
We have mainly been using ASTRAL-III (version 5.1.1 and above). See the article [here](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/s12859-018-2129-y) and ASTRAL documentation [here](https://github.com/smirarab/ASTRAL).  
However, a better option may be Weighted Astral, i.e. [wASTRAL](https://github.com/chaoszhang/ASTER/blob/master/tutorial/astral-hybrid.md), part of the ASTER suite, because it gives a higher weight to clades with higher bootstrap support values, thereby limiting the negative impact that gene tree error has on ASTRAL.  
  
When using ASTRAL-III, it is better to collapse very low support branches in the gene trees before running ASTRAL.  
It can be done with newick utilities in the following way (for branches with less than 10% bootstrap):  
```
/PATH/newick-utils-1.6/src/nw_ed all_trees.tre 'i & b<=10' o > all_trees-BS10.tre
```
Do not over collapse! S. Mirarab's work shows that using a threshold of 10 to 33% of bootstrap support to collapse clades results in less errors in the species tree than inferior or superior thresholds.  
  
  
To run Astral with the -t 0 option, and get the species tree without any annotations or branch lengths:
```
java -Xmx12000M -jar ~/software/ASTRAL/astral.5.5.9.jar -i all_trees-BS10.tre -t 0 -o SpeciesTree.tre
```
If astral had '[p=...]' annotations (other -t options, see the manual), the following commands could help removing all annotations, but it is generally easier to rerun Astral with -t 0.
```
sed 's/\[[^\[]*\]//g'SpeciesTree.tre > SpeciesTree2.tre
sed "s/'//g" SpeciesTree2.tre > SpeciesTree3.tre
```

To run Astral with all annotations:
```
java -jar /PATH/astral.5.7.5.jar -i all_trees-BS10.tre -t 2 -o SpeciesTree_annotQ.tre
```

The ASTRAL option ```-t 10``` does a polytomy test and may help in assessing if a short branch could reflect insufficient data or be due to a true polytomy. See the corresponding paper [here](https://arxiv.org/abs/1708.08916).  



## **7. Rooting trees**

**WARNING!** If you use phyparts (see below) or more generally if you have to compare gene trees to your species tree, it is important that the same rooting method is applied to all trees.

### Root the species tree 

We use the command pxrr of the [phyx](https://github.com/FePhyFoFum/phyx) package, because this is what [S. Smith](https://bitbucket.org/blackrim/) uses so it should work with phyparts (see below):
```
~/software/phyx/src/pxrr -t SpeciesTree.tre -g outgroup_name_as_in_the_tree > SpeciesTree_PxrrRooted.tre
```
One can also use [newick utilities](http://cegg.unige.ch/newick_utils):
```
nw_reroot SpeciesTree.tre outgroup_name_as_in_the_tree > SpeciesTree_NUrooted.tre
```

For phyparts (see below), you need to ensure that the end of the species tree has a ";" and that it finishes with \r\n:
```
sed 's/\;\n/\;\r\n/' SpeciesTree_PxrrRooted.tre > SpeciesTree_PxrrRooted_formated.tre
```
### Root many gene trees at once

You can also use the pxrr command from [phyx](https://github.com/FePhyFoFum/phyx) to root multiple trees at once, and it is also relatively flexible when you need to use multiple outgroups. See the documentation [here](https://github.com/FePhyFoFum/phyx/wiki/Program-list).  
  
If you have multiple and different outgroups per gene tree, and some risk that your outgroups may not be monophyletic in some trees, you can also use our custom R script to generate the adequate command for pxrr for each tree, and then run all commands at once. 

This R script, called root_tree_general_pxrr_vx.R, will generate a command to root each tree on the first preferred taxon that is available.
Input for the script:  
Based on your species tree, create a list with the outgroups, for instance called outgroups.txt. Ensure that:
- Each outgroup is on a separate line
- If the outgroup is a clade, put all taxa of the clade on the same line separated by " ; "
- Each outgroup (and not each taxon) is enclosed in quotes
Such as:  
"OG1"  
"OG2"  
"OG3 ; OG4 ; OG5"  
"OG6"  
"OG7 ; OG8"  
"OG9"  
etc  

Put the list of outgroups in the same folder as the gene trees, avoid puting anything else in the folder.
  
When an outgroup clade is found not monophyletic in the tree, the script looks for the largest combination of the outgroups that is monophyletic and use it to root the tree.  
It will also generate a warning so that you can go and check the tree and the corresponding pxrr command if you want to be sure.  

Once the pxrr commands are generated and you are happy with them, you can run them all at once.  
  
Ensure that the end of all rooted gene trees has a ";" and that it finishes with \r\n:
```
for f in *.tre; do (sed 's/\;\n/\;\r\n/' $f > ${f/.tre}2.tre); done
```

## **8. Estimate confidence in the tree and clades**

### Displaying ASTRAL support values on the tree
  
ASTRAL provides various measures of clade or bipartition [support](https://github.com/smirarab/ASTRAL/blob/master/astral-tutorial.md#branch-length-and-support).  
They can be displayed on the tree using [Figtree](http://tree.bio.ed.ac.uk/software/figtree/) or custom R scripts.  

To interpret the support values provided by Astral, look at [S. Mirarab](https://github.com/smirarab/ASTRAL/blob/master/astral-tutorial.md#branch-length-and-support) website.  

[DiscoVista](https://github.com/esayyari/DiscoVista) can help you explore conflicts among gene trees, quartet by quartet.  


### Displaying bipartition support values on the tree using Phyparts

You can also use [phyparts](https://bitbucket.org/blackrim/phyparts) to obtain a measure of the support for each bipartition in the tree.  
  
**WARNING!**: To use phyparts properly, the species tree and the gene trees have to be rooted using the same rooting method!  
If you generate the species tree with ASTRAL (see above), use the -t 0 option to not have annotations.  

Example of phyparts command:
```
java -jar /PATH/target/phyparts-0.0.1-SNAPSHOT-jar-with-dependencies.jar -a 1 -s 70 -v -d gene-trees/ -m species_tree.tre -o out-prefix
```
Use the -s option to collapse clades with a bootstrap percentage lower than the number indicated (here 70%).  
  
Alternatively, you could collapse clades before loading the trees into phyparts, using newick utilities, and concatenate the collapsed gene trees into one file:
```
for f in *.tre; do (nw_ed $f ‘i & b<=70’ o > ${f/.tre}_collapsed70.tre); done
cat *collapsed70.tre > all_trees.tre
```
  
- Put the output from phyparts in a separate directory in order to visualize it.
- You can generate piecharts corresponding to the results from the phyparts analysis using Matt Johnson's [script](https://github.com/mossmatters/phyloscripts/tree/master/phypartspiecharts)   
- Indicate the path to ete3:
```
export PATH=/PATH/anaconda_ete/bin:$PATH
```
Run the script:
```
xvfb-run python phypartspiecharts_MJohnson.py species_tree.tre phyparts-out/out-prefix gene-number
```
The ""xvfb-run" is sometimes necessary if you run it remotely, but not needed if you run it locally (for instance if you installed ete3 on your computer).  
  
- To understand how to interpret the pie charts provided by phyparts, see [S. Smith](https://bitbucket.org/blackrim/phyparts) and [M. Johnson](https://github.com/mossmatters/phyloscripts/tree/master/phypartspiecharts) websites. Assuming no change in colors from M. Johnson's original script:
  - Blue: Support the shown topology = percentage of gene trees concordant with the shown topology  
  - Green: Conflict with the shown topology (most common conflicting bipartion) = percentage of gene trees showing the most common conflicting topology  
  - Red: Conflict with the shown topology (all other supported conflicting bipartitions) = percentage of gene trees showing any other conflicting topology  
  - Gray: Neither concordant nor conflicting with the shown topology = percentage of gene trees showing neither support nor conflict with the shown topology. Includes topologies compatible with (but not supporting) the shown topology, and topologies supporting or conflicting the shown topology but with low bootstrap support ("low" being what you set up during phyparts analysis).  
  - Numbers above branches: number of gene trees concordant with the shown topology (blue)  
  - Number below branches: number of gene trees conflicting with the shown topology (red + green)  



## 9. Dating divergence times

**DO NOT** use ASTRAL branch lengths (see S. Mirarab [github](https://github.com/smirarab/ASTRAL/blob/master/astral-tutorial.md#branch-length-and-support) for explanations and for coming-soon approach to date phylogenomic datasets)
