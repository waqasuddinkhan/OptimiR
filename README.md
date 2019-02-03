# OptimiR
miRSeq data alignment workflow - integrates genetic information to assess the impact of variants on miRNA expression

### Requirements

The following python libraries must be installed: pysam and biopython (available through `pip`).
The following programs must be installed: bowtie2, cutadapt, samtools. If the path of any of these programs is not in `$PATH`, you can provide it directly to OPTIMIR (see `./OPTIMIR -h`).

As of nov. 2018 OptimiR has been only used (successfuly) in a GNU/Linux environment. Works fine with python2 and python3.

### Usage
```
./OPTIMIR --fq /path/to/sample.fq(.gz) \
	  --dir_output /path/to/output/dir/ \
	  --vcf /path/to/genotypes.vcf (optional)
```
Different options are available to customize the reference library and the outputs. Use `./OPTIMIR -h` to see all options. The user can provide a vcf file (and is recommanded to do so) with a list of variants, and OptimiR will extract variants located on miRNAs and generate new alternative sequences of these miRNAs (a.k.a polymiRs) integrating the alternative version of the variant.

In dir_output, the following directories are created:
 - `OptimiR_lib` : contains the alignment index generated by OptimiR, as well as a pickle directory containing python object (which allow the library to not be regenerated at each run if the inputed files did not change), and a fasta file which contains all the miRs and polymiRs sequences generated by OptimiR
 - `OptimiR_tmp` : contains temporary files generated during OptimiR run (files after trimming, collapsing and mapping steps, as well as cutadapt and bowtie2 reports). The final alignment file is available in the directory `3_PostProcess` in bam format. If `--rmTempFiles` option is given to OPTIMIR, this directory will be deleted. Tip : If OptimiR is run on the same sample more than once and `OptimiR_tmp` not deleted, the sample is not trimmed again, which saves computation time (except if option `--trimAgain` is provided).
 - `OptimiR_Results` : contains the abundances tables for each miRs and polymiRs, and detailed counts of isomiRs. If genotypes are provided, a consistency table is also generated and indicate the consistency between the alignment on a polymiR and the genotypes of the variants involved. Annotation files are also generated to provide information on ambiguous alignments (reads mapping on several references) that could not be resolved by OptimiR, and another file gives an indication on the parental hairpins from which the reads are originating (computed from templated tailed nucleotides).

### About the vcf file and the genotype consistancy analysis

The vcf file is optional. All variants provided in the vcf file that are mapping on miRNAs will be integrated in new sequences generated by OptimiR. Reads will be allowed to align to these new alternative mIRNA sequences.

The genotype can also be provided by the user in the vcf file, in which case OptimiR will perform a genotype consistency step and retain only alignments on polymIRs that are consistent with the genotype.
**The sample name in the vcf file must match the name of the provided fastq** (minus the extension of the fastq file)

When a vcf with genotypes is provided, 2 tables : `consistency_table` and `polymiRs_table` are generated in the outputs and contain summary alignments on each polymiR. In the polymiR table, a detail of alignments on each polymiR for each sample is provided, with the number of reads aligned on the sequences integrating the reference allele and alternative allele. 

In the consistency table, consistent counts are reads that aligned on the polymiR sequence with the allele that match the genotype, while inconsistent counts are alignments that have been discarded because they didn't match the genotype. For samples with heterozygous genotype, reads can align on any version of the polymiR, in which case the Allele Specific Expression (ASE) of these polymiRs can be studied.

###### A few limitations:
 - Multi-allelic variants must be provided on two separate lines. Allele string must not contain a comma, and genotype format must have format 0/0, 0/1, 1/0 or 1/1 but no 0/2 and such... (btw: also works with phased variants format).
 - Combination of indels, or indels and SNPs: if a SNP and an indel map on the same miRNA, or if several indels map on the same miRNA, combinations of these variants might not be generated, in which case a warning will be displayed during library preparation. Combinations of SNPs works fine.
 - Variant positions must respect GRCh38 coordinates (when using miRBase 21 default gff3 file).

