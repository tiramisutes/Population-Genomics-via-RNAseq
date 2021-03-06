# Transcriptome assembly in the Palumbi Lab*

## GENERAL INFORMATION
* Scripts are formatted for Stanford internal use. We use SLURM to send jobs to Stanford's Sherlock cluster.

### How to create a Sherlock account 
- to get access and support, e-mail research-computing-support[at]stanford.edu
- download [Kerberos](https://uit.stanford.edu/service/kerberos)
- [Sherlock Wiki] (http://sherlock.stanford.edu/mediawiki/index.php/Main_Page)
- To configure your SSH client to pass those Kerberos credentials for MAC, open terminal (Click Spotlight, type in terminal) and run the following commands:
```
	mkdir -p ~/.ssh 
	echo "Host sherlock sherlock.stanford.edu sherlock* sherlock*.stanford.edu   
	GSSAPIDelegateCredentials yes   
	GSSAPIAuthentication yes" >> ~/.ssh/config 
```

- create project directory in `$PI_SCRATCH/` (1TB space available)
	- more space and accessible to other users in the lab 
- your personal directory `$HOME` is 15GB
- [more info on Sherlock storage](http://sherlock.stanford.edu/mediawiki/index.php/DataStorage)

### Downloading & storing data from Gnomex
`wget http://monalisa.cern.ch/FDT/` in the directory where you want your raq sequence files

- use command line from Gnomex to download files to Sherlock
- backup raw files on external harddrive in lab

### Sherlock basics 

- to log in:
	```
	kinit user@stanford.edu & type in pw to gain permission
	ssh user@sherlock.stanford.edu to access cluster
	```

- to move around:
`cd $HOME` is your personal directory
	 - limited to 15GB storage
	 - not accessible to other users
`cd $PI_HOME/scripts`
- all common use scripts
`cd $PI_HOME/programs`
- all common use programs

- to add a program to Sherlock
	```
	cd $PI_HOME/programs
	wget <linktofiledownload>
	unzip <file> 
	#or gzip <file.gz>
	```

- modifying your path to include scripts & programs:
	- this allows you to access these directories from any directory without having to hardcode the path	 
	 ```
	 cd ~
	 ls -a #list hidden items
	 nano .bashrc
	 #paste the following into your .bashrc
	 export PATH="$PI_HOME/programs:$PATH"
	 export PATH="$PI_HOME/scripts:$PATH"
	 export PATH="/share/PI/spalumbi/programs/anaconda/bin:$PATH"
	 ```
- ways to submit jobs:
`login_node` this is the default, it's for small jobs and moving around sherlock
`sdev` is for running small test jobs straight from the command line
	- one hour time limit
	- use `sdev -t 2:00:00` to get max 2 hrs
`bash` is for  use for small jobs in terminal
`sbatch` submits the job generated by your script to the cluster

- to run your job on different nodes, in your batch script, add: 
`#SBATCH -p owners` access to 600 nodes only available to owners. You will be kicked off an owner node if that owner logs on. Always check your slurm file to see if your job was aborted
`#SBATCH -p spalumbi` 256GB memory node
`#SBATCH -p hns` 1TB memory node
`#SBATCH -p spalumbi,hns` #will submit your job to whatever node is available first

- to check on status of job
`squeue -u username`
`squeue | grep 'spalumbi`

- to cancel a job
`scancel <jobID>` to cancel 1 job
`scancel -u <username>` will cancel all jobs

- to see how much memory your job has used so far
	- ` sstat --format JobID,NTasks,nodelist,MaxRSS,MaxVMSize,AveRSS,AveVMSize $JOBNUMBER`

### Best practices:

Back up your scratch directory! 

- You can use the Sherlock data transfer node to move large datasets onto a backup hard drive
```
kinit username@stanford.edu
rsync -avz --progress --stats -e 'ssh -o GSSAPIAuthentication=yes' user@sherlock-dtn.stanford.edu:/<sherlock directory> <backup location>
```

- You can also use your Stanford Google Drive (with unlimited storage) to back up your files
```
ml load gdrive
gdrive -help
gdrive upload --recursive <path>
```
- once you're linked to your google account, there is also an example gdrive-upload.sh script in the shared scripts directory

- Check to see how much space you're taking up in the shared SCRATCH directory
`du -sh * | sort -h`

- `chmod 775 *` programs, scripts, some files that you create so others can use them

- `chmod 444` files you don’t want to accidentally write over, 
	`chmod -R ###` for directories

- if downloading a program for the first time, move the executable of the program to the programs folders
 `mv <program> $PI_HOME/programs`

- if downloading a new version of a program, rename with version to not override any executables in that directory

- try to create scripts that are not hard-coded 

- comment your scripts!

- Tips for checking outputs along the way:

	- always look at your slurm files for errors
		- `cat slurm*`
	- when making scripts that split up data into TEMP files for parallel processing, add a line that states your input file in the slurm output
		- `echo $1` if $1 is your input file
	- to count lines in a file
		- `wc - l <filename>`
	- to count contigs in a file
		- `grep -c “>” <filename>`



## DE NOVO TRANSCRIPTOME ASSEMBLY

### 1) Trim & Clip (Trimmomatic) 
- this program removes adapters, low quality seqs, etc.
- [webpage](http://www.usadellab.org/cms/?page=trimmomatic)
-  for paired end reads:
	- `bash batch-trimmomatic-pe.sh *_1.txt`
	- outputs 4 files: 1\_paired.fq, 2\_paired.fq, 1\_unpaired.fq, 2\_unpaired.fq
-  for single end reads:
	`batch-trimmomatic-se.sh`


### 2) Quality check (FastQC) 
- this step is useful for picking samples to use in your Trinity assembly
- this step is also helpful for PE data, where you use the length of your sequences for merging reads (i.e. FLASH)
-  `batch-fastqc.sh`
- to move a file from the cluster to your computer
- open a terminal that is not logged into cluster
`rsync user@sherlock.stanford.edu:/share/PI/spalumbi/... /Users/username/Desktop/...`
`rsync -a` will recursively transfer an entire directory

### 3) Merge paired end reads (FLASH) 
`batch-flash.sh`

- This step is necessary for PE data to merge your two reads (\_1 and \_2) that you will use in transcriptome assembly, but is not necessary for SE data 
- [webpage](https://ccb.jhu.edu/software/FLASH/)
- get lengths of your trimmed seqs above and use in FLASH script


### 4) Pick what samples to assemble into a transcriptome  
- pick samples relevant to your biological question
- restrict sample # for computational capacity


### 5a) De novo assembly (Trinity) 
`example_Trinity_mkm.sbatch`

- [Trinity wiki] (https://github.com/trinityrnaseq/trinityrnaseq/wiki)
- to see insides of Trinity: 
	-  	`.Trinity --show_full_usage_info` 
	-  you need to update this example for your assembly 
- 	[Trinity computing requirements](https://github.com/trinityrnaseq/trinityrnaseq/wiki/Trinity-Computing-Requirements)
	-  1GB RAM for 1M reads
	- 	Example: 3 Balanus samples: 11649437 + 11728787 + 11556893
	-  total pairs of reads = 34,835,117 is 35GB RAM and 35 hours of time

### 5b)Check out the quality of your assembly

- to get median contig length:
  ```
  $ samtools faidx Bg_2assembliesof3.fa
  $ module load R
  $ R
  $ index<-read.delim(‘<file.fa.fai>’, header=F)
  $ median(index[,2])
  $ table(cut(index[,2],30)) #bins into groups of 30
  $ table(index[,2]>300) #how many contigs are greater than 30bp
  ```

- another option for number of contigs, contig n50, etc.
`perl abyss-fac.pl <assembly.fa>`

### 6a)Take the longest Isoform from each contig 
`perl longestisoform_trinity.pl <input.fasta> <output.fasta>`

### 6b)If meta-assembling Trinity assemblies 
- rename contigs in one file by adding “a” to end of file name, for ex: "TRINITYa_blahblah"
	- otherwise CAP3 may get confused if names are repeated
	
`cat Trinityrun1.fa Trinityrun2.fa > all_assemblies.fasta`

### 7)Meta-Assembly (CAP3) 
- [paper](http://genome.cshlp.org/content/9/9/868.full)
- [manual](http://computing.bio.cam.ac.uk/local/doc/cap3.txt)
- CAP3 is an overlap consensus assembler that will merge reads that would not assemble in Trinity due to high heterozygosity

`sbatch cap3.sh`

- merge your contigs and singlets files into one assembly file
`cat file.fasta.cap.contigs file.fasta.cap.singlets > newfile.fasta`
	- contigs are what CAP3 merged, singlets did not merge and still contain the Trinity output name
	
- to look at contig #s after CAP3:  
`grep -c '>' file.fasta`

- to check for contig name duplicates: 
`grep ">" <assembly_file.fa> | perl histogram.pl | head -n`

### 8)Annotate (BLAST) 
-  [Blastx program download](http://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastDocs&DOC_TYPE=Download)
-  [blastx commands, table C4](http://www.ncbi.nlm.nih.gov/books/NBK279675/)

### 8a)Downloading a genome of interest to blast against
- if you don't have this resource; move to 8b
- download cDNA files for blasting your nucleotides against real transcripts
- use the makeblastdb tool from ncbi to make a database:
	`makeblastdb -in <infile.fasta> -dbtype prot -out <outfile>`


###Filter Assembly without available genome

### 8b)Annotate with Uniprot database
- 	Uniprot is a more curated database and is recommended over NCBI-nr

#### How to download & create the Uniprot/Trembl database for the first time:
- [download Swiss-Prot & Trembl databases](http://www.uniprot.org/downloads)

	```
	wget <link>
	gunzip <file>.gz
	
	# merge two database files into one database
	cat uniprot_sprot.fasta uniprot_trembl.fasta > unitprot_db.fasta
	
	# make your new fasta into a database
        makeblastdb -in uniprot_db.fasta -dbtype prot -out uniprot_db
   	```

### Run uniprot blast
`bash batch-blast-uniprot.sh infile`
	- the script above splits your assembly into smaller files and calls the blastx on your uniprot database
	
- after running, check if you get an error during your blasts
	- `cat slurm*` 
	- most often, your blast may time out
- if you get an error:
 	- `grep -B 1 error slurm*.out`
  	- for any file with error, take line before (the tempfile)
	- `cat <TEMPwerror.fa> <TEMPwerror.fa> > <didnotfinish.fa>`
		- this concatenates all TEMP files that contain an error into a new file
	- you may want to reduce the # of contigs the batch-blast-uniprot.sh script generates for each TEMP file
	- `bash batch-blast-uniprot.sh <didnotfinish.fa>` 

### 8c)Annotate with NCBI-protein database
#### How to download & create the ncbi-protein database for the first time
- Make sure local database on sherlock is up to date
- to open .tar files downloaded from genbank: 
	- `for i in *.tar ; do tar -xvf $i ; done &`
	- ncbi files are already databases, no need to use makedb script

- To run blast script:
	- `bash batch-blast-ncbiprot.sh input.fasta`
	- this script splits your assembly into TEMP files for parallel processing
- after running, check if you get an error during your blasts
	- `cat slurm*` 
	- your blast search may time out if it runs too long on the cluster
- if you get an error:
 	- `grep -B 1 error slurm*.out`
  	- for any file with error, take line before (the tempfile)
	- `cat <TEMPwerror.fa> <TEMPwerror.fa> > <didnotfinish.fa>`
		- this concatenates all TEMP files that contain an error into a new file
	- you may want to reduce the # of contigs the batch-blast-ncbiprot.sh script generates for each TEMP file
	- `bash batch-blast-ncbiprot.sh <didnotfinish.fa>`


### 9)Parse XML blast files
- Both the uniprot and nr batch scripts above output XML formatted results to get all data fields available
- XML files are difficult to work with, pick what information you want and parse to a tab delimited format
- XML files currently hold the most information about each blast result
	- i.e. gene name, taxonomy 
- to parse uniprot results:
`bash batch-parse-uniprot.sh`
	- this script calls parse-uniprot-xml.py on your many temp files
- to parse ncbi-protein results:
 `bash batch-parse-nr.sh TEMP*.fa.blast.out`

### 10) Use blast results to remove contamination
- If you BLAST to a general database and not to a specific genome, this step is necessary to filter out any contamination
- remove blasts that are likely environmental contamination, i.e. bacteria, fungi, viruses, alveolata, viridiplantae, haptophyceae, etc.

- For uniprot results:
	- first,  merge all of your parsed blast results into one file
		`cat *_parsed.txt > all_parsed.txt`
	-  second, create a file of only 'good contigs', i.e. metazoans:
		`bash grep-good-contigs.sh all_parsed.txt assembly.fa`
	- third, if you only want to use uniprot results, pull only good contigs from your assembly fasta file
	`sbatch batch-filter-assembly.sh assembly.fa goodcontigs.txt`
	- to check how filtering went, count how many contigs you have before and after filtering:
		`grep -c “>” filteredassembly.fa`

- For ncbi-protein results:
	- first,  merge all of your parsed blast results into one file
		`cat *_parsed.txt > all_parsed.txt`
	- second, create a taxonomic tree to use for filtering
		- make a tab delimited file with contigs names and their associated GI numbers
			- i.e., import parsed file into excel and keep only those to columns, export as txt file
		- use this GI list to get taxonomic tree 
		`sbatch batch-taxonID-from-gi.sh type('n' or 'p') GIlist output`
	- third, grep good contigs from uniprot
		`bash grep-good-contigs-ncbiprot.sh uniprot_goodcontigs.txt`

- combine results from uniprot & ncbi-protein databases:
	`cat uniprot_goodcontigs ncbi_goodcongits > combined_goodcontigs_duplicates.txt`
	-then take only the uniq contigs from the combined file
	`cat combined_goodcontigs_duplicates.txt | sort | uniq > combined_goodcontigs_uniq.txt`
	- you can check the # contigs remaining after this to see how filtering went 
	- finally, pull only good contigs from your assembly
	`sbatch batch-filter-assembly.sh assembly.fa combined_goodcontigs_uniq.txt`
		
## TRANSCRIPTOME ANALYSIS

### Map reads to assembly with Hisat2 (current)

- [Hisat2](https://ccb.jhu.edu/software/hisat2/index.shtml)

```
#create an hisat index
hisat-2 build infile.fasta basename
#call the program through a batch script
bash batch-hisat2-fq-paired.sh hisat2-index chunksize *_1.txt.gz

```
- for single end reads, batch-hisat2-fq-single.sh

### Map reads to assembly with Bowtie2 (old version)
- [Bowtie2 download](http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml#the-bowtie2-build-indexer)

- first, make a bowtie index from your final assembly, this should output 6 files
   `bowtie2-build <assembly.fa> b2index` 
- next, call the script on your raw (i.e. untrimmed/clipped files)
   `bash batch-bowtie2-fq-paired.sh b2index 1 *_1.txt.gz`  
- check TEMPBATCH.sbatch after submitting to see if it started correctly 
   `cat TEMPBATCH.sbatch` 
- after it completes, check for errors by printing your slurm files to the screen
   `cat slurm*`



## SNP CALLING & FILTERING

### SNP Calling (Freebayes) 
- First, make a contig list (a .bed file) that is required for Freebayes
`bash fasta2bed.sh assembly.fa outfile`

- Next, make a new directory to hold your VCF output files
`mkdir vcfout`

- Then, call the script
`sbatch freebayes-cluster.sh assembly.fa vcfout contiglist.bed ncpu *bam`
- this cluster script generates many files to parallelize the process and then calls the freebayes-sequential-intervals.sbatch on each temp file
- ncpu is the number of cpus you want to use, you can try 16 for example

### SNP Calling version 2 (BCFtools)
- in progress...

### Filter SNPs (vcflib)
- [vcflib website](https://github.com/vcflib/vcflib#vcflib)
- [vcflib scripts](https://github.com/vcflib/vcflib/tree/master/scripts)
- can filter for: read depth, read mapping quality, base quality, minor allele frequency, min number of reads mapped, etc.

- First, combine all of your VCF outputs from the parallel processing
`fastVCFcombine.sh combined.vcf *.vcf`

- Next, use VCF tools to filter your SNPS:
`sbatch vcftools-snpfilter.sh combined.vcf`
- this script removes indels, takes only biallelic SNPS, requires 7 mapped reads per allele at each SNP, and sets your min/max allele frequencies
- you can also allow some missing data with this script using --max-missing (1 means no missing data allowed, 0 means allow all missing data)


### Create 0,1,2 genotype SNP matrix (vcftools)
`bash vcftools-012genotype-matrix.sh <combined_filtered_file.vcf> <outfile>`
- this format is used by many downsteams applications, like R

### Protein changes
```
	#get open reading frame predictions for your assembly
	#usage without blastx info:
	sbatch get-orf-predictors.sh assembly.fa empytyfile outfile
	
	#get ORF headers
	grep ">" orf.pep | sed 's/>//g' | sed 's/+//g' > orfpredictor_headers.txt
	
	#get snp protein changes
	snp_protein_changes.py vcf ref.fa orfpredictor_headers.txt OUT
	
```
- this will output a file with 
	- the contig and snp position
	- the reference allele and alternate allele
	- the reference and alternate protein
	- if the protein change was synonymous (S), nonsynonymous (NS), or in a UTR region
	- the position of the change in the codon (CPOS)

### How to subsample a vcf with a list of SNPs names
```
#create a list (goodsnps.txt) of SNP names, for example 'contig/tpos'
awk -v OFS="\t" '{print $1,$2-1,$2}' goodsnps.txt > goodsnps.bed
bedtools intersect -b goodsnps.bed -a allsnps.vcf > goodsnps.vcf
```

###Detect SNP outliers (Bayescan)
- [download program](http://cmpg.unibe.ch/software/BayeScan/download.html)
- make a .bsc file, see R script `Bayescan_generate_input_file.R`
- rsync .bsc file to sherlock
- on Sherlock cluster, run Bayescan 
`sbatch bayescan.sh input.bsc`
- can take SNPs from three separate Bayescan runs
- rsync FST file back to your computer and analyze with R scripts: Bayescan_plot_R.r and Bayescan_plot_results_sample.R
	

## GENE EXPRESSION COUNTS
`bash get-bam-counts.sh *.bam`

