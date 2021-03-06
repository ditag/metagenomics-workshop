========================================
Mapping reads and quantifying genes
========================================

Overview
=========
So far we have only got the number of genes and annotations in the sample.
Because these annotations are predicted from assembled reads we have lost the quantitatve
information for the annotations. So to actually **quantify** the genes, we will map the input
reads back to the assembly.

There are many different mappers available to map your reads back to the
assemblies. Usually they result in a `SAM or BAM file <http://genome.sph.umich.edu/wiki/SAM>`_.
Those are formats that contain the alignment information, where BAM is the binary version of the plain text SAM
format. In this tutorial we will be using `bowtie2 <http://bowtie-bio.sourceforge.net/bowtie2/index.shtml>`_.
You can also take a look at the `Bowtie2 documentation <http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml>`_.

The SAM/BAM file can afterwards be processed with `Picard <http://broadinstitute.github.io/picard/>`_
to remove duplicate reads. Those are likely to
be reads that come from a `PCR duplicate <http://www.biostars.org/p/15818/>`_.

`BEDTools <http://code.google.com/p/bedtools/>`_ can then be used to retrieve
coverage statistics.


Mapping reads with bowtie2
=========
First set up the files needed for mapping. **Replace 'N' with the kmer you used for the velet assembly**::

    mkdir -p ~/mg-workshop/results/annotation/mapping/$SAMPLE/
    cd ~/mg-workshop/results/annotation/mapping/$SAMPLE/
    ln -s ~/mg-workshop/data/$SAMPLE/reads/1M/${SAMPLE_ID}_1M.1.fastq pair1.fastq
    ln -s ~/mg-workshop/data/$SAMPLE/reads/1M/${SAMPLE_ID}_1M.2.fastq pair2.fastq
    ln -s ~/mg-workshop/results/assembly/$SAMPLE/${SAMPLE}_${kmer}/contigs.fa

Then run the ``bowtie2-build`` program on your assembly::

    bowtie2-build contigs.fa contigs.fa

**Question: What does bowtie2-build do? (Refer to the documentation)**

Next we run the actual mapping using ``bowtie2``::

    bowtie2 -p 8 -x contigs.fa -1 pair1.fastq -2 pair2.fastq -S $SAMPLE.map.sam

The output SAM file needs to be converted to BAM format and be sorted, either by read name or by leftmost alignment coordinate. We'll sort by coordinate which is the default. 
For this we will use `samtools <http://samtools.sourceforge.net/>`_.::
    
    samtools sort -o $SAMPLE.map.sorted.bam -O bam $SAMPLE.map.sam

Removing duplicates
=========
We will now use **MarkDuplicates** from the Picard tool kit to identify and remove
duplicates in the sorted and indexed BAM file::

    java -Xms2g -Xmx32g -jar $PICARD_HOME/MarkDuplicates.jar INPUT=$SAMPLE.map.sorted.bam OUTPUT=$SAMPLE.map.markdup.bam \
    METRICS_FILE=$SAMPLE.map.markdup.metrics AS=TRUE VALIDATION_STRINGENCY=LENIENT \
    MAX_FILE_HANDLES_FOR_READ_ENDS_MAP=1000 REMOVE_DUPLICATES=TRUE

Picard's documentation also exists! Two bioinformatics programs in a row with
decent documentation! Take a moment to celebrate, then take a look `at it
<http://sourceforge.net/apps/mediawiki/picard/index.php>`_.

**Question: Why not just remove all identical pairs instead of mapping them
and then removing them?**

**Question: What is the difference between samtools rmdup and Picard MarkDuplicates?**

Counting mapped reads per gene
==========================
We have now mapped reads back to the assembly and have information on how much of the assembly that is covered by the reads in the sample.
We are interested in the coverage of each of the genes annotated in the previous steps by the PROKKA pipeline.
To extract this information from the BAM file we first need to define the regions to calculate coverage for.
This we will do by creating a custom GFF file (actually a GTF file) defining the regions of interest (the PROKKA genes).
Here we use an in-house bash script called prokkagff2gtf.sh_ that searches for the gene regions in the PROKKA output
and then prints them in a suitable format::

    prokkagff2gtf.sh ~/mg-workshop/results/annotation/functional_annotation/prokka/$SAMPLE/PROKKA_${date}.gff > $SAMPLE.map.gtf

We then use htseq_ to count the number of reads mapped to each gene. Here we have to tell htseq that the file is sorted by alignment coordinate `-r pos`::

    htseq-count -r pos -t CDS -f bam $SAMPLE.map.markdup.bam $SAMPLE.map.gtf > $SAMPLE.count

The output file has two columns, the first contains the gene names and the second the number of reads mapped to each gene. 
The last 5 lines gives you some summary information from htseq.

Normalizing to Transcripts Per Million (TPM)
==========================
So now we have abundance values for genes in the assembly in the form of absolute read counts mapped to each gene. But we have not taken into account that longer genes will get more mapped reads than shorter genes just by being longer. Also, if we'd like to compare abundance values between several samples we would have to account for the fact that the total number of reads sequenced (the sequencing depth) may differ significantly between samples.

There are several ways to normalize abundance values in metagenomes. Here we will use the TPM (Transcripts Per Million) method. For information on TPM and how it relates to other ways to normalize, like RPKM, see this_ blog post.

In order to calculate TPM values we need to know:

    - The average read length of the sample
    - The length of all genes

The average read length can be calculated from the fastq sequence file that you started with, but we'll save you the trouble and say it's ~100 bp. 

The gene lengths we can get from the GTF file that you used with htseq::

    cut -f4,5,9 $SAMPLE.map.gtf | sed 's/gene_id //g' | gawk '{print $3,$2-$1+1}' | tr ' ' '\t' > $SAMPLE.genelengths

Here we extract only the start, stop and gene name fields from the file, then remove the 'gene_id' string, print the gene name first followed by the length of the gene, change the separator to tab and store the results in the .genelengths file.

Now we can calculate TPM values using the tpm_table.py_ script::

    tpm_table.py -n $SAMPLE -c $SAMPLE.count -i <(echo -e "$SAMPLE\t100") -l $SAMPLE.genelengths > $SAMPLE.tpm

We now have coverage values for all genes predicted and annotated by the PROKKA pipeline. Next, we will use the annotations and coverage values to summarize annotations for the sample and produce interactive plots.

**Question: Coverage can also be calculated for each contig. Do you expect the coverage to differ for a contig and for the genes encoded on the contig? When might it be a good idea to calculate the latter?**

.. _get_coverage_for_genes.py: https://github.com/EnvGen/metagenomics-workshop/blob/master/in-house/get_coverage_for_genes.py
.. _prokkagff2bed.sh: https://github.com/EnvGen/metagenomics-workshop/blob/master/in-house/prokkagff2bed.sh
.. _prokkagff2gtf.sh: https://github.com/EnvGen/metagenomics-workshop/blob/master/in-house/prokkagff2gtf.sh
.. _this: http://www.rna-seqblog.com/rpkm-fpkm-and-tpm-clearly-explained/
.. _tpm_table.py: https://github.com/EnvGen/metagenomics-workshop/blob/master/in-house/tpm_table.py
.. _htseq: http://www-huber.embl.de/HTSeq/doc/overview.html
