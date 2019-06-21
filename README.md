# Testing the squiggleSV branch on Nanopolish
This is a simple guide on how to install relevant software and branches in order to use the squiggleSV branch on Nanopolish, in order to filter or accept structural variant (SV) candidates using the sequence-to-squiggle likelihood function provided by Nanopolish and event to basecall index map provided by a modified Scrappie branch. Two directories are provided. "test" and "results". We will use the raw fast5s and the candidate SV file in the folder "test" and aim to achieve the final result file test.sniffles.variant.likelihoods.csv.

### Dependencies
- minimap2 version 2.14-r883 or higher.
- samtools 1.9 or higher.
- GNU parallel


- Nanopolish is compiled with gcc version 5.4.0. 
- Scrappie is compiled with cmake version 3.10.2

### Required files
- Human reference genome hg38. (The candidate variant example provided is using hg38 coordinates.)

### Install nanopolish on branch squigglesv
```
git clone --recursive https://github.com/DecodeGenetics/nanopolish.git
cd nanopolish
git checkout squigglesv
make
```
Tested with gcc 5.4.0 on RedHat.



### Install scrappie on branch v1.3.0.events
```
git clone --recursive https://github.com/DecodeGenetics/scrappie.git
cd scrappie
git checkout v1.3.0.events
mkdir build-test
cd build-test
cmake -DCMAKE_BUILD_TYPE=DEBUG -DOPENBLAS_ROOT=/home/dorukb/ONT_SV_analysis/scrappie/openblas_include/ ..
```
Currently providing the openblas headers, if required. Tested with cmake version 3.10.2 on RedHat


# Testing
After installing required software, you can start the sample run by entering the directory "test". The aim is to generate the final result given in "results/test.sniffles.variant.likelihoods.csv"
```
cd test
```

### Run the modified Scrappie to get the event to basecall index maps.
Define executable locations. 
This basecalls the reads using Scrappie, creating a reads.fasta using the fast5 files, alongside the .fast5.events (event tables, i.e. event index to basecall index maps) per fast5 file under the same directory fast5s reside.
```
NUM_THREADS=4
PARALLEL_EXE=#location of GNU  parallel executable
SCRAPPIE_EXE=#location of scrappie exevcutable built above.
cat reads.fast5.paths | ${PARALLEL_EXE} --jobs ${NUM_THREADS} ${SCRAPPIE_EXE} events -f FASTA --threads 1 > reads.fasta
```

### Create the nanopolish index on the reads.fasta
Define executable locations and run command below to generate required nanopolish index files.
```
NANOPOLISH_EXE=#location of nanopolish built above
${NANOPOLISH_EXE} index -d fast5/ reads.fasta
```


### Map the reads using on human reference genome hg38
Generate the sorted bam files using minimap2 and samtools of the reads.fasta sequences.
```
MINIMAP=#location of minimap2
REFERENCE=#location of reference fasta file
${MINIMAP} --MD -Y -a -x map-ont -t ${NUM_THREADS} ${REFERENCE} reads.fasta | ${SAMTOOLS} view -@ ${NUM_THREADS} -bS - > reads.bam
${SAMTOOLS} sort -o reads.sorted.bam reads.bam
${SAMTOOLS} index reads.sorted.bam
```


### Run Nanopolish SV squiggle filtering using Scrappie basecalls "reads.fasta" and respective bam file, on an SV candidate in test.sniffles.variant.vcf
Currently using flank size of 500 bps.
We did not performs tests with more than 1 threads, using GNU parallel on smaller sizes input vcf files, when needed. 
Currently using the entire chromosome as window, with variants from the window chromosome on the input vcf file.
```
${NANOPOLISH_EXE} test-vcf-variants --genome ${REFERENCE} --reads reads.fasta --bam reads.sorted.bam --ploidy 2 -t 1 -f 500 -w chr1 --candidates test.sniffles.variant.vcf -o test.sniffles.variant.likelihoods.csv
```

# Notes
- Last column of test.sniffles.variant.likelihoods.csv called "diff" has the likelihood difference as in L(ALT_SEQ) - L(REF_SEQ). 
- A negative likelihood difference for the given read means that the ref allele is more likely, whereas a positive difference means that the alt allele is more likely.
- If possible two likelihood values are given per read, one for the RIGHT and one for the LEFT flank, as the ref and alt alleles are tested from both flanks, with 500 bps as flank size on both sides.
- Currently, we are accepting SVs that breach a likelihood diff of +50, with at least 3 reads, using either (LEFT/RIGHT) flank.

