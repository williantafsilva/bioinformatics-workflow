# Tutorial: Genome-wide Association Study (GWAS) using GEMMA.

## System setup (command line).

1. Log in to Dardel: `ssh <USERNAME>@dardel.pdc.kth.se`

2. Go to your working directory: `cd /cfs/klemming/projects/supr/sllstore2017078/${USER}-workingdir`

3. Create a directory that will contain your tutorial files (including scripts): `mkdir tutorial-gemmagwas`

4. Go to your tutorial directory: `cd tutorial-gemmagwas`

5. Import the GitHub repository containing the scripts: `git clone https://github.com/williantafsilva/scripts.git`

6. Define the path to your tutorial scripts: `PATHTOTUTORIALSCRIPTS="$(readlink -f scripts)"`

7. Add scripts to the command line PATH: `export PATH="${PATH}:${PATHTOTUTORIALSCRIPTS}"`

8. Make scripts executable:

```
chmod -R +x $(find ${PATHTOTUTORIALSCRIPTS} -type f)
chmod -R -x $(find ${PATHTOTUTORIALSCRIPTS} -name "*.txt")
chmod -R -x $(find ${PATHTOTUTORIALSCRIPTS} -name "*.md")
```

9. If you don't have one yet, create a directory to store copies of the submitted scripts: `mkdir submittedscripts`

10. If you don't have one yet, create a directory to store the slurm files: `mkdir slurmfiles`

11. Define required paths (this should be defined whenever a BASH session is initiated):

```
PATHTOMYSCRIPTS=$(readlink -f scripts)
PATHTOMYSUBMITTEDSCRIPTS=$(readlink -f submittedscripts)
PATHTOMYSLURM=$(readlink -f slurmfiles)
MYSLURMFILE="${PATHTOMYSLURM}/slurm-%J.out"
```

Slurm output file should be set to **-o ${MYSLURMFILE}** during sbatch submission.

## GEMMA input files.

GEMMA requires the following input files:

- **VCF file** (\**.vcf.gz*): Genotypes of the target samples.

- **Phenotype file**: Text file (\**.txt*) with tab-separated numeric values, no header or row names, as shown below.

- **Covariate file**: Text file (\**.txt*) with tab-separated numeric values, no header or row names, as shown below.

- **Relatedness matrix**: Text file (\**.txt*) with tab-separated numeric values, no header or row names, as shown below.

Example of an input file (e.g., phenotype file, covariate file) containing three phenotypes/covariates (columns) and five samples (rows):

$$
\begin{matrix}
  0.2 & 1 & 15 \\
  2 & 2 & 20 \\
  0.9 & 1 & 17 \\
  1.5 & 2 & 10 \\
  1.9 & 1 & 23
\end{matrix}
$$

Note that GEMMA does not receive the names of the columns/rows so it cannot keep track of the order of the samples in the different input files. Therefore, it is essential that the order of the samples is exactly the same in all input files. It is also important to keep track of the order of the phenotypes in the phenotype file, so that the results can be interpreted accordingly.

In order to run GEMMA, we will need to process the raw data so that the files meet the data requirements (e.g., MAF<0.05) and required file format.

## Get raw data for the tutorial.

1. Go to your tutorial directory (if not there yet): `cd tutorial-gemmagwas`

2. Copy the directory containing the raw data: `cp /cfs/klemming/projects/supr/sllstore2017078/wafds-workingdir/output/tutorial-gemmagwas-job20251107111514/rawdata .`

3. Create a directory to save the input files: `mkdir inputfiles`

## Process the raw data.

Define the output directory: `OUTPUTDIR=$(readlink -f inputfiles)`

### Look at the raw data.

- Open the phenotype/covariate data set: `less -S rawdata/rawdata-phenotypes-covariates.txt`

- List of samples in the phenotype/covariate data set: `tail -n+2 rawdata/rawdata-phenotypes-covariates.txt | cut -f1 | less`

- Number of samples in the phenotype/covariate data set: `tail -n+2 rawdata/rawdata-phenotypes-covariates.txt | wc -l`

- Open the genotype data set (VCF file): `zcat rawdata/rawdata-genotypes.vcf.gz | less -S`

