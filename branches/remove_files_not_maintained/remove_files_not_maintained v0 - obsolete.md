# Removing files that are not used or not maintained - v0 - obsolete

There are many files that are not used or not maintained in the repo, and I think this causes confusion. 

We can add a note to the readme that additional scripts were removed and can be viewed by going to commit X, and linking the commit. 



I propose the following, keeping only the functionality supporting quant, intron_coverage, and ir_table



## directories and top level files

### remove 



#### `scripts/` (entire directory)

It contains only `run-drim-seq.R` — the DRIMSeq integration script. You don't use DRIMSeq, so this is safe to remove. Note that `setup.py` references it via `script=["scripts/run-drim-seq.R"]`, but that line is a minor packaging artifact and removing the file won't break `pip install -e .` or any of the three subcommands you use.

#### `data/` (entire directory)

Example/test data files. Not needed at runtime.

#### `docs/` (entire directory)

Documentation only.

#### `tests/` (entire directory)

The only tests are under `tests/e2e/quant/`. Safe to remove since you're not running the test suite.

#### `Pipfile`

Only needed for `pipenv`. Since you use `pip`, irrelevant.

#### `MANIFEST.in`

Only needed for building a PyPI distribution package. Not needed for `pip install -e .`.



### keep

| Item                      | Why                                                          |
| ------------------------- | ------------------------------------------------------------ |
| `setup.py`                | Wires up the `splicedice` CLI entry point                    |
| `requirements.txt`        | Read by `setup.py` at install time                           |
| `README.md`               | Read by `setup.py` for `long_description` — removing it would cause `pip install` to crash |
| `.gitignore`              | Harmless, but irrelevant to function                         |
| `LICENSE`, `CHANGELOG.md` | Tiny, harmless                                               |

## in the splicedice code directory

### remove

```bash
splicedice/compareSampleSets.py   # compare_sample_sets
splicedice/pairwise_fisher.py     # pairwise
splicedice/findOutliers.py        # findOutliers
splicedice/subset.py              # subset
splicedice/similarity.py          # similarity
splicedice/select_samples.py      # select
splicedice/counts_to_ps.py        # counts_to_ps
```

### Keep

```bash
splicedice/__init__.py            # package init, always required
splicedice/__main__.py            # CLI entry point
splicedice/SPLICEDICE.py          # quant
splicedice/intron_coverage.py     # intron_coverage
splicedice/ir_table.py            # ir_table
```

I checked whether any of the "keep" modules import from the "remove" modules; they do not.

```bash
grep -l "import compareSampleSets\|import pairwise_fisher\|import findOutliers\|import subset\|import similarity\|import select_samples\|import counts_to_ps" \
  splicedice/SPLICEDICE.py splicedice/intron_coverage.py splicedice/ir_table.py
```

in splicedice.py, do i need the  writeDrimLine and writeDrimTable functions? 



# Test

server:

ubuntu@hbeale-mesa



# Setup per run

## define location

```bash
this_commit=f112480
this_description=test_Remove-untested-code
this_datestamp=2026.06.22_16.15.05
this_branch=Remove-untested-code
this_dockerfile=Dockerfile_${this_description}.txt
```

```bash

this_base_dir=/mnt/sd/${this_description}_${this_commit}_${this_datestamp}/
code_base=${this_base_dir}/git_code/splicedice_analysis/examples/test-Remove-untested-code
mkdir -p ${this_base_dir}/git_code/ ${this_base_dir}/analysis/

```



## get code

```bash
cd ${this_base_dir}/git_code/
git clone https://github.com/hbeale/splicedice_analysis.git
```

## build docker

```bash
cd ${code_base}
cat ${this_base_dir}/git_code/splicedice_analysis/2026_06_ps_ir_pipeline/Dockerfile_by_branch.txt | sed "s/replace_with_branch/$this_branch/" > ${code_base}/${this_dockerfile}

docker build --build-arg CACHE_BUST=$(date +%s) -t splicedice_analysis:latest -f ${code_base}/${this_dockerfile} .
bash ~/alert_msg.sh "docker build complete"

```

### confirm that the docker contains the expected code

splicedice dir should contain many fewer files than before

```bash
docker run --rm -it --entrypoint=/bin/bash $this_docker -c "ls -lth /usr/local/lib/python3.8/site-packages/splicedice/"
```



