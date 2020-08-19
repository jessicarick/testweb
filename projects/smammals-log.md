[Home](https://jessicarick.github.io/testweb) | [CV](../cv/cv.html) | [Publications](../publications/pubs.html) | [Research](../research/research.html) | [Teaching](../teaching/teaching.html) | [Software](../software/tools.html) | [Projects](../projects/projects.html)

# UHURU Small Mammal Genetics
## Log of Methods
[jump to most recent](#recent)

### STARTING OVER -- now with more data
#### July 2020

Step 1: Barcode parsing

* using old barcode script for 1,2,3; new barcode script for 4,5,6 \
* took < 2 days for 1,2,3; timeout for 4,5,6 \
* took 2 days, 20 hours for 4,6 \

```sh
# UHURU1
Good mids count: 216880875
Bad mids count: 10370274 (4.8%)
Number of seqs with potential MSE adapter in seq: 18036770
Seqs that were too short after removing MSE and beyond: 3703

# UHURU2
Good mids count: 208777637
Bad mids count: 9467255 (4.5%)
Number of seqs with potential MSE adapter in seq: 18856287
Seqs that were too short after removing MSE and beyond: 5192

# UHURU3
Good mids count: 213651963
Bad mids count: 9623340 (4.5%)
Number of seqs with potential MSE adapter in seq: 19246222
Seqs that were too short after removing MSE and beyond: 3445

# UHURU4 
Good mids count: 176225154
Bad mids count: 26225207 (12.9%)
Number of seqs with potential MSE adapter in seq: 229962
Seqs that were too short after removing MSE and beyond: 783

# UHURU5 
Good mids count: 
Bad mids count:
Number of seqs with potential MSE adapter in seq: 
Seqs that were too short after removing MSE and beyond: 

# UHURU6
Good mids count: 199599451
Bad mids count: 29505980 (12.9%)
Number of seqs with potential MSE adapter in seq: 239013
Seqs that were too short after removing MSE and beyond: 858
```

#### August 2020 
Step 2: Barcode trimming

Now that I finnnnnnallllly have all of the barcodes parsed, split, and concatenated (from all the different libraries), I need to trim the reads down to 85bp to make Stacks happy. This time, I'm going to use a combo of ```fastx_toolkit``` and ```seqkit```.

```sh
module load fastx_toolkit
module load seqkit
module load parallel

parallel -j 16 fastx_trimmer -f 1 -l 85 -Q33 -i {} -o {.}.trim.fastq ::: *[0-9].fastq
parallel seqkit seq {} -m 85 -o ../../04-all_samples/{.}.trim.fq.gz ::: *[0-9].trim.fastq
```

Then, I moved each trimmed fastq into the appropriate species' directory. Some files are throwing an error-- maybe something went wrong with the splitting/concatenating of these? *I'll have to figure out which they are and investigate.*

Step 3: Stacks workflow

From here, I used the ```run_stacks.sh``` script for each population, which runs (in sequence) ```ustacks```, ```cstacks```, ```sstacks```, ```tsv2bam```, ```gstacks```, and ```populations```. Results from ```populations``` module:

```sh
## TAHA
Removed 171593 loci that did not pass sample/population constraints from 375129 loci.
Kept 203536 loci, composed of 17316679 sites; 1044 of those sites were filtered, 121655 variant sites remained.
Mean genotyped sites per locus: 85.08bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 4.2107 samples per locus; pi: 0.32013; all/variant/polymorphic sites: 17316622/121655/91732; private alleles: 43921
  N: 2.4155 samples per locus; pi: 0.30441; all/variant/polymorphic sites: 17316622/121655/69204; private alleles: 21393

## ELRU

Genotyped 771664 loci:
  effective per-sample coverage: mean=8.5x, stdev=2.0x, min=5.9x, max=13.8x
  mean number of sites per locus: 85.0
  a consistent phasing was found for 165507 of out 189195 (87.5%) diploid loci needing phasing

Removed 381300 loci that did not pass sample/population constraints from 771664 loci.
Kept 390364 loci, composed of 33216681 sites; 2245 of those sites were filtered, 406799 variant sites remained.
Mean genotyped sites per locus: 85.09bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 4.7968 samples per locus; pi: 0.29472; all/variant/polymorphic sites: 25852749/346427/244377; private alleles: 70368
  N: 2.663 samples per locus; pi: 0.26922; all/variant/polymorphic sites: 30770026/384475/199979; private alleles: 55363
  S: 3.1429 samples per locus; pi: 0.27721; all/variant/polymorphic sites: 31115608/385001/214027; private alleles: 55145

## AEHI

Genotyped 924553 loci:
  effective per-sample coverage: mean=35.2x, stdev=38.0x, min=6.4x, max=256.8x
  mean number of sites per locus: 85.0
  a consistent phasing was found for 39999 of out 53646 (74.6%) diploid loci needing phasing

Removed 866065 loci that did not pass sample/population constraints from 924553 loci.
Kept 58488 loci, composed of 4980048 sites; 1994 of those sites were filtered, 56485 variant sites remained.
Mean genotyped sites per locus: 85.15bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 3.531 samples per locus; pi: 0.24091; all/variant/polymorphic sites: 4855649/54115/28047; private alleles: 4868
  N: 5.9015 samples per locus; pi: 0.21214; all/variant/polymorphic sites: 1162507/22752/13125; private alleles: 3087
  S: 12.966 samples per locus; pi: 0.26049; all/variant/polymorphic sites: 4723322/53624/47180; private alleles: 21358
```
It seems that some of these didn't work out very well, so I'm going to go back through and remove individuals that have low coverage (< 50000 reads to start) and re-run the ```cstacks``` through ```populations``` modules. For each species, I did the following:

```sh
for file in *.fq.gz
	do echo $file
	reads=`zgrep '^@' $file | wc -l`
	echo "$file $reads" >> ../reads_per_ind_AEHI.txt
done

module load r
R
```
```r
AEHI <- read.table("reads_per_ind_AEHI.txt", header=F)
colnames(AEHI) <- c("ind","reads")

summary(AEHI$reads)
length(AEHI$ind[AEHI$reads < 20000])
write.table(AEHI$ind[AEHI$reads < 20000], "lowcov_AEHI", 
	col.names=F, row.names=F, quote=F)
## AEHI
 Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
      1  103276  528904  740004 1133816 4534199
 15 # less than 20k reads
 ## ELRU
    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
 117551 1222813 1582039 1708879 1965295 5652582
 0
## SAME
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
   1093 1080934 2074135 2074894 3174265 4726248
 3
## GERO
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
   1091  383796 1071208 1348931 2071609 5858744
 8
## TAHA
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
    165  966798 1318086 1436825 1963732 4086397
 4
```

After removing individuals with fewer than 50k reads, I ran the sstacks-populations steps for all four species again.

```sh
# AEHI

Removed 836692 loci that did not pass sample/population constraints from 920951 loci.
Kept 84259 loci, composed of 7173027 sites; 2006 of those sites were filtered, 73621 variant sites remained.
Mean genotyped sites per locus: 85.13bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 3.3548 samples per locus; pi: 0.24952; all/variant/polymorphic sites: 6711468/68129/35594; private alleles: 6994
  N: 4.6018 samples per locus; pi: 0.23599; all/variant/polymorphic sites: 3063727/40707/22648; private alleles: 5426
  S: 11.78 samples per locus; pi: 0.26676; all/variant/polymorphic sites: 6448224/67514/58585; private alleles: 25538

# TAHA

Removed 139783 loci that did not pass sample/population constraints from 332696 loci.
Kept 192913 loci, composed of 16410871 sites; 928 of those sites were filtered, 99125 variant sites remained.
Mean genotyped sites per locus: 85.07bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  N: 2.4911 samples per locus; pi: 0.30759; all/variant/polymorphic sites: 16410825/99125/57910; private alleles: 14495
  C: 4.7358 samples per locus; pi: 0.32593; all/variant/polymorphic sites: 16410825/99125/78004; private alleles: 34589

# SAME

Removed 921240 loci that did not pass sample/population constraints from 1284600 loci.
Kept 363360 loci, composed of 30926106 sites; 2532 of those sites were filtered, 273227 variant sites remained.
Mean genotyped sites per locus: 85.11bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 7.2428 samples per locus; pi: 0.22604; all/variant/polymorphic sites: 28949904/257672/161307; private alleles: 24344
  N: 4.0304 samples per locus; pi: 0.21547; all/variant/polymorphic sites: 25558538/233723/114424; private alleles: 13621
  S: 15.609 samples per locus; pi: 0.2462; all/variant/polymorphic sites: 30178865/267872/229620; private alleles: 74261

# GERO

Removed 948377 loci that did not pass sample/population constraints from 1227935 loci.
Kept 279558 loci, composed of 23820166 sites; 2677 of those sites were filtered, 275714 variant sites remained.
Mean genotyped sites per locus: 85.21bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  C: 15.161 samples per locus; pi: 0.21688; all/variant/polymorphic sites: 21901957/257043/194755; private alleles: 29552
  N: 10.667 samples per locus; pi: 0.21309; all/variant/polymorphic sites: 22238852/260269/173990; private alleles: 21975
  S: 18.172 samples per locus; pi: 0.21668; all/variant/polymorphic sites: 22502712/264402/206734; private alleles: 37665
  ```
  
#### August 18, 2020 {#recent}

Now, I'm working with AEHI and GERO to try and improve the Stacks results. After filtering in R, AEHI was left with 54 individuals (out of 9,286) and 9,286 SNPs; GERO had 149 individuals (out of 220) and only 5,097 SNPs. I changed the parameters in ```ustacks``` to ```-m 3 -M 5 -N 7 --model_type bounded --bound_high 0.05``` based off of some recommendations that I've seen-- we'll see how it goes! I also made it so that all individuals belong to the same population, instead of specifying C/N/S, and changed to ```-p 1 -r 0.1```.

Here are the results from ```populations```:

```sh
# AEHI
Removed 1229518 loci that did not pass sample/population constraints from 1279334 loci.
Kept 49816 loci, composed of 4242349 sites; 2272 of those sites were filtered, 60891 variant sites remained.
Mean genotyped sites per locus: 85.16bp (stderr 0.00).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  AEHI: 30.408 samples per locus; pi: 0.23675; all/variant/polymorphic sites: 4242341/60891/60891; private alleles: 0

# GERO


```