- List of samples in the genotype data set (VCF file): 

```
module load bioinfo-tools bcftools
bcftools query -l rawdata/rawdata-genotypes.vcf.gz | less
```

- Number of samples in the genotype data set: `bcftools query -l rawdata/rawdata-genotypes.vcf.gz | wc -l`

- List of target samples (example used in this tutorial): `less rawdata/rawdata-list-targetsamples.txt`

- Number of target samples: `cat rawdata/rawdata-list-targetsamples.txt | wc -l`

### Get the list of samples (among the target samples) that have complete data (both phenotype and genotype data).

```
comm -12 <(cat rawdata/rawdata-list-targetsamples.txt | sort) <(
    comm -12 <(tail -n+2 rawdata/rawdata-phenotypes-covariates.txt | cut -f1 | sort) <(
        bcftools query -l rawdata/rawdata-genotypes.vcf.gz | sort)) > ${OUTPUTDIR}/list-samples-completedata.txt
```

### Process VCF file.

#### Subset VCF file with target samples.

```
GENOTYPEFILE=$(readlink -f rawdata/rawdata-genotypes.vcf.gz)
SAMPLELIST=$(cat ${OUTPUTDIR}/list-samples-completedata.txt | paste -sd,)

#Submit job (~15min).
sbatch \
    -A naiss2024-5-678 \
    -o ${MYSLURMFILE} \
    -p shared \
    -N 1 \
    -n 5 \
    -t 00-00:30:00 \
    -J vcf-selectsamples-GEMMATUTORIAL \
    vcf-selectsamples.sh ${OUTPUTDIR} ${GENOTYPEFILE} ${SAMPLELIST} TUTORIALTARGETSAMPLES

VCFFILE=$(readlink -f ${OUTPUTDIR}/rawdata-genotypes.selectTUTORIALTARGETSAMPLES-job*.vcf.gz)
```

Check sample in the output file: `bcftools query -l ${VCFFILE} | less`

#### Create data frame of genotypes for later use in genotype-phenotype plots.

```
#Submit job (~15min).
sbatch \
    -A naiss2024-5-678 \
    -o ${MYSLURMFILE} \
    -p shared \
    -N 1 \
    -n 5 \
    -t 00-00:30:00 \
    -J vcf-genotypematrix-GEMMATUTORIAL \
    vcf-genotypematrix.sh ${OUTPUTDIR} ${VCFFILE}
```

#### Select variants based on minor allele frequency (MAF>0.5) and fraction of samples with missing genotype (F_MISSING<0.05).

```
#Submit jobs (~10min). 
sbatch \
    -A naiss2024-5-678 \
    -o ${MYSLURMFILE} \
    -p shared \
    -N 1 \
    -n 5 \
    -t 00-00:30:00 \
    -J vcf-filtermafmiss-GEMMATUTORIAL \
    vcf-filtermafmiss.sh ${OUTPUTDIR} ${VCFFILE}

VCFFILE=$(readlink -f ${OUTPUTDIR}/rawdata-genotypes.selectTUTORIALTARGETSAMPLES.filtermafmiss-job*.vcf.gz)
```

Get order of samples in the VCF file: `bcftools query -l ${VCFFILE} > ${OUTPUTDIR}/list-samples-completedata.txt`

### Create GEMMA relatedness file from complete VCF file (all samples) and subset it to keep only target samples.

#### Create GEMMA relatedness matrix from complete VCF file.

```
#Submit job (~10min).
sbatch \
    -A naiss2024-5-678 \
    -o ${MYSLURMFILE} \
    -p shared \
    -N 1 \
    -n 5 \
    -t 00-00:30:00 \
    -J vcf-relatednessmatrix-GEMMA-GEMMATUTORIAL \
    vcf-relatednessmatrix-GEMMA.sh ${OUTPUTDIR} ${GENOTYPEFILE}
```

#### Select target samples and create relatedness matrix, matching the order of samples in the VCF file.

