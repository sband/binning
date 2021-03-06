Statistical binning refers to the method that we introduced in the following paper

* Shamsuzzoha Md Bayzid, Siavash Mirarab, Bastien Boussau, and Tandy Warnow. “Weighted Statistical Binning: Enabling Statistically Consistent Genome-Scale Phylogenetic Analyses.” PLoS ONE 10, no. 6 (2015): e0129183. [doi:10.1371/journal.pone.0129183](http://dx.doi.org/10.1371/journal.pone.0129183).

* Mirarab, Siavash, Md. Shamsuzzoha Bayzid, Bastien Boussau, and Tandy Warnow. “Statistical binning enables an accurate coalescent-based estimation of the avian tree.” Science 346, no. 6215 (2014). [doi:10.1126/science.1250463](http://www.sciencemag.org/content/346/6215/1250463.full)


The code provided here creates the supergene alignmetns (e.g. bins) given a set of bootstrapped gene trees, one per each gene. 

Requirements
==========
The current pipeline has been developed and used only on Linux. It should run on all nix platforms.
The current pipeline uses a Condor cluster for running jobs on distributed systems in parallel.
However, changing the pipeline to work with other clusters should be relatively straight-forward. Also, if you don't have a cluster, you can use the pipeline to build command files that can be run in serial. 

The following are required.

 * python 2.7
 - Dendropy 3.12 (note that Dendropy 4.0 or later will not work). 
 - java 


Installation:
=========
After installing the requirements, unzip the code directory to a location of choice and define `BINNING_HOME`
environmental variable to the location where the files are extracted. 

USAGE:
========
The assumption of the pipeline is that you have all your initial gene trees with support values drawn on them
in a certain directory (we will call it `genes_dir`). Inside `genes_dir`, you need to have one folder per gene. 
Inside that folder, you need to have the alignments and the gene tree with support values. The name of each 
gene (`gene_name`) is just the name of its directory. The name of the alignment file should be `gene_name.fasta`.
So, if your gene is called `300`, your alignment should be called `300.fasta`. The tree file can be named anything, 
*However*, all the trees should have the same exact name. So, for example, the tree for gene 300 could be under
directory `genes_dir/300/RAxML_bootstrap.final`, and if so, the tree for gene 301 should be under
`genes_dir/301/RAxML_bootstrap.final`. We will refer to the name of your tree file as `tree_file_name`.

Once you have your initial gene trees in this structure, to get the bins, you need to:


**Step 1:**
 
  * If you have condor, run:
    
``` 
   $BINNING_HOME/makecondor.compatibility.sh [genes_dir] [support_threshold (e.g. 50)] [pairwise_output_dir] [tree_file_name]
``` 

This generates a condor file (`condor.compat.[support]`) that has all the commands required for calculating all pairwise compatibilities.

  * If you don't have condor, instead run:
    
``` 
   $BINNING_HOME/makecommands.compatibility.sh [genes_dir] [support_threshold (e.g. 50)] [pairwise_output_dir] [tree_file_name]
```
   This creates a bash file that includes all the commands that need to be run.
    
**Step 2:** 

 * If you are using condor, run the condor file using: 

```condor_submit condor.compat.[threshold]```

 * If you used `makecommands.compatibility.sh` in the previous step, use your cluster system to run all the commands in the `commands.compat.[threshold]` file in parallel (the exact way this can be done depends on your cluster). 
   If you don't have a cluster, just run `sh commands.compat.[threshold]` and it will run all pairwise compatibility jobs in serial.  

**Step 3:** 

Once your runs from previous step have finished, it is time to build the bin definitions. Run (be sure to replace 50 with support threshold you used in step 1):

``` 
   cd [pairwise_output_dir]; 
   ls| grep -v ge|sed -e "s/.50$//g" > genes   
   python $BINNING_HOME/cluster_genetrees.py genes 50
```
  
Once this finished, you have all your bins defined in text files called `bin.0.txt`, `bin.1.txt`, etc inside the `[pairwise_output_dir]` directory. 
  You can look at these and examine bin sizes if you want (e.g. run `wc -l [pairwise_output_dir]/bin.*.txt`).

**Step 4:** 

Now is time to actually concatenate all the gene alignments for each bin and to create the supergene alignments.
   Go to the place where you want to have your supergene alignments saved, and run:
   
```
   $BINNING_HOME/build.supergene.alignments.sh [pairwise_output_dir] [genes_dir] [supergenes_output_directory]
``` 
   This will create the directory given with `[supergenes_output_directory]` and 
   will put all the supergene alignments in there. 
   For each supergene, it will also create a `.part` file that tells you what genes are put in the supergene alignment, 
   and the boundary between those genes.    
   **Note:** For bins of size 1, if they exist, a warning is issued. This warning should be taken into accont. 
   You already have a gene tree on these singelton bins, and therefore you might just want to make symlinks 
   instead of rerunning the gene tree estimation step. 
   Look at `$BINNING_HOME/build.supergene.alignments.sh` for commented lines that 
   can be adjusted to create symlinks for your file structure. 

**Step 5:** 

Now you can use your favorite tree estimation software to estimate gene trees for each of these supergenes. 
The `.part` files can be used for a **strongly recommended** partitioned analysis (e.g. using RAxML).

**Step 6:** 

Run your supergenes through your favorite summary method (e.g. ASTRAL, MP-EST, etc.). For doing this, you might want to 1) use MLBS, and 2) weight each bin by the number of genes put in that bin. See notes below for both of these extra steps.

