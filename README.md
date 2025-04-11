# Processing of 16S Nanopore sequencing data 🧬

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
Convert all the .pod5 files saved in the /pod5/ folder to `.fastq.qz` files using the next script. Run this in the AI cluster or in a machine with GPU. Save the results in a new 01_Basecalling folder.

```bash
# Create the output folder
mkdir 01_Basecalling
# Basecall the .pod5 files
dorado duplex dna_r10.4.1_e8.2_400bps_sup@v4.2.0 /path/to/raw/data/pod5 --device cuda:all > 01_Basecalling/my_sequences.bam
# Create a summary file
dorado summary 01_Basecalling/duplex.bam > 01_Basecalling/sequencing_summary.txt
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
guppy_barcoder -i  01_Basecalling/ -s 01_Basecalling --barcode_kits "EXP-PBC096" --trim_adapters --device "cuda:all"
```
Some sequences will be classified inside unused barcodes or as unclassified. To double-check the unclassified reads, proceed as follows (optional):

```bash
# 1. Copy only the folders of the barcodes used during the experiment (for example from barcode01 to barcode 26) including the unclassified folder into a new 03_Demultiplexing folder.
mkdir 03_Demultiplexing
cp -r 01_Basecalling/barcode{01..n} 03_Demultiplexing

# 2. Copy the .fastq files of unused barcode files in the unclassified folder. Replace the (i) with the number of the unused barcodes
for f in 01_Basecalling/barcode(i)/*.fastq;
do
   cp -v "$f" 03_Demultiplexing/unclassified/"${f//\//_}"
done

# 3. Create a single file (unclassified.trim.fastq) of unclassified sequences
cat 03_Demultiplexing/unclassified/*.fastq > 03_Demultiplexing/unclassified.fastq

# 4. Demultiplex the unclassified reads using cutadapt
conda activate cutadapt
cutadapt -j 4 -e 0.05 -O 24 \
        -g file:/path/to/my/barcodes.fasta \
        -o 03_Demultiplexing/Demultiplexed/{name}.fastq \
        03_Demultiplexing/unclassified.fastq

# 5. Move the new demultiplexed files to the corresponding folder
for f in 03_Demultiplexing/Demultiplexed/barcode*.trim.fastq
do
        file=$(basename ${f} .fastq)
        cp ${f} 03_Demultiplexing/${file}
done
```
At the end, you should have a single folder for each barcode inside the 03_Demultiplexing folder

## 4. Filtering
Remove the low quality reads and the fragments with undesired lengths. In our case, we will keep reads with quality scores higher than 10 and a length between 1300 bp and 1900 bp. Revisit the quality check results to define the filtering threshold.

```bash
conda activate chopper-v.0.8.0
mkdir 04_Filtering

for file in 03_Demultiplexing/barcode*
do
        folder=$(basename ${file})
        mkdir 04_Filtering/${folder}
        cat ${file}/*.fastq > ${file}/${folder}_complete.fastq
        chopper --maxlength 1900 --minlength 1300 -q 10 -i ${file}/${folder}_complete.fastq  >  04_Filtering/${folder}/${folder}.filter.fastq
done
```

## 5. Trimming
Trim the primers and remaining adapters of the sequences.
```bash
conda activate cutadapt
mkdir 05_Trimming

for file in 04_Filtering/barcode*
do
       folder=$(basename ${file})
       mkdir 05_Trimming/${folder}
       cutadapt -g AGAGTTTGATCCTGGCTCAG  -a ACGGCTACCTTGTTACGACTT \
       --minimum-length 1300 --maximum-length 1900 -q 10 \
         ${file}/${folder}_complete.fastq -o 05_Trimming/${folder}/output_reads.fastq
done
```

## 6. Second quality control
Check the quality of the filtered and trimmed sequences and compare it with the files before cleaning.