```
SAMPLELIST=$(readlink -f ${OUTPUTDIR}/list-samples-completedata.txt)
RELATEDNESSFILE=$(readlink -f ${OUTPUTDIR}/rawdata-genotypes.gemmarelatedness-job*.sXXwithheader.txt)

head -n1 ${RELATEDNESSFILE} > ${OUTPUTDIR}/tmp1-relatedness.txt
matrix-selectrows-stdout.sh ${RELATEDNESSFILE} ${SAMPLELIST} >> ${OUTPUTDIR}/tmp1-relatedness.txt
matrix-selectcolumns-stdout.sh ${OUTPUTDIR}/tmp1-relatedness.txt ${SAMPLELIST} | tail -n+2 > ${OUTPUTDIR}/matrix-relatedness.sXX.txt

rm -f ${OUTPUTDIR}/tmp*
```

### Process phenotype/covariate files.

#### Rename some variables in the phenotype/covariate data set (if necessary to, for example, remove spaces).

Here I change the names of the phenotypes from PHEN1 to PHENOTYPE1 and from PHEN2 to PHENOTYPE2.

```
cat rawdata/rawdata-phenotypes-covariates.txt | sed 's/PHEN1/PHENOTYPE1/' | sed 's/PHEN2/PHENOTYPE2/' > ${OUTPUTDIR}/phenotypes-covariates.txt
```

#### Add a covariate based on the sample name (just an example, if necessary in other situations).

Here I create a covariate COV9 which has value 1 in samples starting with letter A and value 2 in samples starting with letter B.

```
SAMPLELIST=$(readlink -f ${OUTPUTDIR}/list-samples-completedata.txt)
PHENOTYPEDATA=$(readlink -f ${OUTPUTDIR}/phenotypes-covariates.txt)

paste ${PHENOTYPEDATA} <(
    cat ${PHENOTYPEDATA} | cut -f1 | tail -n+2 | sed 's/[0-9].*//g' | sed '1iCOV9' | sed 's/A/1/g' | sed 's/B/2/g') > ${OUTPUTDIR}/tmp.txt
sed -i 's/\r//g' ${OUTPUTDIR}/tmp.txt
mv ${OUTPUTDIR}/tmp.txt ${PHENOTYPEDATA}
```

#### Convert character/factor variables into numeric variables.

Here I convert the character covariate COV7 into a numeric variable which has value 1 when COV7=Type1 and value 2 when COV7=Type2 (although in the tutorial data set there is no sample COV7=Type2).

```
sed -i 's/Type1/1/g' ${PHENOTYPEDATA}
sed -i 's/Type2/2/g' ${PHENOTYPEDATA}
```

#### Define the order of the samples in the phenotype/covariate file, matching the order of samples in the VCF file.

```
head -n1 ${PHENOTYPEDATA} > tmp1-phenotypes-covariates.txt
matrix-selectrows-stdout.sh ${PHENOTYPEDATA} ${SAMPLELIST} >> tmp1-phenotypes-covariates.txt
mv tmp1-phenotypes-covariates.txt ${PHENOTYPEDATA}
```

#### Create lists of covariates and phenotypes to be used in the analysis.

Here I select all covariates except COV6-7, and select all phenotypes (PHENOTYPE1 and PHENOTYPE2).

```
head -n1 ${PHENOTYPEDATA} | tr '\t' '\n' | grep "^COV" | grep -v "^COV6\|COV7" > ${OUTPUTDIR}/list-covariates.txt
head -n1 ${PHENOTYPEDATA} | tr '\t' '\n' | grep "^PHEN" > ${OUTPUTDIR}/list-phenotypes.txt
```

#### Select covariates and create GEMMA input matrix file.

```
matrix-selectcolumns-stdout.sh ${PHENOTYPEDATA} ${OUTPUTDIR}/list-covariates.txt | tail -n+2 > ${OUTPUTDIR}/matrix-covariates.txt
```

#### Select phenotypes and create GEMMA input matrix file.

```
matrix-selectcolumns-stdout.sh ${PHENOTYPEDATA} ${OUTPUTDIR}/list-phenotypes.txt | tail -n+2 > ${OUTPUTDIR}/matrix-phenotypes.txt
```

## Run GEMMA GWAS.

Create GEMMA output directory: `mkdir gemmagwas.output`