### Plots with R

Two R scripts are available in the directory `src/R_plot/`: `plot_ASE.R` and `plot_isoDist.R`. These scripts are meant to be run on the files `polymiRs_table.annot` and `isomiRs_dist.annot`, respectively.

##### plot_ASE.R : representing Allele Specific Expression (ASE)
For polymiRs integrating genetic variants expressed by heterozygous carriers, both sequences integrating the reference or the alternative allele can be expressed. It might be of interset for the user to analyse the ASE of these polymiRs. For this we provide the script `plot_ASE.R` that automatically gather counts on polymiRs expressed by heterozygotes, and generate a scatter plot representing for each sample the reads aligned on the polymiR integrating the reference allele vs the counts gathered by the sequence integrating the alternative allele.

##### plot_isoDist.R : representing the distribution of isomiRs
miRNAs can undergo post transcriptional modifications that can affect the sequence of the mature miRNA. These modifications mostly affect the 3' extremity of the miRNA, and on a smaller scale the 5' end. A miRNA that does not appear to have undergone such modifications is called *canonical*, while miRNAs that are not canonical are *isomiRs*. The modifications observed are on each end of the isomiR can be:
 - Trimming (Trim): a deletion of one or more nucleotides.
 - Templated tailing (TA) : the addition of one or more nucleotides, that matches the nucleotides that originally surround the mature miRNA sequence in the primary miRNA.
 - Non templated tailing (NTA) : the addition of one or more nucleotides, that does not match the nucleotides that originally surround the mature miRNA.
 - Trimming followed by tailing (Trim+Tail): The deletion of one or more nucleotides followed by the addition of one or more nucleotides.
 - Canonical : no modification.

The script `plot_isoDist.R` allows the user to represent the distribution of these modifications in the processed dataset. By default, only the global distribution accross all miRNA is generated. The script can be easily tweaked by the user if there is a need to represent the distribution of individual miRNA.

### Output files

All the outputs are in tsv format, and easy to process for downstream analyses.

##### Abundance files

First column contains miRNA ids, second column contains sample's counts.
For each sample, 4 abundance files are generated:
 - `miRs_and_polymiRs_abundances` : counts gathered by each miRNA. PolymiRs have counts detailed on each sequence (integrating reference/alternative allele). The sequences that integrate an alternative allele is written as follow : miR_name + `'_'` + rsID. For example the polymiR hsa-miR-4433b-3p integrating the alternative allele for `rs12473206` has its counts accessible with the id `hsa-miR-4433b_rs12473206`, while counts gathered by the sequence integrating the reference allele are on the line with the id `hsa-miR-4433b-3p`.
 - `miRs_merged_abundances` : Same as above, except that counts of each polymiRs are merged under a single id. For example, counts gathered by `hsa-miR-4433b-3p` and `hsa-miR-4433b-3p_rs12473206` are summed and available with the id `hsa-miR-4433b-3p`.
 - `polymiRs_specific_abundances` : same as `miRs_and_polymiRs_abundances` except that only polymiRs are presented in this table.
 - `isomiRs_specific_abundances` : counts gathered by each isomiR. Each isomiR is represented according to the *isotype* format (see below).