---
### Notes:

* **MLBS:** If you would like to run a multi-locus bootstrapping (MLBS) pipeline, once you created the supergene alignments, you need to run a bootstrapped analysis for each bin in step 5 and get bootstrapped gene trees. You can then use your own scripts for building replicate inputs to the summary method, or you can use scripts we provide in [this github repository](https://github.com/smirarab/multi-locus-bootstrapping).

* **Weighting:** In step 6, you could weight each bin by its size. That is, you can repeat the supergene tree by the size of the bin. This is the **strongly recommended** approach, as our new [currently-under-review](http://arxiv.org/abs/1412.5454) manuscript suggests. To do this weighting, you need to replicate each gene tree by number of lines in the `.part` file generated for each supergene. Our multi-locus bootstrapping code has an option for weighting. Even if you are just using maximum likelihood gene trees (as opposed to MLBS), you can run our MLBS code with 1 replicate and giving it weight files, and it will create a weighted input file that you can then use. 

* **Randomizing:** In step 3, we first make a list of genes and then use this list as input to the graph coloring code. As stated in our paper, there is a randomness inherent in our binning approach, related to how ties are broken when sorting bin sizes. The order of the genes in the input file to the coloring code (here, the file called `genes`) determines how these ties are broken. Thus, for a given order of genes, our coloring code is deterministic. But the arbitrary order of genes can make a difference in the bin formation. We suggest that when computionally feasible, you randomize this `genes` file  (e.g. by using the command `ls| grep -v ge|sed -e "s/.50$//g"|sort -R > genes`) to produce various possible ways of binning and see what parts of the tree are robust to these random choices. 

* **Partitioned analyses:** Please use a partitioned analysis to get your gene trees. Note that we recommend that, when computationally feasible, you should use the `-M` in RAxML to enable a full partitioned analysis with re-estimation of branch lengths in addition to the GTR rate matrix and Gamma alpha parameter. 

Acknowledgment
========
In our pipeline, we use few scripts developed by others:

1. Our graph coloring code is a modification of [a Java code](http://shah.freeshell.org/graphcoloring/) by Shalin Shah.
2. Our compatibility code is based on an older versions of [phylonet](http://bioinfo.cs.rice.edu/phylonet) with some reverse-engineering and code modifications to increase speed and fix bugs. 
