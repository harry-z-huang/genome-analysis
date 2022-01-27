# Final Project Bio 312 111642243

# 1. Gene identification and Alignment

## Database creation

    mkdir ~/data/blast
    gunzip proteomes/*.gz
    cat  proteomes/* > ~/data/blast/allprotein.fas
    makeblastdb -in ~/data/blast/allprotein.fas -parse_seqids -dbtype prot
  
## Retrieve query protein and blast against database
First we need to move to your project directory. If you don't have a project directory, make one at this time. 

    cd ~/labs/finalproject-thundernyan
    
Retrieve the query protein

    ncbi-acc-download -F fasta -m protein XP_032221022.1
    
Blast it

    blastp -db ~/data/blast/allprotein.fas -query XP_032221022.1.fa -outfmt 0 -max_hsps 1 > XP_032221022.1.blastp.typical.out
    blastp -db ~/data/blast/allprotein.fas -query XP_032221022.1.fa -outfmt "6 sseqid pident length mismatch gapopen evalue bitscore pident stitle" -max_hsps 1 -out XP_032221022.1.blastp.detail.out
  
## Filtering BLAST output for putative homologs and removing short matches

    export evalue=0.00000000000001
    awk '{if ($6<$evalue)print $1 }' XP_032221022.1.blastp.detail.out > XP_032221022.1.blastp.detail.filtered.out
To count the number of results:

    wc -l XP_032221022.1.blastp.detail.filtered.out


## Obtain and align the gene family sequences

    seqkit grep --pattern-file XP_032221022.1.blastp.detail.filtered.out ~/data/blast/allprotein.fas > XP_032221022.1.blastp.detail.filtered.fas
  
Manually go through XP_032221022.1.blastp.detail.filtered.fas and rename definition lines to reflect the format:
“Genus_species_protein-identifier”, with "protein-identifier" being an identifying term(s) of your choosing. It is recommended you use the protein name as the identifier if available. 

I renamed my file "XP_032221022.1.blastp.detail.filtered.renamed.fas"

## Perform a global multiple sequence alignment in muscle

    muscle -in XP_032221022.1.blastp.detail.filtered.renamed.fas -out XP_032221022.1.blastp.detail.filtered.aligned.renamed.fas
    
Statistics on the alignment can be viewed with the following

    t_coffee -other_pg seq_reformat -in XP_032221022.1.blastp.detail.filtered.aligned.renamed.fas -output sim
    
The average alignment was 21.27%

View the alignment in alv:
    alv -kli --majority XP_032221022.1.blastp.detail.filtered.aligned.renamed.fas | less -RS
  
## Removed highly gapped positions in t_coffee

    t_coffee -other_pg seq_reformat -in XP_032221022.1.blastp.detail.filtered.aligned.renamed.fas -action +rm_gap 50 -out myhomologs.renamed.aligned.r50.fa
    
View again with:

    alv -kli --majority myhomologs.renamed.aligned.r50.fa | less -RS
    
# 2. Phylogenetic Analysis

## Install Newick tools, IQ-TREE, and gotree
If you already have newick tools and GNU Autoconf, skip this step. Just make a note of where it was installed for this project and substitute the location for reproductions of this project. 

    cd ~/tools
    git clone git://github.com/tjunier/newick_utils.git
    cd newick_utils/
    autoreconf -fi
    sudo make install
    
If you installed it, check with the following:

    nw_display -h

It is left to the reader to install IQ-TREE and gotree if they are not installed. 

## Create phylogeny for query gene
Go back to the project directory

    cd ~/labs/finalproject-thundernyan

Use iqtree to create a phylogenetic tree with default parameters

    iqtree -s myhomologs.renamed.aligned.r50.fa -nt 2
    
The unrooted tree can be viewed with an arbitrary root using Newick tools: nw_display

    nw_display myhomologs.renamed.aligned.r50.fa.treefile
    
The unrooted tree can be drawn in png format using gotree

    gotree draw png -w 1000 -i myhomologs.renamed.aligned.r50.fa.treefile  -r -o  myhomologs.renamed.aligned.r50.fa.png
    
## Root the tree

At this point, midpoint rooting was chosen as the preferred method of rooting the query gene because literature did not reveal any outgroup gene

    gotree reroot midpoint -i myhomologs.renamed.aligned.r50.fa.treefile -o myhomologs.renamed.aligned.r50.fa.midpoint.treefile
    
# 3. Reconciliation

## Evaluating Node Support using the Bootstrap
A full bootstrap analysis was used to evaluate support for each node. However, reproduction of this project may incentivize the use of an ultra-fast bootstrap analysis because the full bootstrap analysis takes very long. Both methods are provided below:

Use a full bootstrap analysis to evaluate support for each node. Values above 80% provide good support for a node.
Consider using the ultrafast bootstrap as the total CPU time for bootstrap was 3330.870 seconds and total wall-clock time for bootstrap was 1674.361 seconds for my run.

    iqtree -s myhomologs.renamed.aligned.r50.fa -b 100 -nt 2 --prefix myhomologs.renamed.aligned.r50.fullboot
    
In the interest of time, a ultrafast bootstrap (replace "-b 100" with "-bb 1000") can alternatively be used, where values above 95% are considered good support.

    iqtree -s myhomologs.renamed.aligned.r50.fa -bb 1000 -nt 2 --prefix myhomologs.renamed.aligned.r50.ufboot
    
## Create bootstrap supported tree
With no outgroup, a midpoint rooted tree is preferred.

    gotree reroot midpoint -i myhomologs.renamed.aligned.r50.fullboot.treefile -o myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile

To view bootstrap support:

    nw_display myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile

To produce a graphic to easily see the bootstrap values:

    nw_display -s myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile -w 1000 -b 'opacity:0' > myhomologs.renamed.aligned.r50.fullboot.midpoint.svg
    
## Gene tree and species tree reconciliation
A species tree was provided by the instructor for this analysis in the file named "species.tre". 
For reproductions of this project from scratch without the "species.tre" file, the tree is also written below.

    (((Homo_sapiens,Strongylocentrotus_purpuratus)Deuterostomia,Drosophila_melanogaster)Bilateria,(Nematostella_vectensis,Pocillopora_damicornis)Cnidaria)Eumetazoa;

Make a batch file for Notung in nano:

    nano rrn3batch.txt
    
1st line: filename for species tree "species.tre"
2nd line: filename for gene tree "myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile"

Perform the reconciliation using Notung

    java -jar ~/tools/Notung-3.0-beta/Notung-3.0-beta.jar -b rrn3batch.txt --reconcile --speciestag prefix  --savepng --treestats --events  --phylogenomics 
    
Counting the duplications and losses in rrn3batch.txt.species.tre.duplication.txt and rrn3batch.txt.species.tre.loss.txt, respectively:
10 duplications and 15 losses. Event score is 30.

After it runs, successfully, look at the reconciled gene tree: myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.reconciled.png

## View reconciled tree
First, numpy must be installed

    sudo yum install numpy
    sudo easy_install -U ete3 
    
Generate a RecPhyloXML object to view the gene-within-species tree

    python ~/tools/recPhyloXML/python/NOTUNGtoRecPhyloXML.py -g myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.reconciled --include.species
    
The cost of duplications is 1.5 and the cost of losses is 1.0
The resulting file is called myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.reconciled.xml

Push the resulting .xml file to your remote github repository, then view it at:
http://phylariane.univ-lyon1.fr/recphyloxml/recphylovisu
Save the resulting svg file by uploading it to your remote github repository.

## Rooting via Minimization of Duplications and Deletions in Notung 

    java -jar ~/tools/Notung-3.0-beta/Notung-3.0-beta.jar -b rrn3batch.txt --root --speciestag prefix  --savepng --treestats --events  --phylogenomics   
    
The cost of the tree is 10 duplications and 15 losses. The event score is 30.0

# 4. Reconciliation Based Rearrangement and Topology Tests

## Rerooting via Minimizing duplications and losses by rearranging poorly supported branches
A full bootstrap was used and values above 80 are well supported. A threshold of 80 was used to define well support branches. To avoid overwriting previously rerooted trees, the output will be directed to rrn3reconcileRearrange directory. 

    java -jar ~/tools/Notung-3.0-beta/Notung-3.0-beta.jar -b rrn3batch.txt --rearrange --speciestag prefix  --savepng --treestats --events  --outputdir rrn3reconcileRearrange --edgeweights name --threshold 80 
    
The tree has 8 duplications and 0 losses. The event score is 12.0
This reroot result by rearrangement is the more parsimonous than midpoint or minimization of duplications and deletions. 

## Visualize the re-arranged tree

    python ~/tools/recPhyloXML/python/NOTUNGtoRecPhyloXML.py -g rrn3reconcileRearrange/myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.rearrange.0 --include.species
    
View this at: http://phylariane.univ-lyon1.fr/recphyloxml/recphylovisu

## Confidence set testing
We have two topologies to consider:

- The first is the optimal tree from IQ-TREE named "myhomologs.renamed.aligned.r50.fullboot.treefile"
- The second is the tree Notung rearranged by assessing poorly supported branches. The tree currently resides in the directory "rrn3reconcileRearrange" and is called "myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.rearrange.0"

Change the format of the rearranged tree from Notung to Newick

    java -jar ~/tools/Notung-3.0-beta/Notung-3.0-beta.jar -g rrn3reconcileRearrange/myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.rearrange.0   -s species.tre --reconcile --speciestag prefix  --treeoutput newick --nolosses

The output tree is called myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.rearrange.0.reconciled

Unroot the re-arranged tree

    gotree unroot -i myhomologs.renamed.aligned.r50.fullboot.midpoint.treefile.rearrange.0.reconciled -o myhomologs.renamed.aligned.r50.fullboot.unrooted.treefile.rearrange
    
Put the two trees of interest into a single file

    cat myhomologs.renamed.aligned.r50.fullboot.treefile myhomologs.renamed.aligned.r50.fullboot.unrooted.treefile.rearrange > myhomologs.r50.alternativetrees

Run the topology test in IQ-TREE
If you look at myhomologs.renamed.aligned.r50.fa.iqtree, the best fit model according to BIC is "VT+G4". This model will be specified with option -m
The -z option specifies the file with our trees of interest. The -au specifies that we want to run the "Approximately Unbiased" topology test. The -zb command specifies the number of bootstrap replicates that will be run when conducting the AU test. --prefix specifies that all the output files will have a particular prefix. The -te option defines the optimal tree, which is "myhomologs.renamed.aligned.r50.fullboot.treefile"

    iqtree -s myhomologs.renamed.aligned.r50.fa -z myhomologs.r50.alternativetrees -au -zb 10000 --prefix RRN3altTrees -m VT+G4 -nt 2 -te myhomologs.renamed.aligned.r50.fullboot.treefile
    
The loglikehood scores are:

Tree 1 / LogL: -11023.475

Tree 2 / LogL: -11033.911

The AU topology scores are:

Tree 1 / AU: 0.914

Tree 2 / AU: 0.086

Since the p value for tree 2 is greater than 0.05, the arrangement changes suggested by the reconciliation are not inconsistent the phylogenetic signal. The rearranged tree is supported by both reconciliation and by the phylogenetic signal. 

# 5. Domain Prediction using Interproscan web serive

## Install utilities
If this project is being reproduced from scratch, a perl module and DataMash utility must be installed. If commands are copied and pasted, be sure to do so one by one, after each command has finished running.

    sudo cpan LWP::Protocol::https
    
    cd ~/tools
    wget http://ftp.gnu.org/gnu/datamash/datamash-1.3.tar.gz
    tar -xzf datamash-1.3.tar.gz  
    cd datamash-1.3
    ./configure
    make
    make check
    sudo make install 

## Run interproscan 5

The interproscan web service will be used to scan proteins in the query gene family for pre-defined domains. Several different domain databases will be used, but results from pfam will be focused on.

The renamed, with-gaps, unaligned, blast sequences are needed: XP_032221022.1.blastp.detail.filtered.renamed.fas

    iprscan5   --email harry.huang@stonybrook.edu  --multifasta --useSeqId --sequence   XP_032221022.1.blastp.detail.filtered.renamed.fas

## Pfam filtering

Concatenate all the domains identified for each gene into a single file. 

    cat *.tsv.txt > yourgenefamily.domains.all.tsv
    
Filter for domains defined by Pfam database

    grep Pfam yourgenefamily.domains.all.tsv >  yourgenefamily.domains.pfam.tsv

## Visualize predicted Pfam domains on the phylogeny

     awk 'BEGIN{FS="\t"} {print $1"\t"$3"\t"$7"@"$8"@"$5}' yourgenefamily.domains.pfam.tsv | datamash -sW --group=1,2 collapse 3 | sed 's/,/\t/g' | sed 's/@/,/g' > yourgenefamily.domains.pfam.evol.tsv
     
Transfer the rearranged annotation file yourgenefamily.domains.pfam.evol.tsv to your local computer by pushing it to your GitHub repository and downloading it.

Since the rearranged tree from the topology test has both reconciliation support and is not inconsistent with the phylogenetic signal, it will be used going forward

## Load the tree in Evolview

Now, go to https://www.evolgenius.info/evolview/ Either sign up for an account (suggested), or click "Try Me" Read more about using Evolview here: https://www.evolgenius.info/evolview/helpsite/qst1.html

Upload myhomologs.renamed.aligned.r50.fullboot.unrooted.treefile.rearrange by clicking on the File folder in the upper left hand corner.

## Load the annotation file into Evolview

Upload the annotation file yourgenefamily.domains.pfam.evol.tsv by clicking on the upload tree datasets icon, the up arrow on the left hand side of the screen. 

- Enter a name for the data: pfam
- Hover over the Upload Types until the selected type is upload data for protein domains
- Select the file yourgenefamily.domains.pfam.evol.tsv
- After your hit submit, adjust the tree display using the <- -> icon at the top of the screen (just to the right of the leaf). - You should see your domains annotated by the tips of your tree. Note: you should also see a legend with the names of all the PFAM domains. You may need to refresh the page to see this.

Download the files as a png or pdf and upload to github repository
Uploaded as: rrn3pfam.png
