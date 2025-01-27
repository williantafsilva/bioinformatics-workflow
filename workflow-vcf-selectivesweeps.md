# From VCF to selective sweeps (SweepFinder2, SweeD, OmegaPlus, Rehh)

Starting file: \*.bcf file (CC5KBCF).

## Create output directory.

```
cd ${PATHTOPROJOUTPUT}
mkdir-stdout.sh . cc5k-feralsamples
cd cc5k-feralsamples-job*
```

## Select feral chicken samples from BCF file.

### Submit job.

```
#Submit job (~7h).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-12:00:00 \
    -J vcf-selectsamples-cc5k-feral \
    vcf-selectsamples.sh . ${CC5KBCF} $(vcfsamples-stdout.sh ${CC5KBCF} | grep "_linkoping" | paste -sd,) FERAL 
```

### Rename output file (make it shorter).

```
#Rename output file.
mv-stdout.sh \
    autosomes.4993.annot.merged.filtered.selectFERAL-job*.vcf.gz \
    cc5k.selectFERAL-$(echo autosomes.4993.annot.merged.filtered.selectFERAL-job*.vcf.gz | grep -o 'job[0-9]*').vcf.gz 
```

## Rename samples.

### Create sample names conversion file.

```
#Create sample names conversion file (requires file sampleanamesconversion.txt).
module load bioinfo-tools
module load bcftools
vcfsamples-stdout.sh *.selectFERAL-job*.vcf.gz > samplenamesold.txt

cp samplenamesold.txt samplenamesnew.txt

paste -d' ' <(cut -d' ' -f1 sampleanamesconversion.txt) <(cut -d' ' -f2 sampleanamesconversion.txt) | while IFS=' ' read n k ; do sed -i "s/$n/$k/g" samplenamesnew.txt ; done

sed -i 's/_linkoping//g' samplenamesnew.txt

paste -d' ' samplenamesold.txt samplenamesnew.txt > samplenamesconversion.txt

rm samplenamesold.txt samplenamesnew.txt
```

### Submit job.

```
#Submit job (~10min).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-01:00:00 \
    -J vcf-renamesamples-cc5k-feral \
    vcf-renamesamples.sh . *.selectFERAL-job*.vcf.gz samplenamesconversion.txt
```

## Sort and index VCF file.

```
#Submit job (~4h).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-10:00:00 \
    -J vcf-sort-index-cc5k-feral \
    vcf-sort-index.sh . *.selectFERAL.renamesamples-job*.vcf.gz
```

## Keep only biallelic sites.

```
#Submit job (~3h).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-10:00:00 \
    -J vcf-biallelic-cc5k-feral \
    vcf-biallelic.sh . *.renamesamples.sort-job*.vcf.gz
```

## Phase .vcf.gz file using SHAPEIT5.

```
#Submit job (requires file haploidchrlist.txt) (1d15h). 
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p node \
    -n 1 \
    -C mem256GB \
    -t 03-00:00:00 \
    -J vcf-phase-shapeit5-cc5k-feral \
    vcf-phase-shapeit5.sh . *.renamesamples.sort.biallelic-job*.vcf.gz haploidchrlist.txt
```

## Split .vcf.gz file by chromosome.

### Unphased VCF file.

```
#Submit job (~3h).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-20:00:00 \
    -J vcf-split-cc5k-feral \
    vcf-split.sh . *.renamesamples.sort.biallelic-job*.vcf.gz
```

### Phased VCF file.

```
#Submit job (~1h).
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p core \
    -n 1 \
    -C mem128GB \
    -t 00-03:00:00 \
    -J vcf-split-cc5k-feral \
    vcf-split.sh . *.biallelic.phase-job*.vcf.gz
```

## Split .vcf.gz file by geographic group (Bermuda, Big Island, Kauai, Oahu, Bermuda+Kauai).

### Unphased VCF files.

#### Define groups of samples.

