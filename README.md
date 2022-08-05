# Mass General Brigham Biobank genotyping data QC (Feb 2022 release)

This repository details the quality control (QC) pipeline of the Partners Biobank genotype data, which largely follows the recommendations described in:

Peterson, R. E., Kuchenbaecker, K., Walters, R. K., Chen, C.-Y., Popejoy, A. B., Periyasamy, S., et al. (2019). Genome-wide Association Studies in Ancestrally Diverse Populations: Opportunities, Methods, Pitfalls, and Recommendations. Cell, 179(3), 589–603. http://doi.org/10.1016/j.cell.2019.08.051

The dataset includes 47,319 individuals genotyped on Illumina’s GSA chip in hg19 coordinates.


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
		* Remove individuals PCA outlier (6SD away from the top 10 PCs)

* QC within European samples (metrics may be affected by ancestry; `scripts 09-13`)
	* Remove samples that fail sex check (--check-sex)
	* Absolute value of autosomal heterozygosity rate deviating from the mean (5SD; --het)
	* (SKIP) Identify unrelated Individuals (Pi_hat <0.2) within European samples
	* Remove SNPs that show batch associations
		* Regress each batch indicator on SNPs, adjusting for sex (association P < 1e-4)

* Calculate PCs within related European samples using common, high-quality SNPs (`script 14`)

* Final SNP-level QC within related European samples (`script 15`)
	* SNP-level call rate >0.98
	* HWE >1e-10
	* Retain only autosomal SNPs, excluding indels and monomorphic SNPs (for imputation; HRC: SNP only)

* Prepare data for HRC/1KG imputation using Michigan server (`scripts 16-17`)
	* Harmonize study data with HRC/1KG data
	* Convert plink to vcf by chromosome

* Send related European samples to Michigan server for imputation (using HRC as the reference panel)

* Post-imputation QC (converting vcf dosages to plink hard-call genotypes; `scripts 18-20`)
	* INFO score/Imputation R2 >0.6
	* MAF (based on imputed dosages) >0.5%
	



## Summary of pre-imputation QC

### Genetic ancestry assignment
- Random Forest prediction probability > 0.8 based on top 6 PCs, with 1KG samples as the training data
- "Unclassified" consists of mostly admixed individuals

| 1KG superpop    |  EUR   |  AMR   |  AFR   |  EAS   |  SAS   | Unclassified | Total |
| --- | -----: | -----: | -----: | -----: | -----: | -----------: | -----:|   
| # Samples | 38,046 | 5,361 | 2,327 | 1,037 | 297 | 548 | 47,319 |
| % Total | 80.4% | 11.3% | 4.9% | 2.2% | 0.6% | 1.2% | 100% |



### Sample QC
- Samples are genotyped on 3 batches. Batch 0110: 13,140; batch 0111: 11,649; batch 0113: 22,532

| Sample QC metric | # Samples | % Total |
| ---------------- | -------: | -----: |
| Initial sample size | 47,321 | 100%  |
| **_Batch QC:_**  |   |   |
| Sample-level call rate < 0.98  | 2  | 0.0%  |
| **_Merged QC:_**  |   |   |
| **_EUR (pop-specific) QC:_**  |   |   |
| Failing sex check (reported != imputed sex, using F < 0.2 <br>for female & >0.8 for male) | 332  | 0.87%  |
| Outlying heterozygosity rate (>5SD from the mean) | 134  | 0.35%  |
| _Any of the above two_ | 451  | 0.95%  |
| **_Post-QC_** | 37,259  | 78.7%  |


### Variant QC

| Variant QC metric  | # Variants | % Total |
| ------------- | -------------: | -------------: |
| Initial variant count (avg. across batches) | 0.68M | - |
| **_Batch QC:_**  |   |   |
| SNP call rate < 0.95 and then < 0.98 (avg. across batches)| 0.67M  | -  |
| Common across batches | 656,598 | - |
| Missing rate diff > 0.0075 between any two batches  | 8,295  | -  |
| **_Merged QC:_**  |   |   |
| Total  | 648,303 | 100%  |
| Monomorphic SNPs  | 14,753 | 2.28%  |
| Duplicated SNPs  | 431  | 0.07%  |
| Not confidently mapped (chr & pos=0)  | 0 | 0%  |
| _Any of the above three_  | 15,184  | 2.34%  |
| **_EUR (pop-specific) QC:_**  |   |   |
| Showing batch association (p < 1e-04)  | 4,112  | %  |
| **_Final QC:_**  |   |   |
| SNP-level call rate < 0.98  |   | 7e-05%  |
| pHWE < 1e-10  |   | 0.18%  |
| Non-autosomal, indel, or monomorphic  |   | 9.5%  |
| **_Post-QC_** |  | % |
| **_HRC QC (for imputation):_**  |   |   |
| Not in HRC or mismatched info  |   | %  |
| **_Send to Michigan imputation server_**  | 542,640  | 83.7%  |



## Summary of post-imputation QC

| Variant filter (1) | # Variants | % Total |   | Variant filter (2)  | # Variants | % Total |
| ------------------ | ---------: | ------: |---| ------------------- | ---------: | ------: |
| Total imputed | 33,822,636 | 100% | 		    | Total imputed | 33,822,636 | 100% | 
| MAF (dosage) < 0.01 | 26,051,805 | 77.0% |    | MAF (dosage) < 0.005 | 24,946,776 | 73.8% | 
| Imputation R2 < 0.8 | 17,443,564 | 51.6% |    | Imputation R2 < 0.6 | 9,993,397 | 29.5% |
| _Any of the above two_ | 26,468,624 | 78.3% | | _Any of the above two_ | 25,100,024 | 74.2% |
| Call rate < 0.98 | 0 | 0.0% |                 | Call rate < 0.98 | 0 | 0.0% |
| pHWE < 1e-10 | 167 | 5e-04% |                 | pHWE < 1e-10 | 213 | 6e-04% |
| **_Post-QC_** | 7,353,845 | 21.7% |           |**_Post-QC_** | 8,722,399 | 25.8% |