```bash

total 44K
drwxr-xr-x 2 root root 4.0K Jun 22 23:32 __pycache__
-rw-r--r-- 1 root root  11K Jun 22 23:32 ir_table.py
-rw-r--r-- 1 root root 8.9K Jun 22 23:32 intron_coverage.py
-rw-r--r-- 1 root root 2.4K Jun 22 23:32 __main__.py
-rw-r--r-- 1 root root  12K Jun 22 23:32 SPLICEDICE.py
-rw-r--r-- 1 root root    0 Jun 22 23:32 __init__.py

```

hm, i didn't expect `__pycache__`, 

## identify manifests

```bash
bed_manifest=${this_base_dir}/git_code/splicedice_analysis/SUGP1/bed_manifest_2026-05-05_20-28-22.tsv
bam_manifest=${this_base_dir}/git_code/splicedice_analysis/SUGP1/bam_manifest.tsv 

```

# Run



## run quant



```bash
date
time docker run --rm \
-v /mnt/:/mnt \
splicedice_analysis:latest \
splicedice quant -m ${bed_manifest} \
-o ${this_base_dir}/analysis/
date
 ~/alert_msg.sh "quant run complete"

```

## 



error:

```bash
Mon Jun 22 23:44:19 UTC 2026
Traceback (most recent call last):
  File "/usr/local/bin/splicedice", line 5, in <module>
    from splicedice.__main__ import main
  File "/usr/local/lib/python3.8/site-packages/splicedice/__main__.py", line 6, in <module>
    from . import pairwise_fisher as pf
ImportError: cannot import name 'pairwise_fisher' from 'splicedice' (/usr/local/lib/python3.8/site-packages/splicedice/__init__.py)


```

# Troubleshooting

## rebuild with verbose output

```bash

cd ${code_base}

docker build --progress=plain \
  --build-arg CACHE_BUST=$(date +%s) \
  -t splicedice_analysis:latest \
  -f ${code_base}/${this_dockerfile} \
  . 2>&1 | tee build.log
bash ~/alert_msg.sh "docker build complete"

```

the only wanring i got was

`\#9 16.51 update-alternatives: warning: skip creation of /usr/share/man/es/man1/fakeroot.1.gz because associated file /usr/share/man/es/man1/fakeroot-tcp.1.gz (of link group fakeroot) doesn't exist`

### try running quant again

```bash

    date
    time docker run --rm \
    -v /mnt/:/mnt \
    splicedice_analysis:latest \
    splicedice quant -m ${bed_manifest} \
    -o ${this_base_dir}/analysis/
    date
     ~/alert_msg.sh "quant run complete"

```

same error as before

```bash
Traceback (most recent call last):
  File "/usr/local/bin/splicedice", line 5, in <module>
    from splicedice.__main__ import main
  File "/usr/local/lib/python3.8/site-packages/splicedice/__main__.py", line 6, in <module>
    from . import pairwise_fisher as pf
ImportError: cannot import name 'pairwise_fisher' from 'splicedice' (/usr/local/lib/python3.8/site-packages/splicedice/__init__.py)

```

the repo shouldn't have any references to pairwise_fisher. 

```bash
docker run --rm -it --entrypoint=/bin/bash splicedice_analysis:latest
cd /usr/local/lib/python3.8/site-packages/splicedice
cat __main__.py
```

ah, they're in __main__.py

## update `__main__.py` and rebuild

```bash
cd /mnt/sd/test_Remove-untested-code_f112480_2026.06.22_16.15.05/git_code/splicedice_analysis/examples/test-Remove-untested-code
docker build --progress=plain   --build-arg CACHE_BUST=$(date +%s)   -t splicedice_analysis:latest   -f ${code_base}/${this_dockerfile}   . 2>&1 | tee build2.log
```

### re-run quant



```bash
date
time docker run --rm \
-v /mnt/:/mnt \
splicedice_analysis:latest \
splicedice quant -m ${bed_manifest} \
-o ${this_base_dir}/analysis/
date
~/alert_msg.sh "quant run complete"

```



```bash
Tue Jun 23 17:31:59 UTC 2026
Parsing manifest...
        Done [0:00:0.00]
Getting all junctions from 6 files...
        Done [0:00:4.08]
Finding clusters from 333069 junctions...
        Done [0:00:3.02]
Writing cluster file...
        Done [0:00:3.55]
Writing junction bed file...
        Done [0:00:2.05]
Gathering junction counts...
        Done [0:00:5.97]
Writing inclusion counts...
        Done [0:00:5.34]
Calculating PS values...
        Done [0:00:14.11]
Writing PS values...
        Done [0:00:5.44]
All done [0:00:43.57]

real    0m47.974s
user    0m0.053s
sys     0m0.056s
Tue Jun 23 17:32:47 UTC 2026
{"status":"OK","nsent":2,"apilimit":"2\/1000"}
```

