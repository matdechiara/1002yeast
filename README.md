# First Pipeline: NON-REFERENCE MATERIAL
### This pipeline extracts all the non-reference material from a large set of assemblies ##
——

_Dependencies:_ 
 * _perl_
 * _bioperl_
 * _blastn_
 * _bedtools_ 
 * _emboss (infoseq)_

——
## IMPORTANT: Each contig must have unique names in every assembly, failure to follow this will result in huge data loss! 
——

### Step 1 : 
The path of the blast and of the bed tools has to be specified inside the files "refine_disp.pl" and "refine_disp_split.pl".
Creates a blast database of the reference using the makeblastdb command:
```
$ makeblastdb -in REF.fasta -dbtype ‘nucl’ -out sgd
```
The blast database name must be “sgd”, otherwise, line 15 of the "refine_disp.pl" file has to be changed accordingly. 

——

### Step 2 : 
For each assembly, the following commands are needed:
```
$ ./refine_disp.pl assembly_name.fa assembly_name.m8 
$ perl countNs_foseq2.pl assembly_name.fa.kept.fasta 0.2 2>> err.log
```
where "assembly_name.fa" is the assembly fasta file while "assembly_name.m8" will be a tabulated blast output.


The file "assembly_name.fa.kept.fasta" is one output of the "refine_disp.pl" script. The 0.2 indicates the frequency of “N”s allowed in a given sequence to not discard it.


Final file names will end in "afterN.fasta.kept.fasta".
Each of these files will be the fasta files of the non reference sequences for each strain. 

——

### Step 3 :
Requires all the above final output files to be merged into a single file.
```
$ cat *afterN.fasta.kept.fasta >> singlefasta.fa
```
——

### Step 4 : 
The single fasta file, however, needs to have the sequences ordered according to their lengths.
The scripts to do this requires an infoseq output:
```
$ infoseq singlefasta.fa >> singlefasta.fa.infoseq
```
——

### Step 5 : 
Creates a file called "singlefasta.fa.ordered.fasta”:
```
$ perl order_infoseq.pl singlefasta.fa.infoseq >> order.list
$ perl reorder_fasta.pl order.list singlefasta.fa 
```
——

### Step 6 :
A blast alignment of the file from step 5 against itself is needed:
```
$ makeblastdb -in singlefasta.fa.ordered.fasta -dbtype ‘nucl’ -out nnref
$ blastn -query singlefasta.fa.ordered.fasta -db nnref -outfmt '6 std qlen slen' -perc_identity 90 -reward 1 -penalty -5 -gapopen 5 -gapextend 5 -no_greedy > NONREF.m8
```
——

### Step 7 : 
Since very large datasets can be hard to handle, the script “splitter.pl” creates a number of subfiles named 
"out.XXX.m8part"
where XXX is a sequential number.

——

### Step 8 : 
For all files from step 7 generate a file for each run named "out.XXX.m8part.rem.bed”:
```
$ perl refine_disp_split.pl singlefasta.fa.ordered.fasta out.XXX.m8part 
```
These file have to be merged in a single file:
```
$cat *m8part.rem.bed >> Total.rem.bed
```
——

### Step 9 : 
The following sequence of commands will give the final output
(change $bedpath with the path to the bedtools executable)
```
$ perl invert_bed2.pl singlefasta.fa.ordered.fasta Total.rem.bed > Total.temp.keep.bed
$ cat Total.temp.keep.bed | $bedpath/sortBed -i - | $bedpath/mergeBed -i - -d 100 > Total.keep.bed 2>>err.log
$ perl noshorterthen200.pl Total.keep.bed >Total.kept.bed 2>>err.log
$ perl bedRegions2.pl Total.kept.bed singlefasta.fa.ordered.fasta
```

The final file will be named "singlefasta.fa.ordered.fasta.kept.fasta".

-------------------------------------

# Second pipeline: COLLAPSING SIMILAR ORFS
### This pipeline collapses ORFs that are more similar than a specific threshold
——

_Dependencies:_ 
* _perl with "Graph" module installed_
* _bioperl_
* _fasta36_
* _emboss (infoseq)_

——

### Step 1 : 
Aligns the ORFs all against all, using a fasta command to obtain a tabulated output:
```
$ fasta36  -m8  orfs.fasta orfs.fasta > ORF_vs_ORF.m8
```
——

### Step 2 : 
Then the sequence length can be added to the output file using infoseq and the script "length_in_infoseq.pl”:
```
$ infoseq orfs.fasta > orfs.fasta.infoseq
$ perl lenght_in_infoseq.pl orfs.fasta.infoseq ORF_vs_ORF.m8 > ORF_vs_ORF.m8.len.cg.dat
```
——

### Step 3 : 
Add three columns to the ORF_vs_ORF.m8.len.cg.dat file using Excel:
#16 ratio aln len/query len
#17 ratio aln len/hit len
#18 max (16, 17)

——

###  Step 4 : 
Generates the networks of ORFs which will group the similar sequences. The threshold for this step can be adjusted inside the script (lines 33 to 35):
```
$ perl graph_S288C.genes.pl ORF_vs_ORF.m8.len.cg.dat.modified
```

Two files will be generated by this step:
1. "graph.*.m8.id.95.overl.0.overlper.0.75.YRQ”; listing all the ORFs in each Cluster of ORFs.
2. "graph_summary.*.graph.id.90.overl.200.overlper.0.75.YRQ.dat”; containing three columns:
	1. The cluster number
	2. The name of representative ORF for that cluster,
	3. The number of ORFs in the specific cluster.

——

### Step 5 : 
File 2 will be used by another script to write the final fasta file containing only the representative ORFS:
```
$ perl collapsing_from_graph.pl graph_summary.*.graph.id.90.overl.200.overlper.0.75.YRQ.dat orfs.fasta
```

The final file will have ".collapsed.fromgraph.fasta" attached to the end of the name.
