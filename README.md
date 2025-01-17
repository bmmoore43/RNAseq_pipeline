# RNAseq pipeline

Workflow: Bowtie -> Tophat (maps reads) -> get sam file via samtools -> 
HTseq count [to get counts of reads to each gene or exon] -> Edge R -> differential expression  

# Needed files

1) Genome sequence in FASTA format

2) SRA file containing sequence reads (or folder with all SRA files and a file with a list of all SRAs you want to run)

3) GFF file for genome sequence

# Create Bowtie Index for Genome sequence: estimated time: ~15 minutes

1) load Tophat2 module

    module load Tophat2
    
2) Use bowtie2-build to build the index

    bowtie2-build [genome sequence fasta file] [base name for output files]
    
    bowtie2-build  Slycopersicum_390_v2.5.fa  Slycopersicum_390_v2.5
    
# Map RNA seq reads- Single file

1) modules required:

    module load SRAToolkit; module load FastQC; module load Trimmomatic; module load TopHat2; module load Boost; module load SAMtools
    
2) Single mapping command: *you must indicate whether it is SE or PE, and adapter file must be in your working directory

    python /mnt/home/john3784/Github/RNAseq_pipeline/ProcessSRA_hpcc2.py [SRA file] [root of Bowtie index files] [index 0 for SE, index 1 for PE]
    
 This command incorporates individual commands all in one (no need to run individually):
        
    module load SRAToolkit -- converts SRA file to fastq
    
    fastq-dump SRRfile.sra 
    
    module load FastQC -- runs fastqc on fastq file
    
    cd [dir with SRA files]
    
    fastqc -f fastq SRRfile.fastq 
    
    module load Trimmomatic -- trims adaptor sequences, this is where you specify SE or PE adaptors, 
    or other sequences you may want to trim
    
    java -jar $TRIM/trimmomatic SE SRR314813.fastq SRR314813.fastq.TRIM ILLUMINACLIP:$ADAPTOR:2:30:10 LEADING:3 TRAILING:3      SLIDINGWINDOW:4:30 
    
    filter_fastq.py -- filters reads by length and min average phred score
    
    module load FastQC -- runs fastqc on trimmmed and filtered files
    
    cd [dir with SRA files]
    
    fastqc -f fastq SRRfile.fastq2
    
    module load TopHat2 -- maps trimmed reads to genome
    
    tophat2 -p %s -i %s -I %s -g %s -o %s %s %s" %(tophat_threads,min_intron_size,max_intron_size,max_multiHits,f_tophat_file,genome,filtered_file)
    
    module load SAM tools
    
    converts bam to sam
    
    Filter sam to primary and unique reads
    
    output:
    
3) mapping may take a while, so consider submitting to the hpcc queue

    python ~john3784/Github/parse_scripts/qsub_slurm.py -f submit -c SE_SRA_files.txt.runcc -wd /mnt/scratch/john3784/RNA_seq/SE_SRA_files/ -m 10 -w 239 -mop yes -mo GCC/5.4.0-2.26,OpenMPI/1.10.3,Trimmomatic,TopHat/2.1.1,Bowtie2/2.3.2,SAMtools/1.5,python

# Map RNA seq reads multiple files

1) this script makes a runcc file to map each sra file in a list: *you must make separate lists for SE or PE. If using fastq files, designate -fastq_file T

    ProcessSRA_hpcc-batch_runcc.py [SRA file list] [full path to bowtie index] [0 for SE, 1 for PE]

    python ~john3784/Github/RNAseq_pipeline/ProcessSRA_hpcc-batch_runcc.py SE_SRA_files.txt /mnt/scratch/john3784/RNA_seq/SLyc2.50 0
    
2) qsub runcc file

    python ~john3784/Github/parse_scripts/qsub_slurm.py -f submit -c SE_SRA_files.txt.runcc -wd /mnt/scratch/john3784/RNA_seq/SE_SRA_files/ -m 10 -w 239 -mop yes -mo GCC/5.4.0-2.26,OpenMPI/1.10.3,Trimmomatic,TopHat/2.1.1,Bowtie2/2.3.2,SAMtools/1.5,python

    
# Check for % reads mapped

1) This script checks for mapping that is less than 80% and gives a file with a list of sra files that did not meet the 80% cutoff. You can look at the fastQC of these files and try rerunning them. May need to remove bases at the beginning of the reads (-trim_hcrop <number of bases to remove> option) or add overrepresented sequences to the adapter sequence (-trim_adapter <file with adapters and over-represented sequences> option)

        python get_bad_mapping_files <dir with _tophat directories>
        
        python ~john3784/Github/RNAseq_pipeline/get_bad_mapping_files.py /mnt/scratch/john3784/RNA_seq/SE_SRA_files/
  

# Get Ht-seq and cufflinks output

1) python 4_Runcc_cufflinks_after_tophat.py [folder including tophat directories] [gff file] [genome.fa file] [SE (0) or PE (1)]
   
   **note PE option requires module load samtools
   
   **PE may take a while depending on how many files you have, so consider submitting to the queue with qsub
   
   output is a cufflinks/htseq runcc file
   
2) submit cufflinks/htseq runcc file to the queue with qsub

# Get differentially expressed genes

This section will allow you to identify differentially regulated genes through edgeR

1. go to R_studio

2. check “edgeR.pdf” for info

3. example R script


   here I have 2 replicates for MOCK and 2 replicates for COR and combine the HTseq output files among these samples together.
   
        > counts = read.delim("HTSeqCount_combined.txt", header=F, row.names=1, sep=' ')
        > group <- factor(c("mock","mock","cor","cor"))
        > d <- DGEList(counts=counts,group=group)
        > d <- calcNormFactors(d)
        > d$sample
        > d <- estimateCommonDisp(d)
        > d <- estimateTagwiseDisp(d)
        > de.com <- exactTest(d, pair=c("mock", "cor"))
        > results <- topTags(de.com,n = length(d[,1]))
        > write.table(as.matrix(results$table),file="outputFile.txt",sep="\t")
        > pdf("edgeR-MA-plot.pdf")
        > plot(results$table$logCPM,results$table$logFC,xlim=c(-3, 20), ylim=c(-12, 12), pch=20, cex=.3, col = ifelse( results$table$FDR < .1, "red", "black" ) )
        > dev.off()
        # making larger design table
        > group <- factor(c(1,1,2,2,3,3))
        > design <- model.matrix(~group)
        > fit <- glmFit(y,design)
        # To compare 2 vs 1:
        > lrt.2vs1 <- glmLRT(fit,coef=2)
        > topTags(lrt.2vs1)
        # To compare 3 vs 1:
        > lrt.3vs1 <- glmLRT(fit,coef=3)
        # To compare 3 vs 2:
        > lrt.3vs2 <- glmLRT(fit,contrast=c(0,-1,1))
        # genes in all 3
        > lrt <- glmLRT(fit,coef=2:3)
        > topTags(lrt)


   