```
#Define groups of samples.
INPUTVCF=$(find *.biallelic-job*.vcf.gz -maxdepth 0)
BERSAMPLES=$(bcftools query -l ${INPUTVCF} | grep BDA | paste -sd,)
BISAMPLES=$(bcftools query -l ${INPUTVCF} | grep BI | paste -sd,)
KAUSAMPLES=$(bcftools query -l ${INPUTVCF} | grep ERC | paste -sd,)
OAHSAMPLES=$(bcftools query -l ${INPUTVCF} | grep -v 'BDA\|ERC\|BI' | paste -sd,)
BERKAUSAMPLES=$(bcftools query -l ${INPUTVCF} | grep 'BDA\|ERC' | paste -sd,)
```

#### Submit jobs.

```
#Submit jobs (~3h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    SAMPLELIST=$(echo "${GROUPCODE}SAMPLES")
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-12:00:00 \
        -J vcf-selectsamples-cc5k-${GROUPCODE} \
        vcf-selectsamples.sh . ${INPUTVCF} ${!SAMPLELIST} ${GROUPCODE} 
done
```

### Phased VCF files.

#### Define groups of samples.

```
#Define groups of samples.
INPUTVCF=$(find *.biallelic.phase-job*.vcf.gz -maxdepth 0)
BERSAMPLES=$(bcftools query -l ${INPUTVCF} | grep BDA | paste -sd,)
BISAMPLES=$(bcftools query -l ${INPUTVCF} | grep BI | paste -sd,)
KAUSAMPLES=$(bcftools query -l ${INPUTVCF} | grep ERC | paste -sd,)
OAHSAMPLES=$(bcftools query -l ${INPUTVCF} | grep -v 'BDA\|ERC\|BI' | paste -sd,)
BERKAUSAMPLES=$(bcftools query -l ${INPUTVCF} | grep 'BDA\|ERC' | paste -sd,)
```

#### Submit jobs.

```
#Submit jobs (~1h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    SAMPLELIST=$(echo "${GROUPCODE}SAMPLES")
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-06:00:00 \
        -J vcf-selectsamples-cc5k-${GROUPCODE} \
        vcf-selectsamples.sh . ${INPUTVCF} ${!SAMPLELIST} ${GROUPCODE} 
done
```

## Keep only biallelic sites in grouped .vcf.gz files.

### Unphased VCF files.

```
#Submit jobs (~3h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-10:00:00 \
        -J vcf-biallelic-cc5k-${GROUPCODE} \
        vcf-biallelic.sh . $(find *.biallelic.select${GROUPCODE}-job*.vcf.gz -maxdepth 0)
done
```

### Phased VCF files.

```
#Submit jobs (~30min).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J vcf-biallelic-cc5k-${GROUPCODE} \
        vcf-biallelic.sh . $(find *.biallelic.phase.select${GROUPCODE}-job*.vcf.gz -maxdepth 0)
done
```

## Split grouped .vcf.gz files by chromosome.

### Unphased VCF files.

#### Submit jobs.

```
#Submit jobs (~3h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-05:00:00 \
        -J vcf-split-cc5k-${GROUPCODE} \
        vcf-split.sh . $(find *.biallelic.select${GROUPCODE}.biallelic.sort-job*.vcf.gz -maxdepth 0)
done
```

#### Output file control.

```
#Check number of files created.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of VCF files: $(ls ${GROUPDIR}/*.vcf.gz | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#If files are missing or empty, check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job.
```

### Phased VCF files.

#### Submit jobs.

```
#Submit jobs (~30min).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-05:00:00 \
        -J vcf-split-cc5k-${GROUPCODE} \
        vcf-split.sh . $(find *.biallelic.phase.select${GROUPCODE}.biallelic-job*.vcf.gz -maxdepth 0)
done
```

#### Output file control.

```
#Check number of files created.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of VCF files: $(ls ${GROUPDIR}/*.vcf.gz | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job.
```

## Run SweepFinder2.

### Create output directory.

```
#Create SweepFinder2 output directory.
mkdir-stdout.sh . sweepfinder2.files
```

### Produce SweepFinder2 input files (.SF2input).

#### Unphased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.SF2format")
    mkdir-stdout.sh sweepfinder2.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~10min).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    OUTPUTDIR=$(find sweepfinder2.files-job*/*.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    find ${GROUPDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 1 \
            -C mem128GB \
            -t 00-05:00:00 \
            -J vcf-SF2input-${GROUPCODE}-${F##*/} \
            vcf-SF2input.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
cd sweepfinder2.files-job*

