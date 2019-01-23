# Local De Novo Assembly with Long Reads
A quick tutorial on local de novo assembly of long reads for SV validation

------
## Tutorial

Perform local de novo assembly for a heterozygous 6kb inversion reported by the 1000 Genomes Project.

Bam file is a subset on chromosome 10. BLASR aligned to GRCh38 for individual HG00513. 

Inversion call

```
#hg19
10	56767192	56773196	1|0	HG00513	

#hg38
10	55007432	55013436	1|0	HG00513
```

----------------------------

## Methodology

1. Convert BAM -> FASTQ 
2. Map with Minimap2/NGMLR
3. Extract Split-Reads and Convert to FASTQ
4. Local De Novo Assembly of Split-Reads with Flye
5. Align Contigs with LAST

----------------------------

## Requires 

* python 2.7 (for Flye)
* samtools
* minimap2 / ngmlr
* picard 
* last

---------------------------

## Raw Split-Reads 

* Top: Minimap2; Bottom: NGMLR

The red bar on top is the INV call. 

Red reads are on the forward strand, blue reads on the reverse strand. 

In both alignments, the same molecule is highlighted (bold red). If you look closely enough you can see the selected alignment is on the forward strand (arrow pointing right), while the selected alignments flanking the breakpoint are on the reverse strand (pointing left)

![alt text](https://raw.githubusercontent.com/dantaki/Local-De-Novo-Assembly-with-Long-Reads/master/1kgp_inv_example.png)

---------------------------

## Assembled Contigs from Split-Reads

* Top: assembled contig from Minimap2 split-reads; Bottom: assembled contig from NGMLR split-reads

The red bar on top of the image shows the INV call. You can see for both assemblies the contig is split according to read strand: red, forward; blue. reverse. 

The assembled contigs are in these SAM files 

* 1kgp_minimap2_flye.sam
* 1kgp_ngmlr_flye.sam

![alt text](https://raw.githubusercontent.com/dantaki/Local-De-Novo-Assembly-with-Long-Reads/master/1kgp_inv_asm.png)

---------------------------
## Commands

### 1 bam2fq

```
samtools bam2fq 1kgp.bam  >raw.fq
```

### 2 map with minimap2 or ngmlr

```
minimap2 -ax map-pb human_g1k_v37.fasta raw.fq >minimap2.sam

ngmlr -r human_g1k_v37.fasta -q raw.fq -x pacbio >ngmlr.sam
```

### 3 extract split-reads 

```
# get the header
samtools view -H minimap2.sam  >minimap2.sr.sam

# grep split-reads
samtools view minimap2.sam | grep "SA:Z"  >>minimap2.sr.sam

# convert to FASTQ
samtools bam2fq minimap2.sr.sam >minimap2.sr.fq
```

### 4 Assemble Contigs with Flye

```
# -g is the genome size, which is around 15kb for the region in question
## --meta is for metagenomes / uneven coverage. Since we are assembling split-reads 
## the coverage is very uneven. Might want to experiment with this option, though 

flye  --pacbio-raw minimap2.sr.fq -g 15k -o minimap2_sr_flye --meta
```

### 5 Align with LAST
```
# make a reference, just from chromosome 11

samtools faidx human_g1k_v37.fasta 11 >grch37.chr10.fa
java -jar picard.jar CreateSequenceDictionary REFERENCE=grch37.chr10.fa OUTPUT=grch37.chr10.dict
lastdb -uNEAR -R01 grch37.chr10.last grch37.chr10.fa

# optional, clean up contig names for LAST
awk '/>/ {$0 = ">" ++n} 1' minimap2_sr_flye/scaffolds.fasta >minimap2_sr_flye.fa

# create paramter file for LAST
last-train -Q0 grch37.chr10.last minimap2_sr_flye.fa >minimap2_sr_flye.par

# align with LAST and convert MAF file to SAM format
lastal -p minimap2_sr_flye.par grch37.chr10.last minimap2_sr_flye.fa | last-split -m1 >minimap2_sr_flye.maf
maf-convert -f grch37.chr10.dict sam minimap2_sr_flye.maf >minimap2_sr_flye.sam
```

-------------------

## Useful Links

* [FLYE Assembler](https://github.com/fenderglass/Flye)
* [LAST Source](http://last.cbrc.jp/)
* [LAST Alignments for Long Reads](https://github.com/mcfrith/last-rna/blob/master/last-long-reads.md)

Also you can call SVs with LAST alignments using NanoSV, however the last time I checked NanoSV's source code it was written in Perl and could only call INS, DEL, and DUP
NanoSV is now written in python and maybe can call INV now. 

* [NanoSV](https://github.com/mroosmalen/nanosv)