### run intron coverage

```bash
n_threads=8
mkdir ${this_base_dir}/coverage_output/
date
time docker run --rm \
    -v /mnt/:/mnt \
    splicedice_analysis:latest \
    splicedice intron_coverage \
    -b $bam_manifest  \
    -j ${this_base_dir}/analysis/_junctions.bed \
    -n "${n_threads}" \
    -o ${this_base_dir}/coverage_output/
date
~/alert_msg.sh "intron_coverage run complete"


```



```bash
Tue Jun 23 17:37:30 UTC 2026
getting paths for bam files
creating junction percentiles
SRR12801019 starting 4.923929929733276
SRR12801019 collected 662.4295976161957
SRR12801019 counted 1298.469098329544
SRR12801019 done 1312.528217792511
SRR12801023 starting 6.5050530433654785
SRR12801023 collected 692.4812061786652
SRR12801023 counted 1345.8466906547546
SRR12801023 done 1360.9949433803558
SRR12801024 starting 7.282258749008179
SRR12801024 collected 818.651285886764
SRR12801024 counted 1491.1707291603088
SRR12801024 done 1504.774990081787
SRR12801028 starting 8.932100057601929
SRR12801028 collected 855.0672144889832
SRR12801028 counted 1583.3459951877594
SRR12801028 done 1598.885360956192
SRR12801027 starting 8.140107870101929
SRR12801027 collected 823.679744720459
SRR12801027 counted 1585.147456407547
SRR12801027 done 1600.4222221374512
Your runtime was 1625.9853701591492 seconds.
```



## Create intron table

takes about an hour

```bash
genes=/mnt/ref/gencode.v47.primary_assembly.annotation.gtf
date
time docker run --rm \
-v /mnt/:/mnt \
splicedice_analysis:latest \
splicedice ir_table \
--annotation $genes \
-i ${this_base_dir}/analysis/_inclusionCounts.tsv \
-c ${this_base_dir}/analysis/_allClusters.tsv \
-d ${this_base_dir}/coverage_output \
-n 8 \
-o ${this_base_dir}/analysis/
date
~/alert_msg.sh intron_table_creation_complete 

```



std out

```bash
Tue Jun 23 18:09:43 UTC 2026

Starting ir_table with 6 samples
Loading annotation...
Annotation loaded: 528735 annotated junctions. 62.9s
Gathering inclusion counts and clusters...
Loaded 6 samples and 333069 clusters. 69.9s
Collecting junctions across all samples...
getJunctions complete: 210470 junctions. 5.5s
RSD filtering complete: 96399 junctions retained. 24.8s
Junction collection and RSD filtering complete: 96399 junctions retained. 94.6s
Writing IR table...
IR calculated for 6/6 samples
IR table written. 152.0s
Done. Total runtime: 152.0s
real    2m36.147s
user    0m0.068s
sys     0m0.055s
Tue Jun 23 18:12:19 UTC 2026
```



# Claude prompts - useless

## setup



```bash
cd /mnt/scratch/claude_sd_deletions
git clone https://github.com/hbeale/splicedice_analysis.git
git clone --depth 1 --branch master \
      https://github.com/BrooksLabUCSC/splicedice.git splicedice_master
git clone --depth 2  --branch Remove-untested-code  \
      https://github.com/BrooksLabUCSC/splicedice.git splicedice_Remove-untested-code
```



## prompts



* clone this repo: https://github.com/hbeale/splicedice_analysis.git - failed; manually cloned above
* The code in splicedice_master works with this Docker, splicedice_analysis/code/Dockerfile_master_branch, but when I try to build it from the code in splicedice_Remove-untested-code with the dockerfile "examples/test-Remove-untested-code/Dockerfile_test_Remove-untested-code.txt", and run the docker, it fails. 

```bash


docker run --rm \
-v /mnt/:/mnt \
splicedice_analysis:latest \
splicedice quant -m splicedice_analysis/SUGP1/bed_manifest_2026-05-05_20-28-22.tsv \
-o ./analysis/
```