```
ISOTYPE FORMAT: miRNA-ID~[5'end modifications, 3'end modifications]
--------------
Modifications description:
 +N : templated tailing with nucleotide N
 +n : non templated tailing with nucleotide N
 -x : trimming of x nucleotides

Example of alignments on hsa-miR-4433b-3p, with their isotype
  ccagaaaCAGGAGTGGGGGGTGGGACGTaaggagg : reference mature sequence, with surrounding nucleotides in pri-miRNA
         ---------------------
	     mature miRNA

       AACAGGAGTGGGGGGTGGGACGT	:hsa-miR-4433b-3p~[+AA,]
        ACAGGAGTGGGGGGTGGGACGT	:hsa-miR-4433b-3p~[+A,]
        ACAGGAGTGGGGGGTGGGACG	:hsa-miR-4433b-3p~[+A,-1]
         CAGGAGTGGGGGGTGGGACGTA	:hsa-miR-4433b-3p~[,+A]
         CAGGAGTGGGGGGTGGGACGTt	:hsa-miR-4433b-3p~[,+t]
         CAGGAGTGGGGGGTGGGACGT	:hsa-miR-4433b-3p~[,]
         CAGGAGTGGGGGGTGGGACG	:hsa-miR-4433b-3p~[,-1]
         CAGGAGTGGGGGGTGGGACt	:hsa-miR-4433b-3p~[,-2+t]
         CAGGAGTGGGGGGTGGGAC	:hsa-miR-4433b-3p~[,-2]	 
```

##### Additional files

For each samples several complementary files can be generated (the user can control which ones with option `--annot`):
 - `consistency_table` : For each polymiR, reports all counts that are consistent with genotype, and the number of reads that were discarded because of inconsistency. If more than 5 counts or 1% of total reads were discarded, the flag "SUSPICIOUS" is added in the `flag` column (the 0.01 default rate can be modified by the user with the option `--consistentRate`). This threshold is set to detect misalignments on polymiRs that are not due to sequencing errors that might replicate a genetic variant.
 - `polymiR_table` : For each polymiR, report the number of reads gathered by the sequence integrating the reference allele, and the number of reads aligned on the sequence integrating the alternative allele.
 - `isomiRs_dist` : For each miRNA / polymiR, report the percentage of modifications on each end, the global percentage of canonical alignments, as well as the global read counts.
 - `inconsistents.sam` : file reporting all alignments that have been discarded because of an inconsistency between the provided genotype and the alignment on a polymiR.
 - `expressed_hairpins` : file reporting the primary miRNA and thus the locus from which a miRNA is transcribed. In miRBase, mature miRNAs with identical sequence can originate from different hairpins, and thus different genomic loci. When a read is aligned on such miRNAs with templated tailing, and if the nucleotides surrounding the mature miRNA are different in the potential parental hairpins, we use this information to deduce from which hairpin the miRNA might originate. When using this information, the user must be aware that templated tailing might not be due to imprecise cleavage from ribonucleases during the miRNA maturation, but could be due to terminal nucleotidyl transferase that add the same nucleotides to the mature miRNA end than those originally surrounding it in the hairpin. The reporting format of this information is detailed below.
 - `remaining_ambiguous` : When a read align on multiple references, we call it ambiguous, or cross-mapping. OptimiR has an internal scoring algorithm that allow it to choose the most probable alignment when possible. If tipically ~90% of ambiguous alignments are resolved, the remaining ones that could not be resolved are detailed in this file. For the remaining ambiguous, if a read aligns on n references, then each alignment receive a weight of 1/n. The reporting format of this information is detailed below. 
```
Example:
-------
REFERENCE	SAMPLE	Expressed_hairpins								Remaining_ambiguous
hsa-let-7a-3p	525	multiple[100.00%]	
hsa-let-7a-5p	471517	hsa-let-7a-2[0.02%]/multiple[99.69%]/hsa-let-7a-3[0.18%]/hsa-let-7a-1[0.10%]	hsa-let-7c-5p+hsa-let-7b-5p[2278(0.48%)]/hsa-let-7c-5p[27534(5.84%)]
hsa-let-7b-5p	3842888	hsa-let-7b[100.00%]								hsa-let-7c-5p+hsa-let-7a-5p[2278(0.06%)]
hsa-let-7c-5p	51910	hsa-let-7c[100.00%]								hsa-let-7a-5p[27534(53.04%)]/hsa-let-7a-5p+hsa-let-7b-5p[2278(4.39%)]
hsa-let-7f-5p	439810	hsa-let-7f-1[0.36%]/hsa-let-7f-2[24.53%]/multiple[75.11%]	
```
###### Expressed_hairpins format: `potential-hairpin-id[percentage of reads with templated tailing matching potential-hairpin-id]`

