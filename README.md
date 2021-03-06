# BIOINF525 Lab 3.2

* [Introduction](#intro)
* [Setting up your workspace](#setup)
* [Retrieval of sequence data](#retrieval)
* [Basic quality checking with FastQC](#fastqc)
* [Trimming adapter sequence from reads](#trimming)
* [Aligning the trimmed reads to a reference genome](#aligning)
* [Sifting the aligned reads](#sifting)
* [Calling peaks on the aligned reads](#callingpeaks)
* [Creating a browser track so we can look at the peaks in the UCSC Genome Browser](#browsertrack)

## <a name="intro"></a>Introduction

We're going to work through a basic ATAC-seq data analysis
pipeline. We'll check the quality of the assay data, clean it up, map
it to a reference genome, call peaks on the aligned reads, and create
a browser track for the UCSC Genome Browser.

The commands you'll run will be set in code blocks with a gray
background and a `$` shell prompt at the start of each command, like
this:

```bash
$ echo "This is a command."
$ echo "This is another command."
```

To run each command, copy the entire line _after_ the `$` and paste it
into a shell window on the class workstation.

## <a name="setup"></a>Setting up your workspace

Open up a terminal session and create a working directory under your
home directory by copying and pasting the commands after the dollar
signs below:

```bash
$ mkdir -p ~/lab3.2
$ cd ~/lab3.2
```

Then make sure we're running the right versions of the tools:

```bash
$ export PATH=/class/local/bin:$PATH
```

## Retrieval of sequence data

Whether trying to replicate published results or working with your own
data, it's essential to know how high-throughput sequence data was
produced. Paired-end data is processed differently than single-ended,
and many of the tools we use are sensitive to details like read length
and reference genome size.

We're going to work with data from the original ATAC-seq paper by
Buenrostro et al.:

http://www.ncbi.nlm.nih.gov/pmc/articles/PMC3959825/

We don't have time to download and analyze the entire sample during lab, so we're
going to start with a subset in the following FASTQ files:

* SRR891268.1.fq.gz -- contains the first reads of the pairs
* SRR891268.2.fq.gz -- contains the second reads of the pairs

Copy them from the class data directory to your working directory:

```bash
$ cp /class/data/bioinf525/SRR891268*.fq.gz ~/lab3.2
```

If you're curious about how to retrieve published sequence data, read
on, otherwise skip to [Basic quality checking with FastQC](#fastqc).

The authors submitted their data to the NCBI Sequence Read Archive. In
footnotes under the paper's methods section, they provide a link:

http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE47753

Our data comes from the first sample:

http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM1155957

From that page, we know the sample was extracted from 50,000 human
lymphoblastoid cells from the GM12878 line. The library was sequenced
on an Illumina HiSeq 2000, producing paired-end reads.

The sample page also links to the SRA record:

http://www.ncbi.nlm.nih.gov/sra?term=SRX298000

The run ID at the bottom of the page, `SRR891268`, is what you need to
download the data. You also need to have the NCBI SRA tools installed,
so you can use fastq-dump:

```bash
$ fastq-dump --gzip --split-files SRR891268
```

The `--split-files` argument is used to download into two files, one
containing the first read of the pairs, and the other containing the
second reads (mates). Tools used later in the pipeline often use the presence of the
mate file to determine whether they're processing paired end
data.

## <a name="fastqc"></a>Basic quality checking with FastQC

In Lab 1-4 you ran FastQC in Galaxy to check RNA-seq data. Here you'll
run it at the command line, and we'll check the ATAC-seq reads.

```bash
$ fastqc SRR891268.1.fq.gz
```

When it completes, open the HTML report in a web browser with:

```bash
$ xdg-open SRR891268.1_fastqc.html
```

Note the section on adapter contamination. How does this look?

When you're done reviewing, exit the browser. In real work you'd of
course check both files, but in the interest of time we'll move on.

## <a name="trimming"></a>Trimming adapter sequence from reads

For ATAC-seq data, we trim adapter sequence using Jason Buenrostro's
approach: try to align the paired-end reads to each other, and if that
can be done with a Levenshtein edit distance of one or less, chop off
any sequence outside the alignment. The nice thing about this
technique is that you don't need to know which adapter sequences were
used. Other tools generally need to be told, or can sometimes guess,
using a list of known adapters. The command line:

```bash
$ trim_adapters SRR891268.1.fq.gz SRR891268.2.fq.gz
```

When it's done, compare the first few reads in the original and
trimmed files of first reads:

```bash
$ zdiff -u SRR891268.1.fq.gz SRR891268.1.trimmed.fastq.gz | less
```

You should see something like this, where the lines marked with `+`
and `-` in the left margin differ. The `-` lines are the original
versions of the reads or quality lines, and the `+` lines are
trimmed. (Note that there are also comment lines in the FASTQ files
that start with `+` -- ignore those.)

```diff
@@ -15,9 +15,9 @@
 +SRR891268.38259 HWI-ST281:266:C1LTTACXX:1:1101:14488:7554 length=50
 CCCFFFFFGHHHHJJJJJJJJJJJJIJ@GIHIHJJJBEEHBDFFEEDDDB
 @SRR891268.38633 HWI-ST281:266:C1LTTACXX:1:1101:18609:7598 length=50
-TTTCTCGTGTTACATCGCGCCATCATTGGTATATGGCTGTCTCTTATACA
+TTTCTCGTGTTACATCGCGCCATCATTGGTATATG
 +SRR891268.38633 HWI-ST281:266:C1LTTACXX:1:1101:18609:7598 length=50
-CCCFFFFFHHHHHJJJJJJJJJJJJJJJJGHIJJJJJJJJJJJJJJIJJJ
+CCCFFFFFHHHHHJJJJJJJJJJJJJJJJGHIJJJ
 @SRR891268.43221 HWI-ST281:266:C1LTTACXX:1:1101:12315:8330 length=50
 GGGCCGGGCGGTCCCTTTAACGGCGCGGCCCGAGGGGCGCAGGCGGGAGG
 +SRR891268.43221 HWI-ST281:266:C1LTTACXX:1:1101:12315:8330 length=50
```

You can exit from the less program by pressing `q`.

## <a name="aligning"></a>Aligning the trimmed reads to a reference genome

With the adapter cleanup complete, we can finally align the reads to a
reference genome and see where the ATAC-seq transpositions happened.

```bash
$ bwa mem -t 4 /class/data/bioinf525/hg19 SRR891268.1.trimmed.fastq.gz SRR891268.2.trimmed.fastq.gz | samtools sort -@ 4 -O bam -T SRR891268.tmp -o SRR891268.bam -
```

We specify bwa's `mem` algorithm, and give it both files of paired-end
reads. The `mem` algorithm is the latest, and recommended for any
reads longer than 70bp. It also requires just one step, which is why
we're using it with the 50bp reads in this lab, but if you're ever
working with short reads, you'll probably want to at least try the
older "backtrack" algorithm (invoked with `bwa aln` ) for each file of
reads, add the separate `bwa sampe` step to combine the results for
each pair, and compare to the `bwa mem` alignments.

We also pipe bwa's output through `samtools sort` to create the final
BAM file. You'll see a lot of piping in bioinformatics analyses on
Linux. It's generally more efficient, since each command doesn't have
to write its output to disk. Sometimes it is worth preserving the
output of big tasks, though, if you know you'll be feeding it to
multiple downstream processes.

The `-O bam` argument to `samtools sort` requests BAM output, the `-T`
option specifies the basis for the temporary files it creates while
sorting, `-o` names the output file, and `-` specifies that the input
will come from standard input -- the pipe into which bwa sends its
output. You may also see standard input specified as `/dev/stdin`,
usually when a program doesn't recognize `-`; `/dev/stdin` just looks
like a regular file to them.

Finally, note the `-t 4` option: we're telling bwa to use four
processing threads to align the reads faster. We also tell samtools to
use four threads for sorting and compressing with the `-@ 4` option.

On Linux, you can see how many processors are available with the
`lscpu` command. Picking the right number of threads can be
tricky. Too few and your analysis takes longer than it should, but too
many and it could take even longer, as the machine struggles to
balance all the work. You need to know how busy the machine is, and
also how well a program can use multiple processors; some don't scale
well, so there's a point of diminishing returns, after which you're
wasting processors and not getting your results any sooner.

The combination of bwa and samtools is pretty efficient. Running the
above command took about 24 seconds with one thread, and only six
seconds with four. That's on this tiny sample data; with typical
genomic analyses, the difference can be many hours. But running with
eight threads only shaved another second and a half from the run time,
and even with 16 threads, it still took four seconds.

When you have a lot of data to align, it can be more efficient to run
multiple bwa commands concurrently, with a few threads each, than to
run one at a time with a large number of threads.

## <a name="sifting"></a>Sifting the aligned reads

Not all reads map well. We'll use bwa's annotations to sift out the
good ones for subsequent analysis.

Then we'll mark duplicate alignments. Duplication is a complicated
assessment: duplicate reads can come from the same original DNA
fragment, or they can be PCR or optical artifacts of the library prep
or sequencing process. The documentation for the tool we'll use,
[Picard, from the Broad Institute](https://broadinstitute.github.io/picard/command-line-overview.html),
explains how it identifies duplicates.

Here's the command:

```bash
$ java -Xmx8g -jar /class/local/bin/picard/MarkDuplicates.jar I=SRR891268.bam O=SRR891268.md.bam ASSUME_SORTED=true METRICS_FILE=SRR891268.markdup.metrics VALIDATION_STRINGENCY=LENIENT
```

The output will be a BAM file containing all of bwa's output, with
duplicate reads marked. You could also just have Picard remove them, if you
have no need for them later in the pipeline.

Now we need to index the BAM file with duplicates marked:

```bash
$ samtools index SRR891268.md.bam
```

Finally, we'll sift out the good alignments -- reads that mapped
uniquely, with good quality, to autosomal references:


```bash
$ export CHROMOSOMES=$(samtools view -H SRR891268.md.bam | grep '^@SQ' | cut -f 2 | grep -v -e _ -e chrM -e chrX -e chrY -e 'VN:' | sed 's/SN://' | xargs echo); samtools view -b -h -f 3 -F 4 -F 8 -F 256 -F 1024 -F 2048 -q 30 SRR891268.md.bam $CHROMOSOMES > SRR891268.ppmaq.nd.bam
```

Yes, really. You'll see complicated commands strung together like this
all the time. If it helps, this is more complex than average.

The first bit, before the semicolon, creates an environment variable
`CHROMOSOMES` to hold a list of autosomal references obtained from the
header of the BAM file:

> export CHROMOSOMES=$(samtools view -H SRR891268.md.bam | grep '^@SQ' | cut -f 2 | grep -v -e _ -e chrM -e chrX -e chrY -e 'VN:' | sed 's/SN://' | xargs echo);

That environment variable is used in the last argument to the
`samtools view command to only retrieve reads that aligned to those
references:

> samtools view -b -h -f 3 -F 4 -F 8 -F 256 -F 1024 -F 2048 -q 30 SRR891268.md.bam $CHROMOSOMES

As for the rest of the arguments:

- `-b`: requests BAM output
- `-h`: requests that the header from the input BAM file be included

The next few use SAM flags to filter alignments. There's a
[detailed specification](https://samtools.github.io/hts-specs/SAMv1.pdf) for SAM files, which describes the flags that
tools can use to annotate aligned reads.

- `-f 3`: only include alignments marked with the SAM flag `3`, which means
  "properly paired and mapped"
- `-F 4`: exclude aligned reads with flag `4`: the read itself did not map
- `-F 8`: exclude aligned reads with flag `8`: their mates did not map
- `-F 256`: exclude alignments with flag `256`, which means
  that bwa mapped the read to multiple places in the reference genome,
  and this alignment is not the best
- `-F 1024`: exclude alignments marked with SAM flag `1024`, which
  indicates that the read is an optical or PCR duplicate (this flag
  would be set by Picard)
- `-F 2048`: exclude alignments marked with SAM flag `2048`,
  indicating chimeric alignments, where bwa decided that parts of the
  read mapped to different regions in the genome. These records are
  the individual aligned segments of the read. They usually indicate
  structural variation. We're not going to base peak calls on
  them.

Finally, we use a basic quality filter, `-q 30`, to request
high-quality alignments.

## <a name="callingpeaks"></a>Calling peaks on the aligned reads

We'll use [MACS2](https://github.com/taoliu/MACS) to "call peaks" in the aligned reads -- we're looking
for regions with lots of transposition events, which indicate open
chromatin.

```bash
$ macs2 callpeak -t SRR891268.ppmaq.nd.bam -n SRR891268.broad -g hs -q 0.05 --nomodel --shift -100 --extsize 200 -B --broad
```

The arguments are:

- `-t`: the "treatment" file -- the input, which is the sifted BAM
  file from the last step
- `-n`: the name of the experiment, which is used to name files
- `-g`: the genome's mappable size; 'hs' is an alias for the human
  genome's mappable size
- `-q`:  the false discovery rate cutoff for significant regions
  (peaks)
- `--nomodel`, `--shift`, and `--extsize`: MACS2 was designed for
  ChIP-seq data, so we're telling it not to use its built-in model,
  but to extend and shift reads in a way appropriate for ATAC-seq.
- `-B`: Create bedGraph files we'll use to create a browser track.
- `--broad`: request that adjacent enriched regions be combined into
  broad regions

## <a name="browsertrack"></a>Creating a browser track so we can look at the peaks in the UCSC Genome Browser

We're going to use a prefab browser track that contains peaks called
on the entire data, not the subset we're working with in the lab, but
this is how you would create a track for the called peaks:

```bash
$ LC_COLLATE=C sort -k1,1 -k2,2n SRR891268.broad_treat_pileup.bdg > SRR891268.broad_treat_pileup.sorted.bdg
$ bedGraphToBigWig SRR891268.broad_treat_pileup.sorted.bdg /class/data/bioinf525/hg19.chrom_sizes SRR891268.broad_peaks.bw
```

First open a web browser and navigate to the following URL:
http://research.nhgri.nih.gov/manuscripts/Collins/islet_chromatin/

At the bottom of the page, click on the track hub link.

This should open to the familiar GCK locus from lecture.

Now scroll down and click on "add custom tracks" and then in the "Paste URL or data" box, paste the following track:

```
track type=bigWig name="GM12878" description="GM12878 ATAC-seq" visibility=full color=255,128,0 alwaysZero=on maxHeightPixels=50:50:50 windowingFunction=mean smoothingWindow=3 bigDataUrl=https://theparkerlab.med.umich.edu/gb/tracks/bioinf525/gm12878.broad_treat_pileup.bw
```

Then click on "submit", then click "go" to return to the GCK locus now with GM12878 ATAC-seq data. How does the GM12878 chromatin accessibility look at GCK? At the two flanking genes, POLD2 and YKT6? Can you see the promoter region?

Now in the search menu, look for the following SNP ID:

`rs12946510`

Click on the first link to the NHGRI GWAS catalog hit. Then zoom out 100x so you can see a clear picture of the GM12878 chromatin accessibility at this SNP. Does it look like it occurs in an active regulatory element in GM12878? If you click on the SNP rsID it will take you to a page that explains the association. What is this SNP associated with? Does it make sense that it occurs in an active regulatory element for an immune cell type?


Note that the UCSC Genome Browser bigWig track format is explained here:
https://genome.ucsc.edu/goldenpath/help/bigWig.html


Now compare your ATAC-seq browser results to what you can find at HaploReg:
http://www.broadinstitute.org/mammals/haploreg/haploreg.php

Enter the SNP rsID in the query box and submit. How many SNPs are closely linked to this single SNP? What are the reference and alternate alleles (bases) for this SNP? Where does this SNP occur relative to genes? Next click on our SNP rsID of interest and explore the chromatin annotations across cells/tissues.