#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of .SF2input files: $(ls ${GROUPDIR}/*.SF2input | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/*.SF2input | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

### Produce SweepFinder2 site frequency spectrum files (.SF2sfs).

#### Unphased VCF files.

##### Submit jobs.

```
#Submit jobs (~10min).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-01:00:00 \
        -J SF2input-SF2sfs-${GROUPCODE} \
        SF2input-SF2sfs.sh . ${GROUPDIR}
done
```

##### Output file control.

```
#Count output files.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    echo "-----> GROUP ${GROUPCODE}:"
    echo "Number of .concatSF2input files: $(ls *.biallelic.select${GROUPCODE}.biallelic.*.concatSF2input | wc -l)"
    echo "Number of .SF2sfs files: $(ls *.biallelic.select${GROUPCODE}.biallelic.*.SF2sfs | wc -l)"
done

#Check file content (count number of lines in each file).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPFILE1=$(ls *.biallelic.select${GROUPCODE}.biallelic.*.concatSF2input)
    GROUPFILE2=$(ls *.biallelic.select${GROUPCODE}.biallelic.*.SF2sfs)
    echo "-----> GROUP ${GROUPCODE}:"
    echo "Number of lines: "
    echo "${GROUPFILE1}: $(cat ${GROUPFILE1} | wc -l)"
    echo "${GROUPFILE2}: $(cat ${GROUPFILE2} | wc -l)"
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job.
```

### Run SweepFinder2 without the pre-computed site frequency spectrum files (.SF2sfs).

#### Unphased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.SF2withoutSFSoutput")
    mkdir-stdout.sh . ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~1d3h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    OUTPUTDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format.SF2withoutSFSoutput-job* -maxdepth 0)
    find ${GROUPDIR}/*.SF2input -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 1 \
            -C mem128GB \
            -t 02-00:00:00 \
            -J SF2input-SF2output-${GROUPCODE}-${F##*/} \
            SF2input-SF2output.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2withoutSFSoutput-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of .SF2output files: $(ls ${GROUPDIR}/*.SF2output | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2withoutSFSoutput-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/*.SF2output | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

##### Format output data.

```
#Create directory.
mkdir-stdout.sh . sweepfinder2.results

#Submit jobs (~5min).
OUTPUTDIR=sweepfinder2.results-job*
find *SF2withoutSFSoutput-job* -maxdepth 0 | while read D ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J formatdata-SweepFinder2-${D##*/} \
        Rscript.sh formatdata-SweepFinder2.R ${OUTPUTDIR} ${D}
done

#Count output file lines.
countlines-stdout.sh sweepfinder2.results-job*/*.csv
```

### Run SweepFinder2 with the pre-computed site frequency spectrum files (.SF2sfs).

#### Unphased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.SF2withSFSoutput")
    mkdir-stdout.sh . ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~17h).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format-job* -maxdepth 0)
    OUTPUTDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2format.SF2withSFSoutput-job* -maxdepth 0)
    SFSFILE=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SF2format.SF2sfs-job*.SF2sfs)
    find ${GROUPDIR}/*.SF2input -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 1 \
            -C mem128GB \
            -t 02-00:00:00 \
            -J SF2input-SF2output-${GROUPCODE}-${F##*/} \
            SF2input-SF2output.sh ${OUTPUTDIR} ${F} ${SFSFILE}
    done
done
```

##### Output file control.

```
#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2withSFSoutput-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of .SF2output files: $(ls ${GROUPDIR}/*.SF2output | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in BER BI KAU OAH BERKAU ; do
    GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.*.SF2withSFSoutput-job* -maxdepth 0)
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/*.SF2output | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

##### Format output data.

```
#Submit jobs (~5min).
OUTPUTDIR=sweepfinder2.results-job*
find *SF2withSFSoutput-job* -maxdepth 0 | while read D ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J formatdata-SweepFinder2-${D##*/} \
        Rscript.sh formatdata-SweepFinder2.R ${OUTPUTDIR} ${D}
done