If a miRNA has only one hairpin from which it originate, then it will always have 100% of its reads originating
from it. Ex: `miRNA-hairpin-id[100%]`
If a miRNA has more than one potential hairpin, each hairpin with matching templated tailing is reported, with
the percentage of total aligned reads reported in brackets.
If several hairpins are reported, they are separated with a `'/'` symbol.
For alignments where the parental hairpin cannot be distinguished using templated tailing, these are reported
as `'multiple'`.
In the example, reads aligned on `hsa-let-7f-5p` have 0.36% of aligned reads that are potentially originating
from the hairpin `hsa-let-7f-1`, 24.53% from `hsa-let-7f-2`, and the remaining 75.11% could originate from any of these
two hairpins.

###### Remaining_ambiguous format: `mature-id[number_of_reads_involved(percentage_of_reads_involved)]`

We report here other matures miRNA on which reads ambiguously aligned. These alignments could not be resolved
using OptimiR's scoring algorithm to determine the most probable alignment.
If a read cross mapped on more than one other miRNA, then they are all reported and separated with a `'+'` symbol.
In the example, 5.84% of reads aligned aligned on `hsa-let-7a-5p` (which amount to 27,534 reads) also mapped
on `hsa-let-7c-5p`. And 2,278 reads (or 0.48% of reads aligned to `hsa-let-7a-5p`) aligned simulteanously
to `hsa-let-7a-5p`, `hsa-let-7b-5p` and `hsa-let-7c-5p`.
This happens a lot to reads mapping on `let-7` mature miRNAs, because their sequences are very similar.


### Summary Files

If several samples must be processed, it is recommanded to provide the same output directory for all runs. The python script `OPTIMIR_SUMMARY` with the output directory as a parameter will produce summary files for all runs (abundances tables and additional files).

Usage: `python OPTIMIR_SUMMARY /path/to/Results/`

### Run on an example

We provide 2 very small fastq samples and a vcf file with OptimiR, available in the directory `example`. In the main directory, 2 scripts allow the user to run OptimiR on these examples, either on a single sample (`example_1.sh`) or on both samples sequentially (`example_all.sh`).

To run the script, use `./example_1.sh` or `./example_all.sh`

The script `example_all.sh` has 3 steps:
1. Prepare the alignment library ahead with the provided vcf. OptimiR will rely on this library to quantify reads from the 2 samples. Their genotypes for a set of variants loacted on miRNAs are included in the vcf file (`genotypes.vcf`). **This step is mandatory when processing samples in parallel.**
1. Run OPTIMIR sequentially on both samples. Since the library is already prepared and available in the provided output directory, OptimiR won't prepare it again and will align and quantify miRNAs using this library.
1. Run OPTIMIR_SUMMARY on the output directory, in order to gather the results from both samples in single files (abundances and annotation tables).

