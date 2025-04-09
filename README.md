# Data processing for 16S Nanopore sequencing 🧬

This document describes the steps to process sequencing data obtained from Oxford Nanopore Technology (ONT) for a study of 16S metabarcoding.
Before starting, please ensure you have installed all the required dependencies. This can be done using conda. If you need to install conda, follow the instructions provided here: https://gist.github.com/kauffmanes/5e74916617f9993bc3479f401dfec7da

## Requirements and dependencies

1. [Dorado](https://github.com/nanoporetech/dorado)
2. [Nanoplot](https://github.com/wdecoster/NanoPlot)
3. [PycoQC](https://a-slide.github.io/pycoQC/)
4. [Samtools](https://github.com/samtools/samtools)
5. [Guppy](https://nanoporetech.com/document/Guppy-protocol#guppy-software-overview)
6. [Chopper](https://github.com/wdecoster/chopper)
7. [Cutadapt](https://cutadapt.readthedocs.io/en/stable/)
8. [emu](https://github.com/treangenlab/emu)

You will also need the primers and the list of the barcodes (as a fasta file) used during the study. In this case, the amplicons were obtained using the 27F (5'-AGAGTTTGATCCTGGCTCAG-3') and the 1492R (5'-ACGGCTACCTTGTTACGACTT-3'), and our barcodes list looks like this:
```bash
>barcode01
AAGAAAGTTGTCGGTGTCTTTGTG
>barcode02
TCGATTCCGTTTGTAGTCGTCTGT
>barcode03
GAGTCTTGTGTCCCAGTTACCAGG
>barcode04
TTCGGATTCTATCGTGTTTCCCTA
>barcode05
CTTGTCCAGGGTTTGTGTAACCTT
...
```

## 1. Basecalling
Convert all the .pod5 files saved in the /pass/ folder to `.fastq.qz` files using the next script. Run this in the AI cluster or in a machine with GPU. Save the results in a new 01_Basecalling folder.

```bash
# Create the output folder
mkdir 01_Basecalling
# Basecall the .pod5 files
dorado duplex dna_r10.4.1_e8.2_400bps_sup@v4.2.0 /path/to/raw/data/pass --device cuda:all > 01_Basecalling/my_sequences.bam
# Create a summary file
dorado summary 01_Basecalling/duplex.bam > $OUTDIR/sequencing_summary.txt
```
## 2. Quality control
Check the quality of the data using Nanoplot or pycoQC. Use the sequencing_summary.txt file in the 01_Basecalling folder and save the results in a new 02_QualityControl folder.

```bash
# Create the output folder
mkdir 02_QualityControl
# Quality check using Nanoplot
conda activate Nanoplot
NanoPlot --summary 01_Basecalling/sequencing_summary.txt -o 02_QualityControl -p After_Basecalling
# Quality check using pycoQC
conda activate pycoQC
pycoQC -f 01_Basecalling/sequencing_summary.txt -o 02_QualityControl/After_Basecalling.html --quiet
```
You will get a condensed report After_Basecalling-report.html with figures like this:

![](/Figures/pycoQC_result.png)

Explore the results to decide the appropriate threshold  to filter your data.

## 3. Demultiplexing
After basecalling, you should have a unique .bam file. To separate the reads in barcodes, convert the .bam file to .fastq and demultiplex using Guppy. Run this part in a machine with GPU.
```bash
# Convert the .bam to a .fastq file
conda activate Samtools
samtools fastq 01_Basecalling/my_sequences.bam > 01_Basecalling/my_sequences.fastq
# Demultiplex using Guppy
guppy_barcoder -i  01_Basecalling/ -s 01_Basecalling/ --barcode_kits "EXP-PBC096" --trim_adapters --device "cuda:all"
```








   



