# BWA-MEM2 benchie

This benchmarking repo [originates from the need of testing some `zlib` variants for bwa-mem2](https://github.com/bioconda/bioconda-recipes/pull/56242#issuecomment-2915839436).

## Human (GiAB h37)

There are practical problems with benchmarking `bwa-mem2` with human genomes on public (limited resources) CI runners such as GitHub actions.

For starters, just [indexing the human genome requires at least 60GB of RAM](https://github.com/bwa-mem2/bwa-mem2/issues/141#issuecomment-2686311376).

Furthermore, there are interesting threading effects that [require machines having up to 64 cores](https://github.com/bioconda/bioconda-recipes/pull/56407#issuecomment-2917034411) if
we'd like to see scalability that neatly translate to "real world bioinformatics" pipeline scenarios (at least those based exclusively on CPUs, not GPUs, FPGA nor other custom accelerators).

OTOH, if there's access to custom runners that could routinely satisfy the harware requirements above, here's a simple recipe on how to get this (roughly) up and running:

```shell
$ wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/human_g1k_v37.fasta.gz
$ wget ftp://ftp-trace.ncbi.nih.gov/giab/ftp/data/AshkenazimTrio/HG002_NA24385_son/fastq/NIST7035_TAAGGCGA_L001_R1_001.fastq.gz
$ gunzip human_g1k_v37.fasta.gz
$ bwa-mem2 index human_g1k_v37.fasta
$ bwa-mem2 mem -t$(nproc) human_g1k_v37.fasta NIST7035_TAAGGCGA_L001_R1_001.fastq.gz
```

But would other organisms with smaller genomes be a good substitute for lower resource intensive CI workers?

## E.coli

The following code snippet grabs **Escherichia coli** genome and generates a FASTQ dataset out of generated reads... in the hopes that
in-silico performance metrics (linearly?) match those from the human genome described previously.

A few more questions linger, though:

1. Would a smaller organism (PhiX) be still representative for benchmarking purposes?
1. Does aligning different organisms under bwa-mem2 behave like human genomes or will there be noticeable runtime artifacts due to different I/O and CPU/caching/memory allocation patterns?

```shell
$ conda install art bwa-mem2
$ gunzip GCF_000005845.2_ASM584v2_genomic.fna.gz
$ art_illumina -ss HS25 -i GCF_000005845.2_ASM584v2_genomic.fna -rs 666 -l 100 -f 1 -o ecoli_art_illumina_simlated_reads

    ====================ART====================
             ART_Illumina (2008-2016)
          Q Version 2.5.8 (June 6, 2016)
     Contact: Weichun Huang <whduke@gmail.com>
    -------------------------------------------

                  Single-end Simulation

Total CPU time used: 0.814264

The random seed for the run: 666

Parameters used during run
	Read Length:	100
	Genome masking 'N' cutoff frequency: 	1 in 100
	Fold Coverage:            1X
	Profile Type:             Combined
	ID Tag:

Quality Profile(s)
	First Read:   HiSeq 2500 Length 126 R1 (built-in profile)

Output files

  FASTQ Sequence File:
	ecoli_art_illumina_simlated_reads.fq

  ALN Alignment File:
	ecoli_art_illumina_simlated_reads.aln

$ time bwa-mem2 index GCF_000005845.2_ASM584v2_genomic.fna
[bwa_index] Pack FASTA... 0.04 sec
* Entering FMI_search
init ticks = 2831479
ref seq len = 9283304
binary seq ticks = 1749084
build suffix-array ticks = 11281796
ref_seq_len = 9283304
count = 0, 2284124, 4641652, 6999180, 9283304
BWT[1462939] = 4
CP_SHIFT = 6, CP_MASK = 63
sizeof CP_OCC = 64
pos: 1160414, ref_seq_len__: 1160413
max_occ_ind = 145051
build fm-index ticks = 3974676
Total time taken: 0.8631
bwa-mem2 index GCF_000005845.2_ASM584v2_genomic.fna  0.82s user 0.05s system 46% cpu 1.893 total

$ bwa-mem2 mem GCF_000005845.2_ASM584v2_genomic.fna ecoli_art_illumina_simlated_reads.fq > aln.sam
```