### OPTIMIR -h
```
usage: OPTIMIR [-h] -i FASTQ [-o OUTDIR] [-g VCF] [--seedLen SEEDLEN]
               [--w5 WEIGHT5] [--scoreThresh SCORE_THRESH]
               [--consistentRate INCONSISTENT_THRESHOLD] [--rmTempFiles]
               [--annot ANNOT_FILES] [--adapt3 ADAPT3] [--adapt5 ADAPT5]
               [--readMin READMIN] [--readMax READMAX] [--bqThresh BQTHRESH]
               [--trimAgain] [--maturesFasta MATURES]
               [--hairpinsFasta HAIRPINS] [--gff3 GFF3] [--quiet]
               [--cutadapt CUTADAPT] [--bowtie2 BOWTIE2]
               [--bowtie2_build BOWTIE2_BUILD] [--samtools SAMTOOLS]

OptimiR: A bioinformatics pipeline designed to detect and quantify miRNAs,
isomiRs and polymiRs from miRSeq data, & study the impact of genetic
variations on polymiRs' expression

optional arguments:
  -h, --help            show this help message and exit
  -i FASTQ, --fq FASTQ  Full path of the sample fastq file (accepted formats
                        and extensions: fastq, fq and fq.gz)
  -o OUTDIR, --dirOutput OUTDIR
                        Full path of the directory where output files are
                        generated
  -g VCF, --vcf VCF     Full path of the vcf file with genotypes
  --seedLen SEEDLEN     Choose the alignment seed length used in option '-L'
                        by Bowtie2 (default:17)
  --w5 WEIGHT5          Choose the weight applied on events detected on the 5'
                        end of aligned reads (default:4)
  --scoreThresh SCORE_THRESH
                        Choose the threshold for alignment score above which
                        alignments are discarded (default:9)
  --consistentRate INCONSISTENT_THRESHOLD
                        Choose the rate threshold for inconsistent reads
                        mapped to a polymiR above which the alignment is
                        flagged as highly suspicious (default:0.01)
  --rmTempFiles         Add this option to remove temporary files (trimmed
                        fastq, collapsed fastq, mapped reads, annotated bams
  --annot ANNOT_FILES   Control which annotation file is produced by adding
                        corresponding letter : 'h' for expressed_hairpins, 'p'
                        for polymiRs_table, 'i' for consistency_table, 'c' for
                        remaining_ambiguous, 's' for isomiRs_dist. Ex: '--
                        annot hpics' (default) will produce all of them
  --gff			Add this option to generate results in mirGFF3 format
  --vcf			Add this option to generate genotypes inferred by OptimiR
  			in VCF format
  --adapt3 ADAPT3       Define the 3' adaptor sequence (default is NEB &
                        ILLUMINA: AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC -a
                        TGGAATTCTCGGGTGCCAAGG -> hack: use -a to add adapter
                        sequences
  --adapt5 ADAPT5       Define the 5' adaptor sequence (default is NEB:
                        ATCTACACGTTCAGAGTTCTACAGTCCGACGATC
  --readMin READMIN     Define the minimum read length defined with option -m
                        in cutadapt (default: 15)
  --readMax READMAX     Define the maximum read length defined with option -M
                        in cutadapt (default: 27)
  --bqThresh BQTHRESH   Define the Base Quality threshold defined with option
                        -q in cutadapt (default: 28)
  --trimAgain           Add this option to trim files that have been trimmed
                        in a previous application. By default, when temporary
                        files are kept, trimmed files are reused. If you wish
                        to change a paramater used in the trimming step of the
                        workflow, this parameter is a must.
  --maturesFasta MATURES
                        Path to the reference library containing mature miRNAs
                        sequences (default: from miRBase v21)
  --hairpinsFasta HAIRPINS
                        Path to the reference library containing pri-miRNAs
                        sequences (default: from miRBase v21)
  --gff3 GFF3           Path to the reference library containing miRNAs and
                        pri-miRNAs coordinates (default: from miRBase v21,
                        GRCh38 coordinates)
  --quiet               Add this option to remove OptimiR progression on
                        screen
  --cutadapt CUTADAPT   Provide path to the cutadapt binary
  --bowtie2 BOWTIE2     Provide path to the bowtie2 binary
  --bowtie2_build BOWTIE2_BUILD
                        Provide path to the bowtie2 index builder binary
  --samtools SAMTOOLS   Provide path to the samtools binary
```

### UPDATES

 - 02/03/19 :
   - Supports outputs in mirGFF3 format (see the [mirtop/mirGFF3 GitHub page](https://github.com/miRTop/mirGFF3) for more details on this format and the mirtop project)
   - Supports outputs in VCF format summarizing the genotypes inferred by OptimiR

