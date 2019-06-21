### Install nanopolish on branch squigglesv
git clone --recursive https://github.com/DecodeGenetics/nanopolish.git
cd nanopolish
git checkout squigglesv
# Tested with gcc 5.4.0 on RedHat.
make


### Install scrappie on branch v1.3.0.events
git clone --recursive https://github.com/DecodeGenetics/scrappie.git
cd scrappie
git checkout v1.3.0.events
mkdir build-test
cd build-test
# providing the openblas headers on cmake, if required. Tested with cmake version 3.10.2 on RedHat
cmake -DCMAKE_BUILD_TYPE=DEBUG -DOPENBLAS_ROOT=/home/dorukb/ONT_SV_analysis/scrappie/openblas_include/ ..



### Create file reads.fast5.paths containing full paths of the fast5 files that are going to be used, separated by newline.
# Define executable locations
NUM_THREADS=4
PARALLEL_EXE=#location of gnu parallel executable
SCRAPPIE_EXE=#location of scrappie exevcutable built above.
# Create the fasta file provided by Scrappie, alongside the .fast5.events event tables per fast5 file under the same directory fast5s reside.
cat reads.fast5.paths | ${PARALLEL_EXE} --jobs ${NUM_THREADS} ${SCRAPPIE_EXE} events -f FASTA --threads 1 > reads.fasta


### Create nanopolish index on the reads.fasta
# Define executable.
NANOPOLISH_EXE=#location of nanopolish built above
# Create index using the Scrappie basecalled reads.
${NANOPOLISH_EXE} index -d /home/dorukb/ONT_SV_analysis/test_run/fast5/ reads.fasta


### Map the reads using minimap2
MINIMAP=#location of minimap2
REFERENCE=#location of reference fasta file
${MINIMAP} --MD -Y -a -x map-ont -t ${NUM_THREADS} ${REFERENCE} reads.fasta | ${SAMTOOLS} view -@ ${NUM_THREADS} -bS - > reads.bam
# Samtools sort/index the reads.bam
${SAMTOOLS} sort -o reads.sorted.bam reads.bam
${SAMTOOLS} index reads.sorted.bam


### Run Nanopolish SV squiggle filtering using Scrappie basecalls "reads.fasta" and respective bam file, on an SV candidate in test.sniffles.variant.vcf
# Currently using flank size of 500 bps.
# Did not test with more than 1 threads, currently using it with GNU parallel on smaller sizes input vcf files. 
# Currently using the entire chromosome as window, and only using variants from a single chromosome on the input vcf file.
${NANOPOLISH_EXE} test-vcf-variants --genome ${REFERENCE} --reads reads.fasta --bam reads.sorted.bam --ploidy 2 -t 1 -f 500 -w chr1 --candidates test.sniffles.variant.vcf -o test.sniffles.variant.likelihoods.csv


##### NOTES #####
### Last column of test.sniffles.variant.likelihoods.csv called "diff" has the likelihood difference as in L(ALT_SEQ) - L(REF_SEQ). 
### A negative likelihood difference for the given read means that the ref allele is more likely, whereas a positive difference means that the alt allele is more likely.
### If possible two likelihood values are given per read, one for the RIGHT and one for the LEFT flank, as the ref and alt alleles are tested from both flanks, with 500 bps as flank size on both sides.
### Currently, I am accepting SVs that breach a likelihood diff of +50, with at least 3 reads, using either (LEFT/RIGHT) flank.