#Count output file lines.
countlines-stdout.sh *.csv
```

## Run SweeD.

### Create output directory.

```
#Create SweeD output directory.
cd ..
mkdir-stdout.sh . sweed.files
```

### Run SweeD to produce .SweeDoutput output files.

#### Unphased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    fi
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.SweeDoutput")
    mkdir-stdout.sh sweed.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~12h).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find sweed.files-job*/*.renamesamples.sort.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find sweed.files-job*/*.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    find ${GROUPDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 1 \
            -C mem128GB \
            -t 01-00:00:00 \
            -J vcf-SweeD-${GROUPCODE}-${F##*/} \
            vcf-SweeD.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
cd sweed.files-job*

#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of SweeD_Report files: $(ls ${GROUPDIR}/SweeD_Report* | wc -l)"
    echo "Number of SweeD_Info files: $(ls ${GROUPDIR}/SweeD_Info* | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/SweeD_Report* | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

#### Phased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr-job* -maxdepth 0)
    fi
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.SweeDoutput")
    mkdir-stdout.sh sweed.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~9h).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find sweed.files-job*/*.biallelic.phase.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find sweed.files-job*/*.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    find ${GROUPDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 1 \
            -C mem128GB \
            -t 01-00:00:00 \
            -J vcf-SweeD-${GROUPCODE}-${F##*/} \
            vcf-SweeD.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
cd sweed.files-job*

#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of SweeD_Report files: $(ls ${GROUPDIR}/SweeD_Report* | wc -l)"
    echo "Number of SweeD_Info files: $(ls ${GROUPDIR}/SweeD_Info* | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr.SweeDoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.SweeDoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/SweeD_Report* | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

#### Format output data.

```
#Create directory.
mkdir-stdout.sh . sweed.results

