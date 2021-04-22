# Chrom-Lasso
## Introduction of Chrom-Lasso
Chrom-Lasso is a tool to analyze Hi-C data for identifying cis chromatin interactions (interactions occur within chromosome).
## Introduction of Folders
The following folders are code, input files, test data, and tutorial for Chrom-Lasso.
1. Code：
This folder contains codes for Chrom-Lasso, the usage of them is shown in the "Tutorial" folder.
2. Prepare_Input_File：
This folder contains input files needed by Chrom-Lasso to do analysis, mainly are domain files and cutting site files for Mouse and Human,
The domain files are constant for specific species, but the genomic coordinate in the file can be changed with genome version by tool "LiftOver".
The cutting site files can be produced by tool "Oligomatch" with reference genome and cutting sequence for the restriction endonuclease.
3. Test_Data：
This folder contains "sortChr" files for Mouse and Human for testing Chrom-Lasso, the "sortChr" file are preprocessed file from raw "fastq" sequencing files,
the preprocessing of "fastq" files can be done with our tutorial, which is the same as "Juicer", so you can use "Juicer" as well.
The "Juicer" generates "merged_nodups" file, and you can use the Shell scripts in tutorial that sorts the file by chromosome order to generate "sortChr" file for further analysis.
4. Tutorial：
This folder contains the analysis pipeline for Mouse and Human, you can directly use "sotrChr" files in "Test_Data" folder to test the pipeline, 
you can also use your own "fastq" files from Hi-C experiments to test the tutorial step by step from the preprocessing to running Chrom-Lasso.
## Environment
The compile of Chrom-Lasso recommends: gcc (4.9.2), boost_1.51. And it also needs R (>=3.0) to run polynomial regression and lasso regression. Users can use "makefile" in Code folder for compile.
## Tutorial (mouse)
### Prepare input files
Chrom-Lasso needs 3 input files prepared according to the Hi-C experimental design. 
#### 1. Cutting site file
The cutting site file contains the cutting sites of restriction enzyme used in Hi-C experiments. Each line stands for a cutting site locus on the genome.<br>  
![cutting site file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/cutting_site_bed.png)<br>  
This bedfile can be generated by an open source tool OligoMatch following the instruction in /Prepare_Input_File/Cutting_Site_File/Prepare_CuttingSite_File_By_Oligomatch.
#### 2. Domain file
The domain file contains the genomic regions identified as TADs. Each line stands for a TAD identified.<br>  
![domain file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/domain_file.png)<br>
The raw domain files are downloaded from: (Dixon, J., Selvaraj, S., Yue, F. et al. Topological domains in mammalian genomes identified by analysis of chromatin interactions. 
Nature 485, 376–380 (2012)). The raw version for Mouse genome is mm9 and for Human genome is hg18. Users can transform the genome version via tool "LiftOver". Here, we provide users with domain filed for mouse and human in /Prepare_Input_File/Domain_File/.
#### 3. sortChr file
Users should first generate merged_nodups.txt file by JUICER (https://github.com/aidenlab/juicer) from raw Hi-C sequencing fastq files.<br>  
Here, we use Mouse_merged_nodups.txt file as example to generate sortChr file<br>
```
awk '{if($1==0){strand1=1;}else if($1==16) {strand1=0;} if($5==0){strand2=1;}else if($5==16) {strand2=0;} print $2"\t"$3"\t"strand1"\t"$6"\t"$7"\t"strand2"\t0";}' Mouse_merged_nodups.txt > Mouse.formatted
for chr in {1..19} X;do awk '{if($1=="'$chr'") print $0;}' Mouse.formatted;done > Mouse.sortChr
```
The sortChr file contains the paired end sequencing information of Hi-C data.<br>
![sortChr file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/sortChr.png)<br>
### Run Chrom-Lasso to detect cis chromatin interactions
#### 1. Arrange hybrid fragments based on domain file and cutting site file
```
/Code/2_Arrange_Domain/HiC_mixturePLD_singleThread -g mm10 -w 51 500 10000 26 1 -d /Prepare_Input_File/Domain_File/total.domain.mm10  -c
/Prepare_Input_File/Cutting_Site_File/MboI.mm10.bed Mouse.sortChr > "Mouse.sortChr_summary"
```
##### Parameters:
-g: genome version (mm9, mm10, hg18, hg19)<br>
-w: read size, distance to cutting site threshold, model bin size, number of nodes, cores per node<br>
-d: path to domain file<br>
-c: path to cutting site file
##### Results:
For each chromosome, this step generates 3 files, "chr_csInterChromTotalMap", "chr_domainCSinterFreq", "chr_domainSitesMap".<br>
chr_csInterChromTotalMap: this file contains inter-chromosomal hybrid fragments for evaluating bias.<br>
chr_domainCSinterFreq: this file contains cutting site contact frequency for each domain.<br>
chr_domainSitesMap: this file contains cutting site loci on this chromosome.
#### 2. Model the background distribution
```
for chr in chr{1..19} chrX;do awk 'function abs(x){return ((x < 0.0) ? -x : x)}BEGIN{i=0;}{if($2!=0){map[i]=$1;++i;}}END{for(j=0;j<i;++j){for(k=j+1;k<i;++k){dis=abs(map[j]-map[k]);if(dis<100000){dist[int(dis/100)]++;}else{break;}}}for(i=0;i<1000;++i) print i"\t"dist[i];}' "$chr"_csInterChromTotalMap;done | awk '{map[$1]+=$2;}END{for(i=0;i<1000;++i) print i"\t"map[i];}' > csDistanceDistr
cat Mouse.sortChr | awk 'function abs(x){return ((x < 0.0) ? -x : x)}{dis=abs($2-$5);if($1==$4 && dis<1000000){map[int(dis/100)]++;}}END{for(i=0;i<10000;++i) print i"\t"map[i];}' > empericalDist.bin100				
cat Mouse.sortChr | awk 'function abs(x){return ((x < 0.0) ? -x : x)}BEGIN{nonloop=0;}{if($3!=$6 && (($2<$5 && $3==1) || ($2>$5 && $3==0))){nonloop++;}else{dis=abs($2-$5);if($1==$4 && dis<1000000){map[int(dis/100)]++;}}}END{for(i=0;i<10000;++i) {print i"\t"map[i];} print i"\t"nonloop}' > bin100.withNonloop
Rscript /Code/3_Model_Distribution/empericalDist.r
```
##### Results:
This step generates the "PolyCoef" file used for estimating data background.<br>
"PolyCoef" file contains parameters used in the 7th-degree polynomial regression.<br>
#### 3. Identify interactions for lasso regression
```
for chr in {1..19} X;
do 
   mkdir chr$chr
   cd chr$chr
   /Code/4_Find_IntraDomain_Interaction/findIntraDomainInteraction  /Output_path/chr$chr"_csInterChromTotalMap" /Output_path/chr$chr"_domainSitesMap" /Output_path/chr$chr"_domainCSinterFreq"  chr$chr"_out"  /Output_path/PolyCoef \-13.95 1.854 > chr$chr"_out_summary"
   cd ../
done;
```
##### Results:
For each chromosome, this step generates an independent folder containing "regionData" and "distMatrix" files for each domain on this chromosome.<br>
"regionData" and "distMatrix" files containing frequency and genomic distance information for interactions that need lasso regression to select.
#### 4. Lasso regression to select interactions
```
for chr in {1..19} X;
do 
   cd chr$chr
   domainNum=`tail -n1 chr"$chr"_out_debug | awk '{print $2}'`
   for (( c=0; c<=$domainNum; c++ ));
   do
   Rscript /Code/5_Lasso_Determine_Center/nnlasso.HiC.r regionData_$c distMatrix_$c > nnlassoOut_$c
   done;
cd ../
done;
```
##### Results:
In each chromosome folder, this step generates "nnlassoOut" file for each domain, which contains the coefficients resulting from lasso regression.
#### 5. Identify independent interacting center
```
for chr in {1..19} X;
do 
   cd chr$chr
   /Code/6_Test_Distribution/outputTestPosPvalue  /Output_path/chr$chr"_csInterChromTotalMap" /Output_path/chr$chr"_domainSitesMap" /Output_path/chr$chr"_domainCSinterFreq" chr$chr"_posP" /Output_path/Mouse.polyCoef \-13.95 1.854
   cd ../
done;
```
##### Results:
In each chromosome folder, this step generates "oneCol" and "testPosP" files for each domain, which contain the information for independent interacting center.
#### 6. Summary of interactions
```
for chr in {1..19} X
do
    cd chr$chr
    domainNum=`tail -n1 chr"$chr"_out_debug | awk '{print $2}'`
    for (( c=0; c<=$domainNum; c++ ))
    do
        testNum1=`cat nnlassoOut_"$c" | tail -n2 | head -1 | awk '{print $2}'`;testNum2=`cat regionData_"$c" | tail -n2 | head -1 | awk '{print $2}'`;testNum3=`wc -l regionData_"$c" | awk '{print $1;}'`
        if [ "$testNum1" = "$testNum2" ] || [ "$testNum3" == 0 ]
        then
            echo $chr.$c.match
        else
            echo $chr.$c.notMatch
        fi
    done
    cd ../
done > domainNnlassoResults
for chr in {1..19} X
do
    echo $chr
    cd chr$chr
    domainNum=`tail -n1 chr"$chr"_out_debug | awk '{print $2}'`
    for (( c=0; c<=$domainNum; c++ ))
    do
        testNum1=`cat nnlassoOut_"$c" | tail -n2 | head -1 | awk '{print $2}'`;testNum2=`cat testPosP_"$c" | awk '{if(NF==7) {m=$3;}}END{print m;}'`;testNum3=`wc -l regionData_"$c" | awk '{print $1;}'`
        if [ "$testNum1" = "$testNum2" ] || [ "$testNum3" == 0 ]
        then
            echo $chr.$c.match
        else
            echo $chr.$c.notMatch
        fi
    done
    cd ../
done > domainTestPos
for chr in {1..19} X
do
    cd chr$chr
    domainNum=`tail -n1 chr"$chr"_out_debug | awk '{print $2}'`
    for (( c=0; c<=$domainNum; c++ ))
    do
	cat testPosP_"$c" | awk '{if(NF==7) {printf("%s\t%s\t%s\t%s\t%s\t%s\t%s\t%.10f\t%.10f\n",$1,$2,$3,$4,$5,$6,$7,$6,$7);}else{print $0;}}' > temp1
	cat nnlassoOut_"$c" | awk '{if($1!~/^\[/) {printf("%.10f",$1);for(i=2;i<=NF;i++){printf("\t%.10f",$i);}printf("\n");}else{print $0;}}' > temp2
	awk 'FNR==NR{if(NF==9){i=$2;j=1;beta[i]=$6;pval[i]=$7}else{if(NF==3){map[i,j]=$0;}j++;} next;}{if($1~/^\[/){i=$2;}else{if(i in pval){for(j=1;j<=NF;j++) {if(((i,j) in map) && $j!="Error!"){print map[i,j]"\t"beta[i]"\t"pval[i]"\t"$j;}}}}}' temp1  temp2 | sort -k1,1n -k2,2n -u | gzip > sigTest_"$c".all.gz
    done
    cd ../
done
for chr in {1..19} X
do
    echo $chr
    cd chr$chr
    domainNum=`tail -n1 chr"$chr"_out_debug | awk '{print $2}'`
    for (( c=0; c<=$domainNum; c++ ))
    do
	testNum1=`cat nnlassoOut_"$c" | tail -n2 | head -1 | awk '{print $2+1}'`;testNum2=`zcat sigTest_"$c".all.gz | wc -l | awk '{print $1;}'`;testNum3=`wc -l regionData_"$c" | awk '{print $1;}'`
	if [ "$testNum1" = "$testNum2" ] || [ "$testNum3" == 0 ]
	then
	    echo $chr.$c.match
	else
	    echo $chr.$c.notMatch
	fi
    done
    cd ../
done > domainSigTest.all
for chr in {1..19} X;do zcat chr$chr/sigTest_*.all.gz | awk '{print "'$chr'\t"$0;}';cat chr$chr/oneCol_* | awk '{print "'$chr'\t"$1"\t"$2"\t0\t"$3"\t"$4"\t"$3;}'; done | sort -k1,1 -k2,2n -k3,3n > all_interactions
```
##### Results:
This step provides users with a summary of detected interactions.<br>
![results file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/results.png)<br>
Each line stands for a detected interaction.<br>
Column 1: chromosome<br>
Colume 2: interacting end1<br>
Column 3: interacting end2<br>
Column 4: users can ignore this column.<br>
Column 5: beta coefficients for testing distribution, which can be used to infer the relative proportion of cells with this interaction. The bigger this value, the larger the proportion.<br>
Column 6: p value<br>
Column 7: beta coefficients for lasso regression. if column 5 is not equal to column 7, this means that this interaction comes from lasso regression. Users can grap interactions from lasso regression seperately and select them based on column 7.<br>
#### 7. Estimate FDR level
```
for chr in {1..19} X;
do 
    cd chr$chr
   /Code/7_Present_Significance/randomSamplingForFDR /Output_path/chr$chr"_csInterChromTotalMap" /Output_path/chr$chr"_domainSitesMap" /Output_path/chr$chr"_domainCSinterFreq" chr$chr /Output_path/Mouse.polyCoef \-13.95 1.854
   cd ../
done;
cat chr*/randomSamples* > randomSamples.combined
Rscript /Code/7_Present_Significance/fdrFromRandomSamples.r
```
##### Results:
This step generates "randomSamples.combined.fdr" and "randomSamples.combined.posFdr" files to estimate the FDR level for interactions.<br>
"randomSamples.combined.fdr" contains FDR level for all randomly pickig loci pairs.<br>
"randomSamples.combined.posFdr" contains FDR level for randomly picking loci pairs with beta coefficients above 0.<br>
![fdr file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/fdr.png)<br>
Each line stands for a randomly picking loci pair.<br>
Column 5: p value multiple correction based on FDR<br>
Colume 6: p value multiple correction based on BY
##### Selecting:
Get significance level, for FDR<0.05:
```
awk '{if($5<0.05 && $5>0.0499) print $0;}' randomSamples.combined.posFdr | sort -k5,5n | tail -n1 | awk '{print $4}' > BH_0.05 
```
if BH_0.05=0.00000587
Select interactions with FDR<0.05
```
awk '{if($6<0.00000587) print $0;}' all_interactions > interactions_fdr0.05
```
### Reminder:
*1. For cutting site input file, please make sure this file is derived from the `same genome version` with the one you use for mapping.<br>
*2. When using Shell scripts in the totorial for sorting chromosome and do "for" circulation, please make sure the `total number of chromosomes` is changed befor running the code.<br>
*3. Chrom-Lasso focuses on detecting long-range chromatin interactions with the distance between interaction loci over 20000bp, but this parameter can be changed in the `Cpp code` to satisfy the needs of study.<br>
![distance file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/distance.png)<br>
*4. When testing for the reads distribution surrounding potential interaction loci pair, Chrom-Lasso defines the testing range by parameter "NEIGHBDIS" in `cpp code`, when this parameter is 5, it defines a 11 cutting site window centered by the potential loci, and you can change this parameter in Cpp code to satisfy the needs of study.<br>
![neighbour file](https://github.com/Lan-lab/Chrom-Lasso/blob/main/documentation/neighbour.png)<br>






 