```bash
mkdir 06_QualityControl_2
conda activate Nanoplot

# 1. Check the quality of the separated barcodes before cleaning
for file in 03_Demultiplexing/barcode*
do
        folder=$(basename ${file})
        mkdir 06_QualityControl_2/${folder}
        NanoPlot --fastq ${file}/${folder}_complete.fastq -o 06_QualityControl_2/${folder} -p ${folder}_before
done

# 2. Check the quality of the separated barcodes after cleaning
for file in 05_Trimming/barcode*
do
       folder=$(basename ${file})
       NanoPlot --fastq ${file}/output_reads.fastq -o 06_QualityControl_2/${folder} -p ${folder}_after
done

# 3. Create a summary table using the Stats results
for file in 06_QualityControl_2/barcode*
do
        folder=$(basename ${file})
        sed -n 2,8p ${file}/${folder}_beforeNanoStats.txt | sed 's/ /_/;s/ /_/' > ${file}/${folder}_Stats_before.txt
        sed -n 2,8p ${file}/${folder}_afterNanoStats.txt | sed 's/ /_/;s/ /_/' > ${file}/${folder}_Stats_after.txt
        join ${file}/${folder}_Stats_before.txt ${file}/${folder}_Stats_after.txt > ${file}/${folder}_join.txt
        echo -e ${folder} | cat - ${file}/${folder}_join.txt > ${file}/${folder}_summary.txt
done

cat 06_QualityControl_2/barcode*/barcode*_summary.txt > 06_QualityControl_2/Stats_summary.txt
```

In the Stats_summary.txt file you will find a summary with some of the parameters evaluated during the quality check. 

|barcode01 | Before | After|
| --- | --- | --- |
|Mean_read_length: |	1524.3 |	1524.6 |
|Mean_read_quality: |	12.3 |	13.1 |
|Median_read_length: |	1616	| 1520 |
|Median_read_quality: |	15.7	| 16.5 |
|Number_of_reads:	| 40553 |	21305 |
|Read_length_N50:	| 1622 |	1521 |
|STDEV_read_length:	| 499.2 |	65.1 |
|**barcode02**		|
| ... | ... | ... |

Verify the number of reads keep after the cleaning before proceed with the last step.

## 7. Taxonomic classification and abundance estimation
The taxonomic classification and abundance estimation is performed using [emu](https://github.com/treangenlab/emu). To proceed, download the database from the emu [OSF UI](https://osf.io/56uf7/). For this exercise, We downloaded the prebuilt SILVA database (silva.tar). 

```bash
# Create a file for the database and move the database there
mkdir EMU_DB/SILVA
mv Downloads/silva.tar EMU/SILVA
# Extract the files from the data base
tar -xvf EMU/SILVA/silva.tar

# Classification
mkdir 07_Classification

for file in 05_Trimming/barcode*
do
       folder=$(basename ${file})
       emu abundance ${file}/output_reads.fastq --threads 24 --db EMU/SILVA \
               --output-dir  07_Classification --output-unclassified \
               --output-basename ${folder} \
               --keep-counts --keep-read-assignments
done

emu combine-outputs $OUTDIR/EMU "tax_id" --counts

# Merge taxa files
awk 'FNR==1 && NR!=1 {next} {print}' 07_Classification/barcode*_rel-abundance.tsv > 07_Classification/complete_taxa.tsv
```
For each barcode you will get a rel-abundance.tsv file with the relative abundance of each taxon, a read-assigment-distribution.tsv file with the distribution of the reads and an unclassified.fq file with the reads that couldn't be classified. Additionally, you will find a emu-combined-tax_id-counts.tsv file that contains the abundance of each taxon per barcode and a complete_taxa.tsv file with the taxonomic classification of all the identified taxa. Use this two files for your further analysis, for example, as an input for [phyloseq](https://joey711.github.io/phyloseq/).  

Your emu-combined-tax_id-counts.tsv file should look like this:
|tax_id|barcode01|barcode02|...|
| --- | --- | --- | --- |
|10080 | 23.238703 | 24.18286 | ... |
|10126| 25.979147 | 11.303532 | ...|
| ... | ... | ... | ... |

And your complete_taxa.tsv file should look like this:

|tax_id	|abundance|	lineage|	estimated counts|
| --- | --- | --- | --- |
|10080	|0.003789243549658579|	Bacteria;Acidobacteriota;Acidobacteriae;Acidobacteriales;Acidobacteriaceae (Subgroup 1);Acidicapsa;|	79.90756797520011|
| 10126	|0.0011270978962392869	|Bacteria;Acidobacteriota;Acidobacteriae;Acidobacteriales;Acidobacteriaceae (Subgroup 1);Acidicapsa;acidisoli;	|23.768240435894082|

You can find and example of the outputs in the the [Examples](/Examples/) folder















   



