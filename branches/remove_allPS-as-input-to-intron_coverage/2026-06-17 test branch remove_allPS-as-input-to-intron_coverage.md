# 2026-06-17 test branch remove_allPS-as-input-to-intron_coverage.md



# Setup per server

## copy gdc file

```bash
cp /mnt/git_code/gdc-user-token.2026-05-28T20_33_35.481Z.txt ~
```

make sure gdc-client is in the path



# Setup per run

## define location

```bash
this_commit=6244a35
this_description=branch_test
this_base_dir=/mnt/sd/ex_${this_description}_${this_commit}_2026.06.17_16.36.39/
code_base=${this_base_dir}/git_code/splicedice_analysis/2026_06_ps_ir_pipeline
mkdir -p ${this_base_dir}/git_code/ ${this_base_dir}/analysis/
```



## get code

```bash
cd ${this_base_dir}/git_code/
git clone https://github.com/hbeale/splicedice_analysis.git
git clone https://github.com/hbeale/splicedice_modifications.git
```

## make dockerfile

```bash
cd ${this_base_dir}/git_code/splicedice_modifications/branches/quant_divide_by_zero_error
cat  ../../../splicedice_analysis/code/Dockerfile_ir_table_for_high_sample_number | sed 's/ir_table_for_high_sample_number/remove_allPS-as-input-to-intron_coverage/' > Dockerfile_remove_allPS-as-input-to-intron_coverage

```



## build docker

```bash

this_dockerfile=Dockerfile_remove_allPS-as-input-to-intron_coverage
docker build --build-arg CACHE_BUST=$(date +%s) -t splicedice_analysis:latest -f $this_dockerfile .
bash ~/alert_msg.sh "docker build complete"

```



## confirm that expected code is in the dockerfile

```bash
docker run --rm splicedice_analysis:latest splicedice intron_coverage --help
```

```bash
usage: splicedice intron_coverage [-h] -b BAMMANIFEST -j JUNCTIONFILE
                                  [-s BINSIZE] [-n NUMTHREADS] -o OUTPUTDIR

```

that's the expected result; --splicediceTable doesn't appear in the usage note



## make manifests

```bash
n_samples_to_keep=2
cat ${code_base}/manifests/primary_manifest.txt | sed "s|/mnt/data/tcga|${this_base_dir}/bams|" | \
sed "s|/mnt/data/intron_prospector_runs/common|${this_base_dir}/intron_beds|" | \
head -$(( n_samples_to_keep + 1 )) > ${this_base_dir}/analysis/primary_manifest.txt

cat ${this_base_dir}/analysis/primary_manifest.txt | grep -v dataset_id | cut -f1,3,4 > ${this_base_dir}/analysis/quant_manifest.txt

```



# Run pipeline

## jump ahead to intron_coverage step 

```bash
comparison_dir=/mnt/sd/ex_streamlined_4dd834b_2026.06.17_14.51.32/
cp ${comparison_dir}/analysis/_junctions.bed ${this_base_dir}/analysis/

```

## modify script

```bash
sed -i '/_allPS.tsv/d' ${code_base}/scripts/run_intron_coverage.sh
```

### check that script is fixed

```bash
cat ${code_base}/scripts/run_intron_coverage.sh | grep _allPS.tsv

```



## run intron_coverage

```bash
batch_size=2

bash ${code_base}/scripts/run_intron_coverage_pipeline.sh \
    --manifest ${this_base_dir}/analysis/primary_manifest.txt \
    --analysis-base ${this_base_dir}/analysis/ \
    --batch-size $batch_size \
    --disk-constraint no
 ~/alert_msg.sh "intron_coverage run complete"
 
 
```



```bash
 

=== 2 samples still need intron_coverage ===

========================================================
BATCH 1: 2 samples
========================================================

--- downloading BAMs for batch 1 ---
using token: /home/ubuntu/gdc-user-token.2026-05-28T20_33_35.481Z.txt
TCGA-86-8074-01A: BAM already present, skipping
TCGA-62-8402-01A: BAM already present, skipping
All BAMs already present, nothing to download.

--- running intron_coverage for batch 1 ---
Running intron_coverage on 2 samples with 2 threads...
Thu Jun 18 00:36:16 UTC 2026
getting paths for bam files
creating junction percentiles
TCGA-62-8402-01A starting 3.6051487922668457
TCGA-62-8402-01A collected 1034.8556258678436
TCGA-62-8402-01A counted 1789.358901500702
TCGA-62-8402-01A done 1797.860322713852
Your runtime was 2872.3977344036102 seconds.

real    47m57.401s
user    0m0.261s
sys     0m0.072s
Thu Jun 18 01:24:13 UTC 2026
intron_coverage complete.
--- disk_constraint=no: keeping BAMs ---

batch 1 complete

========================================================
All batches complete.
========================================================
{"status":"OK","nsent":2,"apilimit":"2\/1000"}
```







## Compare intron_Coverage results



```bash
diff -r ${comparison_dir}/analysis/coverage_output/ ${this_base_dir}/analysis/coverage_output/
```



identical!!

```bash
01:43 [hbeale-clin-validation:ex_branch_test_6244a35_2026.06.17_16.36.39]$ diff -r ${comparison_dir}/analysis/coverage_output/ ${this_base_dir}/analysis/coverage_output/
01:44 [hbeale-clin-validation:ex_branch_test_6244a35_2026.06.17_16.36.39]$ diff -rq ${comparison_dir}/analysis/coverage_output/ ${this_base_dir}/analysis/coverage_output/
01:44 [hbeale-clin-validation:ex_branch_test_6244a35_2026.06.17_16.36.39]$ ls ${comparison_dir}/analysis/coverage_output/
TCGA-62-8402-01A_intron_coverage.txt  TCGA-86-8074-01A_intron_coverage.txt
01:45 [hbeale-clin-validation:ex_branch_test_6244a35_2026.06.17_16.36.39]$ ls ${this_base_dir}/analysis/coverage_output/
TCGA-62-8402-01A_intron_coverage.txt  TCGA-86-8074-01A_intron_coverage.txt

```



​	go ahead and merge the branch "remove allPS from intron_coverage"