```
INPUTVCFFILE=$(readlink -f inputfiles/rawdata-genotypes.selectTUTORIALTARGETSAMPLES.filtermafmiss-job*.vcf.gz)
INPUTPHENOTYPEFILE=$(readlink -f inputfiles/matrix-phenotypes.txt)
INPUTCOVARIATEFILE=$(readlink -f inputfiles/matrix-covariates.txt)
INPUTRELATEDNESSFILE=$(readlink -f inputfiles/matrix-relatedness.sXX.txt)
OUTPUTDIR=$(readlink -f gemmagwas.output)

#Submit job (~3h).
sbatch \
    -A naiss2024-5-678 \
    -o ${MYSLURMFILE} \
    -p shared \
    -N 1 \
    -n 10 \
    -t 00-01:00:00 \
    -J vcf-phenotype-covariate-sXX-GWAS-GEMMA-GEMMATUTORIAL \
    vcf-phenotype-covariate-sXX-GWAS-GEMMA.sh ${OUTPUTDIR} ${INPUTVCFFILE} ${INPUTPHENOTYPEFILE} ${INPUTCOVARIATEFILE} ${INPUTRELATEDNESSFILE}
```

### Calculate FDR p-values and get significant hits.

Create output directory: `mkdir gemmagwas.results`

```
module load PDC/24.11 R/4.4.2-cpeGNU-24.11

OUTPUTDIR=$(readlink -f gemmagwas.results)
DATADIR=$(readlink -f gemmagwas.output/rawdata-genotypes.selectTUTORIALTARGETSAMPLES.filtermafmiss.gemmagwas-job*)
PHENOTYPELIST=$(readlink -f inputfiles/list-phenotypes.txt)
SUMMARYFILE=${OUTPUTDIR}/gemmagwas.output.FDR.txt

echo -e "Phenotype\tChromosome\tPosition\tBeta\tSE\tPvalueLRT\tPvalueFDR" > ${SUMMARYFILE}
find ${DATADIR}/*.assoc.txt | while read F ; do
    PHENOTYPEINDEX=$(echo ${F##*/} | sed 's/.*gemma_gwasoutput_phenotype\([0-9]*\)-job.*/\1/')
    PHENOTYPENAME=$(cat ${PHENOTYPELIST} | awk -v PHENOTYPEINDEX="${PHENOTYPEINDEX}" 'NR==PHENOTYPEINDEX')
    TMPFILE=$(echo ${OUTPUTDIR}/tmp-${PHENOTYPENAME}.txt)
    paste <(cat ${F}) <(Pvalue-FDR-stdout.sh ${F} 14 | sed '1iPvalueFDR') > ${TMPFILE}
    cat ${TMPFILE} | awk -v PHENOTYPENAME="${PHENOTYPENAME}" '($16 <= 1) {print PHENOTYPENAME"\t"$1"\t"$3"\t"$8"\t"$9"\t"$14"\t"$16}' >> ${SUMMARYFILE}
done

rm -f ${OUTPUTDIR}/tmp*
```

### Get significant hits (FDR<=0.2).

```
DATA=${OUTPUTDIR}/gemmagwas.output.FDR.txt
DATA_SIG=${OUTPUTDIR}/gemmagwas.output.significantFDR02.txt
head -n1 ${DATA} > ${DATA_SIG}
cat ${DATA} | awk '($7 <= 0.2) {print $0}' >> ${DATA_SIG}
```

### Get genotypes for significant SNPs.

```
GENOTYPES_COMPLETE=$(readlink -f inputfiles/rawdata-genotypes.selectTUTORIALTARGETSAMPLES.genotypematrixTGTformat-job*.txt)

paste -d':' <(cut -f2 ${DATA_SIG} | tail -n+2) <(cut -f3 ${DATA_SIG} | tail -n+2) | sort | uniq > ${OUTPUTDIR}/list-SNPs-significantFDR02.txt  

head -n1 ${GENOTYPES_COMPLETE} > ${OUTPUTDIR}/genotypes-significantFDR02.txt
grep -wF -f ${OUTPUTDIR}/list-SNPs-significantFDR02.txt ${GENOTYPES_COMPLETE} >> ${OUTPUTDIR}/genotypes-significantFDR02.txt
```

***Now explore the results on R!***