#Submit jobs (~5min).
OUTPUTDIR=sweed.results-job*
find *SweeDoutput-job* -maxdepth 0 | while read D ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J formatdata-SweeD-${D##*/} \
        Rscript.sh formatdata-SweeD.R ${OUTPUTDIR} ${D}
done

#Count output file lines.
countlines-stdout.sh *.csv
```

## Run OmegaPlus.

### Create output directory.

```
#Create OmegaPlus output directory.
cd ..
mkdir-stdout.sh . omegaplus.files
```

### Run OmegaPlus to produce OmegaPlus_Info and OmegaPlus_Report output files.

#### Unphased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    fi
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.OmegaPlusoutput")
    mkdir-stdout.sh omegaplus.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~3h).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find omegaplus.files-job*/*.renamesamples.sort.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find omegaplus.files-job*/*.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    find ${GROUPDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 10 \
            -C mem256GB \
            -t 02-00:00:00 \
            -J vcf-OmegaPlus-${GROUPCODE}-${F##*/} \
            vcf-OmegaPlus.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
cd omegaplus.files-job*

#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of OmegaPlus_Report files: $(ls ${GROUPDIR}/OmegaPlus_Report* | wc -l)"
    echo "Number of OmegaPlus_Info files: $(ls ${GROUPDIR}/OmegaPlus_Info* | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.renamesamples.sort.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/OmegaPlus_Report* | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

#### Phased VCF files.

##### Create directories.

```
#Create directories.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr-job* -maxdepth 0)
    fi
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.OmegaPlusoutput")
    mkdir-stdout.sh omegaplus.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```
#Submit jobs (~1h30min).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find omegaplus.files-job*/*.biallelic.phase.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find omegaplus.files-job*/*.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    find ${GROUPDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p core \
            -n 10 \
            -C mem256GB \
            -t 01-00:00:00 \
            -J vcf-OmegaPlus-${GROUPCODE}-${F##*/} \
            vcf-OmegaPlus.sh ${OUTPUTDIR} ${F}
    done
done
```

##### Output file control.

```
cd omegaplus.files-job*

#Count output files.
countfiles-stdout.sh . 2

for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    echo "Number of OmegaPlus_Report files: $(ls ${GROUPDIR}/OmegaPlus_Report* | wc -l)"
    echo "Number of OmegaPlus_Info files: $(ls ${GROUPDIR}/OmegaPlus_Info* | wc -l)"
    echo "Number of .script files: $(ls ${GROUPDIR}/*.script | wc -l)"
    echo "Number of .slurm files: $(ls ${GROUPDIR}/*.slurm | wc -l)"
    echo "Total size of directory: $(sizeof ${GROUPDIR} | grep "total" | cut -f1)"
done

#Check file content in each directory (count number of lines in each file).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find *.biallelic.phase.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    else 
        GROUPDIR=$(find *.biallelic.phase.select${GROUPCODE}.biallelic.splitbychr.OmegaPlusoutput-job* -maxdepth 0)
    fi
    echo "-----> GROUP ${GROUPCODE}: ${GROUPDIR}"
    ls ${GROUPDIR}/OmegaPlus_Report* | while read F ; do
        echo "${F##*/}: $(cat ${F} | wc -l)"
    done
done

#If files are missing or empty (0/1 lines), check the .slurm file for error messages, adjust the sbatch parameters (time limit, memory allocation, number of cores, partition) and resubmit the job for those specific chromosomes.
```

#### Format output data.

```
#Create directory.
mkdir-stdout.sh . omegaplus.results

#Submit jobs (~5min).
OUTPUTDIR=omegaplus.results-job*
find *OmegaPlusoutput-job* -maxdepth 0 | while read D ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J formatdata-OmegaPlus-${D##*/} \
        Rscript.sh formatdata-OmegaPlus.R ${OUTPUTDIR} ${D}
done

#Count output file lines.
countlines-stdout.sh *.csv
```

## Run Rehh.

### Create output directory.

```
#Create rehh output directory.
cd ..
mkdir-stdout.sh . rehh.files
```

### Unphased VCF files.

####  Produce iHH data output files.

##### Create directories.

```
#Create directories.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
    else 
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr-job* -maxdepth 0)
    fi
    OUTPUTDIRPREFIX=$(echo "${GROUPDIR}" | sed 's/-job[0-9].*$//')
    OUTPUTDIR=$(echo "${OUTPUTDIRPREFIX}.iHH")
    mkdir-stdout.sh rehh.files-job* ${OUTPUTDIR}
done
```

##### Submit jobs.

```



????? HERE



#Submit jobs (~2d).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        INPUTDIR=$(find cc5k.select${GROUPCODE}.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find rehh.files-job*/cc5k.select${GROUPCODE}.renamesamples.sort.biallelic.splitbychr.iHH-job* -maxdepth 0)
    else 
        INPUTDIR=$(find *.biallelic.select${GROUPCODE}.*.splitbychr-job* -maxdepth 0)
        OUTPUTDIR=$(find rehh.files-job*/*.biallelic.select${GROUPCODE}.*.iHH-job* -maxdepth 0)
    fi
    find ${INPUTDIR}/*.vcf.gz -maxdepth 0 | while read F ; do
        PHASING=FALSE
        sbatch \
            -A ${PROJECT_ID_DW} \
            -o ${MYSLURMFILE} \
            -p node \
            -n 1 \
            -C mem256GB \
            -t 02-00:00:00 \
            -J Rscript-Rehh-iHH-${GROUPCODE}-${F##*/} \
            Rscript.sh Rehh-vcf-iHH.R ${OUTPUTDIR} ${F} ${PHASING}
    done
done



?????

GROUPCODE=FERAL
F=/crex/proj/sllstore2017078/willians-workingdir/output/cc5k-feralsamples-job20241025145945/cc5k.selectFERAL.renamesamples.sort.biallelic.splitbychr-job51106787/1.*.vcf.gz
OUTPUTDIR=/crex/proj/sllstore2017078/willians-workingdir/output/cc5k-feralsamples-job20241025145945/rehh.files-job20241029091037/cc5k.selectFERAL.renamesamples.sort.biallelic.splitbychr.iHH-job20241029091517
PHASING=FALSE
sbatch \
    -A ${PROJECT_ID_DW} \
    -o ${MYSLURMFILE} \
    -p node \
    -n 1 \
    -C mem256GB \
    -t 02-00:00:00 \
    -J Rscript-Rehh-iHH-${GROUPCODE}-${F##*/} \
    Rscript.sh Rehh-vcf-iHH.R ${OUTPUTDIR} ${F} ${PHASING}




echo "${GROUPCODE}
    ${GROUPDIR}
    ${OUTPUTDIR}"






?????

#Run an interactive job.
salloc \
    -A ${PROJECT_ID_WTAFS} \
    -p shared \
    -t 05:00:00 \
    --mem=10GB
    
GROUPCODE=FERAL
INPUTDIR=$(find cc5k.select${GROUPCODE}.renamesamples.sort.biallelic.splitbychr-job* -maxdepth 0)
OUTPUTDIR=$(find rehh.files-job*/cc5k.select${GROUPCODE}.renamesamples.sort.biallelic.splitbychr.unphasediEHH-job* -maxdepth 0)
echo "${GROUPCODE}
${INPUTDIR}
${OUTPUTDIR}"
find ${INPUTDIR}/1.*.vcf.gz -maxdepth 0 | while read F ; do
        echo ${F}
        #PHASING=FALSE
        #Rscript.sh Rehh-vcf-iEHH.R ${OUTPUTDIR} ${F} ${PHASING}
done
```

##### Output file control.

```


?????


#Count output files.
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.splitbychr.iHH-job* -maxdepth 0)
    else 
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.iHH-job* -maxdepth 0)
    fi
    echo "${GROUPDIR}: 
-----> $(ls ${GROUPDIR}/*iHH*.rds | wc -l) *iHH*.rds files."
done

#Check file content in ech directory (show file size).
for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.splitbychr.iHH-job* -maxdepth 0)
    else 
        GROUPDIR=$(find cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.iHH-job* -maxdepth 0)
    fi
    echo "${GROUPDIR}:"
    ls ${GROUPDIR}/*iHH*.rds | while read F ; do
        echo "-----> ${F##*/}: $(du -hc ${F} | head -n1 | cut -f1)"
    done
done
```

#### Produce iHS data output files.

##### Submit jobs.

```

```

##### Output file control.

```

```

### Unphased VCF files.

####  Produce iHH data output files.

##### Create directories.

```

```

##### Submit jobs.

```

```

##### Output file control.

```

```

#### Produce iHS data output files.

##### Submit jobs.

```

```

##### Output file control.

```

```

## Create plots and get lists of genes.

### Create directory.

```
#Create plots output directory.
cd ..
mkdir-stdout.sh . plots
```

### SweepFinder2.

```
#Submit jobs (~5min).
OUTPUTDIR=plots-job*

for GROUPCODE in FERAL BER BI KAU OAH BERKAU ; do
    if [[ ${GROUPCODE} == "FERAL" ]] ; then
        FILE_SWEEPFINDER2=NA
        FILE_SWEED=$(find sweed.files-job*/sweed.results-job*/cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SweeDformat.SweeDoutput.SweeDdata-job*.csv -maxdepth 0)
        FILE_OMEGAPLUS=$(find omegaplus.files-job*/omegaplus.results-job*/cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.OmegaPlusoutput.OmegaPlusdata-job*.csv -maxdepth 0)
        FILE_REHH=
    else 
        FILE_SWEEPFINDER2=$(find sweepfinder2.files-job*/sweepfinder2.results-job*/cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SF2format.SF2output.SF2data-job*.csv -maxdepth 0)
        FILE_SWEED=$(find sweed.files-job*/sweed.results-job*/cc5k.selectFERAL.renamesamples.sort.biallelic.select${GROUPCODE}.biallelic.sort.splitbychr.SweeDoutput.SweeDdata-job*.csv -maxdepth 0)
    fi

    echo "-----> ${GROUPCODE}:
    ${FILE_SWEEPFINDER2}
    ${FILE_SWEED}
    ${FILE_OMEGAPLUS}
    ${FILE_REHH}"
done

    





find sweepfinder2.files-job*/sweepfinder2.results-job*/*.SF2data-job*.csv -maxdepth 0 | while read F ; do
    sbatch \
        -A ${PROJECT_ID_DW} \
        -o ${MYSLURMFILE} \
        -p core \
        -n 1 \
        -C mem128GB \
        -t 00-03:00:00 \
        -J plot-selectivesweep-${GROUPCODE} \
        Rscript.sh plot-selectivesweep-analysis ${OUTPUTDIR} ${GROUPCODE} ${FILE_SWEEPFINDER2} ${FILE_SWEED} ${FILE_OMEGAPLUS} ${FILE_REHH}
done

#Count output file lines.
countlines-stdout.sh *.csv
```


```




?????




```






