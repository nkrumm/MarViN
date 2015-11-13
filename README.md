# MarViN
Genotype imputation and refinement

See our [pre-print](http://biorxiv.org/content/biorxiv/early/2015/11/12/031484.full.pdf) for a full description of the method.

##License
MarViN source code is provided under the [GPLv3](https://git.illumina.com/rarthur/MarViN/blob/master/LICENSE.txt) license. MarViN includes several third-party packages provided under other open source licenses, please see [COPYRIGHT.txt](https://git.illumina.com/rarthur/MarViN/blob/master/COPYRIGHT.txt) for additional details.

MarViN relies on HTSlib and Eigen. [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page) is a header-only library for matrix algebra released under the MPL2 license - see the link (https://www.mozilla.org/en-US/MPL/2.0/) and [COPYRIGHT.txt](https://git.illumina.com/rarthur/MarViN/blob/master/COPYRIGHT.txt). [HTSlib](http://www.htslib.org/) is a library for efficently parsing vcf files released under the MIT/Expat License - see the link (https://opensource.org/licenses/MIT) and [COPYRIGHT.txt](https://git.illumina.com/rarthur/MarViN/blob/master/COPYRIGHT.txt).
Both Eigen and HTSlib are included with MarViN.

##Installation
You can install MarViN via the following commands:
```
git clone https://github.com/sequencing/MarViN
cd MarViN/
make
```
Which creates the two executables: `marvin` and `marvin_prep`.

##marvin_prep
Set up means and covariances for marvin - only necessary if you have a reference panel - if you want to genotype a large cohort from likelihoods you do not need to run this.

Usage:
```./marvin_prep -f input.vcf.gz -O z -o sites.vcf.gz -om mu.dat -ov sig.dat -ow omega.dat```
Expects input.vcf.gz to contain hard genotypes. Options:

* -f : The input vcf file. Can be valid vcf or bcf, compressed or uncompressed. Should contain at least two samples and must have a GT field with no missing entries for any sample/variant combination.
* -o : marvin\_prep outputs a site only vcf containing the sites kept from the input vcf. Specify the site file name with this argument.
* -O : output format for site only vcf. Compressed BCF (b), uncompressed BCF (u), compressed VCF (z), uncompressed VCF (v). Default is v.
* -om : file name for the vector of allele frequencies.
* -os : file name for the covariance matrix. This is not required by any later routine so can be omitted.
* -ow : file name for the matrix of inverses.
* -ov : file name for the vector of variances.
* -sigma_reg : use sigmoid function of allele frequency in regularization. Can perform better at low allele frequencies. Default regularization (without this option) is `Sigma(i,i) += lambda` with this option it is `Sigma(i,i) += lambda / (1.0 + exp( lambda2 * ( Sigma(i,i) - pct ) ) )`
* -lambda : regularization parameter, changing this has some effect on the results but 0.06 was optimal for most of our tests. Default is 0.06.
* -lambda2 : regularization parameter, controls steepness of sigma regularization. Default is 4.
* -pct : regularization parameter, controls the midpoint of the sigma regularization. Default is 0.2.
* -r : regions chr:start-end. Section of the genome to operate on. 

##marvin
Imputation from genotype likelihoods

Usage:
```./marvin -f input.vcf.gz -O z -o out.vcf.gz```
Expects input_filename.vcf.gz to contain GL or PL field
* I/O
  * -f : the input vcf file. Can be valid vcf or bcf, compressed or uncompressed. 
  * -o : output file name containing the imputed genotypes and dosages.
  * -O : output format. Compressed BCF (b), uncompressed BCF (u), compressed VCF (z), uncompressed VCF (v). Default is v.
  * -Ogl : Add GP and GQ field to output.
  * -r : regions chr:start-end. Section of the genome to operate on.
  * -b : padding to the left and right.
* Panel input
  * -fm : vector of allele frequencies output by marvin_prep (-om output).
  * -fw : matrix of inverses output by marvin_prep (-ow ouput).
  * -fv : vector of variances output by marvin_prep (-ov output).
  * -site : site only vcf output by marvin_prep (-o output).
  * -zm : zero missing rows (assumes 0/0 for missing data)
  * -c : collapse snps|indels|both|all|some|none. Controls how intersection of sample and panel is performed. Similar to bcftools isec. Default none.
* Run pararmeters
  * -max_its : Number of ‘outer’ iterations of marvin (re-estimations of the covariance matrix). Only has an effect when not using panel. Default 5.
  * -inner_its : Number of ‘inner’ iterations of marvin (re-estimations of data with fixed covariance). Default 1.
  * -maxlr : Maximum allowed likelihood ratio. Likelihood ratios larger than specified threshold will set the smaller likelihood to zero.
  * -bias : MarViN works with expected values, when calling hard genotypes if the expected value is less than this parameter it reports hom-ref. If the expected value is between bias and 2-bias it reports het if above 2-bias reports hom-alt. Default 0.5.
  * -EMits : When not using a panel marvin does EM on allele frequencies to compute an initial guess. Specifies how many iterations to perform (fewer is generally better). Default 1.
* Regularization parameters
  * -sigma_reg : use sigmoid function of allele frequency in regularization. Can perform better at low allele frequencies. Default regularization (without this option) is `Sigma(i,i) += lambda` with this option it is `Sigma(i,i) += lambda / (1.0 + exp( lambda2 * ( Sigma(i,i) - pct ) ) )`.
  * -lambda : regularization parameter, changing has some effect on the results but 0.06 was optimal for most of our tests. Default is 0.06.
  * -lambda2 : regularization parameter, controls steepness of sigma regularization. Default is 4.
  * -pct : regularization parameter, controls the midpoint of the sigma regularization. Default is 0.2.

#Examples

MarViN should be run on a small window (recommend 200Kb) using the -r parameter. If there are M variants in a window MarViN scales like M<sup>2</sup> for memory consumption and M<sup>3</sup> for speed. Linkage-Disequilibrium, which creates the correlation patterns used by MarViN, typically decays rapidly with distance. In our experiments we found window sizes between 50Kb and 200Kb to be adequate.

##Call genotypes from a population given likelihoods
Assuming that `input.vcf.gz` contains genotype likelihoods and is indexed (.csi or .tbi)
```
./marvin -f input.vcf.gz -O z -o out.vcf.gz -r chr20:200000-400000 -b 5000
```
`out.vcf.gz` will contain genotypes imputed under MarViN's LD model, using overlap of 5000 on either side of the specified window.

##Improve calls in sample given a reference panel
Preprocessing the panel, which must be done once to precalculate the necessary matrix inverses.
```
./marvin_prep -f panel.vcf.gz -O z -o sites.vcf.gz -om mu.dat -ov sig.dat -ow omega.dat -r chr20:195000-405000
```
Then run
```
./marvin -f input.vcf.gz -O z -o out.vcf.gz -r chr20:200000-400000 -b 5000 -fm mu.dat -fv sig.dat -fw omega.dat -max_its 1 -inner_its 5
```
`out.vcf.gz` has the GT field added or overwritten with the imputed genotypes.


