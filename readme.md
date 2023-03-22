# Pipeline

## mock reference genome

concerning the FastA file of the reference mock community; i downloaded it from the pdf file you provided me (link inside).
Then i created two fasta files : mock_chipfilter.fa & mock_chipfilter_V2.fa

the first one encompass all the genomes (size = 93.6 Mo)
the second one (V2) encompass all the dereplicated at 99% identity (size = 74.1 Mo) ... meaning i have removed the different coli strain genomes and keep only one.

## preparing the environment

We first create a directory for the project

```sh
mkdir chipfilter_metagenomic/
```
 and then a directory to store the FastQ files for the different samples

```sh
mkdir -p chipfilter_metagenomic/FastQ/
```

and finally a directory for the reference genomes

```sh
mkdir -p chipfilter_metagenomic/FastA/
```

we can now move the different FastQ and FastA files to this directory

```sh
mv *.gz  chipfilter_metagenomic/FastQ/
mv *.fa  chipfilter_metagenomic/FastA/
```
and also a perl script mandatory for calculating the coverage

```sh
mv jgi_summarize_bam_contig_depths chipfilter_metagenomic/
```

in case there is problem of execution, we can change the status of this script

```sh
chmod +X chipfilter_metagenomic/jgi_summarize_bam_contig_depths
```

finally, i create a directory for the output files and a for temporary ones

```sh
mkdir -p chipfilter_metagenomic/output/
mkdir -p chipfilter_metagenomic/temp/
```

## data cleaning

To clean the raw data, we used Trimmomatic with the following options --> ILLUMINACLIP:TruSeq3_PE_adapt.fa:2:30:10:2:True LEADING:3 TRAILING:3 MINLEN:36

```sh
for library in $(ls chipfilter_metagenomic/FastQ/ | sed 's/_R/ /' | awk '{print $1}' | sort -u)
do

    reads_for=chipfilter_metagenomic/FastQ/"$library"_R1_001.fastq.gz
    reads_rev=chipfilter_metagenomic/FastQ/"$library"_R2_001.fastq.gz
    out_fold=chipfilter_metagenomic/FastQ/
    project="$library"

    Trimmomatic PE -threads 32 "$reads_for" "$reads_rev" "$out_fold"/"$project"_cleaned_R1.fq.gz "$out_fold"/"$project"_unpaired_R1.fq.gz "$out_fold"/"$project"_cleaned_R2.fq.gz "$out_fold"/"$project"_unpaired_R2.fq.gz ILLUMINACLIP:TruSeq3_PE_adapt.fa:2:30:10:2:True LEADING:3 TRAILING:3 MINLEN:36

done
```

if you want, you can check the cleaning using FastQC

## mapping

We then mapp the cleaned Shotgun Reads on the mock reference using BOWTIE2.

the first step is to create an index for bowtie

```sh
mkdir -p chipfilter_metagenomic/index/
```

```sh
bowtie2-build chipfilter_metagenomic/FastA/mock_chipfilter_V2.fa chipfilter_metagenomic/index/mock_V2
bowtie2-build chipfilter_metagenomic/FastA/mock_chipfilter.fa chipfilter_metagenomic/index/mock
```

then we can perform the different alignments

you will also need SAMTOOLS software in order to generates BAM files.

and the little perl script to generate the coverage for each genomes

```sh

project=chipfilter_metagenomic/

for lib in $(ls "$project"/FastQ/ | sed 's/_R/ /' | awk '{print $1}' | sort -u)
do
    reads_for=chipfilter_metagenomic/FastQ/"$library"_cleaned_R1.fq.gz
    reads_rev=chipfilter_metagenomic/FastQ/"$library"_cleaned_R2.fq.gz
    index=chipfilter_metagenomic/index/mock_V2

    bowtie2 --very-sensitive-local -p 32  -x "$index" -U "$reads_for" -S "$project"/temp/for.sam
    samtools view -S -b "$project"/temp/for.sam > "$project"/temp/for_"$lib".bam

    bowtie2 --very-sensitive-local -p 32  -x "$index" -U "$reads_rev" -S "$project"/temp/rev.sam
    samtools view -S -b "$project"/temp/rev.sam > "$project"/temp/rev_"$lib".bam

    samtools sort "$project"/temp/for_"$lib".bam -o "$project"/temp/for_"$lib"_sort.bam

    samtools sort "$project"/temp/rev_"$lib".bam -o "$project"/temp/rev_"$lib"_sort.bam

done

chipfilter_metagenomic/./jgi_summarize_bam_contig_depths --outputDepth "$project"/output/coverage.tsv "$project"/temp/*_sort.bam

rm "$project"/temp/*.sam
rm "$project"/temp/rev_"$lib".bam
rm "$project"/temp/for_"$lib".bam
```

you can now play with the output files ;)
