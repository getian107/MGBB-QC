# Mass General Brigham Biobank genotyping data QC (Feb 2022 release)

This repository details the quality control (QC) pipeline of the MGB Biobank samples genotyped on Illumina’s GSA chip, which largely follows the recommendations described the reference below and leverages the scripts (https://github.com/Annefeng/PBK-QC-pipeline) used to QC ealier MGBB releases genotyped on Illumina’s Multi-Ethnic Global array:

Peterson, R. E., Kuchenbaecker, K., Walters, R. K., Chen, C.-Y., Popejoy, A. B., Periyasamy, S., et al. (2019). Genome-wide Association Studies in Ancestrally Diverse Populations: Opportunities, Methods, Pitfalls, and Recommendations. Cell, 179(3), 589–603. http://doi.org/10.1016/j.cell.2019.08.051

The dataset includes 47,321 individuals genotyped on Illumina’s GSA chip in hg19 coordinates.

Note that there is substantial overlap between the samples genotyped on the MEG array and GSA array.


## Quality control pipeline

* QC for each genotyping batch (metrics not affected by ancestry; `scripts 01-04`)
	* SNP-level call rate >0.95
	* Sample-level call rate >0.98
	* SNP-level call rate >0.98
	* Maximum SNP-level missing rate difference between two batches < an empirical threshold (0.75%)

* Merge genotyping batches (`scripts 05-06`)
	* Remove duplicated SNPs
	* Remove monomorphic SNPs
	* Remove SNPs not confidently mapped (chr=0)

* Population assignment (`scripts 07-08`)
	* Select common, high-quality SNPs for population inference
		* SNP-level call rate >0.98
		* MAF >5%
		* Remove strand ambiguous SNPs and long-range LD regions (chr6:25-35Mb; chr8:7-13Mb inversion)
		* Prune to <100K independent SNPs
	* Identify individuals with European ancestry using selected SNPs
		* Run PCA combining study samples + 1KG data
		* Use Random Forest to classify genetic ancesty with a prediciton prob. >0.8
		* Remove PCA outliers (6SD away from the population mean for each of the top 10 PCs)

* QC within European samples (metrics may be affected by ancestry; `scripts 09-13`)
	* Remove samples that fail sex check (--check-sex)
	* Absolute value of autosomal heterozygosity rate deviating from the mean (5SD; --het)
	* Remove SNPs that show batch associations
		* Regress each batch indicator on SNPs, adjusting for sex (association P < 1e-4)

* Calculate PCs within related European samples using common, high-quality SNPs (`script 14`)

* Final SNP-level QC within European samples (`script 15`)
	* SNP-level call rate >0.98
	* HWE >1e-10
	* Retain only autosomal SNPs, excluding indels and monomorphic SNPs (for imputation; HRC: SNP only)

* Prepare data for HRC/1KG imputation using Michigan server (`scripts 16-17`)
	* Harmonize study data with HRC/1KG data
	* Convert plink to vcf by chromosome

* Send European samples to Michigan server for imputation (using HRC as the reference panel)

* Post-imputation QC (converting vcf dosages to plink hard-call genotypes; `scripts 18-20`)
	* INFO score/Imputation R2 >0.6
	* MAF (based on imputed dosages) >0.5%
	* HWE >1e-10
	* SNP-level call rate >0.98
	


## Summary of pre-imputation QC

### Genetic ancestry assignment
- Random Forest prediction probability > 0.8 based on top 6 PCs, with 1KG samples as the training data
- "Unclassified" consists of mostly admixed individuals

| 1KG superpop    |  EUR   |  AMR   |  AFR   |  EAS   |  SAS   | Unclassified | Total |
| --- | -----: | -----: | -----: | -----: | -----: | -----------: | -----:|   
| # Samples | 33,614 | 2,599 | 2,036 | 908 | 480 | 7,682 | 47,319 |
| % Total | 71.1% | 5.5% | 4.3% | 1.9% | 1.0% | 16.2% | 100% |



### Sample QC
- Samples are genotyped on 3 batches. Batch 0110: 13,140; batch 0111: 11,649; batch 0113: 22,532

| Sample QC metric | # Samples | % Total |
| ---------------- | -------: | -----: |
| Initial sample size | 47,321 | 100%  |
| **_Batch QC:_**  |   |   |
| Sample-level call rate < 0.98  | 2  | 0.0%  |
| **_Merged QC:_**  |   |   |
| Non-European | 13,705 | 29.0% |
| **_EUR (pop-specific) QC:_**  |   |   |
| Failing sex check (reported != imputed sex, using F < 0.2 <br>for female & >0.8 for male) | 332  | 0.87%  |
| Outlying heterozygosity rate (>5SD from the mean) | 134  | 0.35%  |
| _Any of the above two_ | 451  | 0.95%  |
| PCA outliers (6SD away from the mean in top 10 PCs) | 96 | 0.20% |
| **_Post-QC_** | 33,067 | 69.9%  | 


### Variant QC

| Variant QC metric  | # Variants | % Total |
| ------------- | -------------: | -------------: |
| Initial variant count (avg. across batches) | 0.68M | - |
| **_Batch QC:_**  |   |   |
| SNP call rate < 0.95 and then < 0.98 (avg. across batches)| 10K  | -  |
| Common across batches | 656,598 | - |
| Missing rate diff > 0.0075 between any two batches  | 8,295  | -  |
| **_Merged QC:_**  |   |   |
| Total  | 648,303 | 100%  |
| Monomorphic SNPs  | 14,753 | 2.28%  |
| Duplicated SNPs  | 427  | 0.07%  |
| Not confidently mapped (chr & pos=0)  | 0 | 0%  |
| _Any of the above three_  | 15,184  | 2.34%  |
| **_EUR (pop-specific) QC:_**  | 633,119  | 97.66%  |
| Showing batch association (p < 1e-04)  | 4,112  | 0.63%  |
| **_Final QC:_**  |   |   |
| SNP-level call rate < 0.98  | 14  | 2.16e-3% |
| pHWE < 1e-10  | 7,541  | 1.16% |
| Non-autosomal, indel, or monomorphic  | 36,868 | 5.69% |
| **_Post-QC_** | 584,584 | 90.17%  |
| **_HRC QC (for imputation):_**  | | |
| Not in HRC or mismatched info  | 41,944  | 6.47% |
| **_Send to Michigan imputation server_**  | 542,640  | 83.7%  |



## Summary of post-imputation QC

| Variant filter     | # Variants | % Total |   
| ------------------ | ---------: | ------: |
| Total imputed | 34,921,029 | 100% | 		   
| MAF (dosage) < 0.005 or Imputation R2 < 0.6 | 26,240,369 | 75.1% |    
| Call rate < 0.98 or pHWE < 1e-10 | 1,395 | 4.00e-3% |       
| **_Post-QC_** | 8,679,265 | 24.9% |

