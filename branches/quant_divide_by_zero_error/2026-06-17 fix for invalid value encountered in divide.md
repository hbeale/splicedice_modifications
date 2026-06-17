# 2026-06-17 fix for invalid value encountered in divide



# error:

```
/usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py:213: RuntimeWarning: invalid value encountered in divide
 psi[self.junctionIndex[junction],:] = inclusions / (inclusions + exclusions)
```



# the fix

lines 213-218 of "SPLICEDICE.py"

            denom = inclusions + exclusions
            psi[self.junctionIndex[junction],:] = np.where(
                denom > 0,
                np.divide(inclusions, denom, where=denom > 0),
                np.nan
            )



# steps to identify and test the fix



## Enter the container interactively

```bash
docker run -it --rm \
    -v /mnt/:/mnt \
    splicedice_analysis:latest \
    bash
```

## edit the file



```bash
python3 -c "
import re
path = '/usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py'
old = 'psi[self.junctionIndex[junction],:] = inclusions / (inclusions + exclusions)'
new = 'denom = inclusions + exclusions\n        psi[self.junctionIndex[junction],:] = np.where(denom > 0, inclusions / denom, np.nan)'
text = open(path).read()
assert old in text, 'pattern not found!'
open(path, 'w').write(text.replace(old, new))
print('done')
"
```

## run quant



```bash
this_base_dir=/mnt/sd/ex_streamlined_4dd834b_2026.06.17_14.51.32/
mkdir ${this_base_dir}/analysis/quant_nanfix_test/

splicedice quant \
    -m ${this_base_dir}/analysis/quant_manifest.txt \
    -o ${this_base_dir}/analysis/quant_nanfix_test/
```

failed



## edit the file again

```python
python3 -c "
path = '/usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py'
old = '''    def calculatePsi(self):
        \"\"\" \"\"\"
        psi = np.zeros((len(self.clusters),len(self.manifest)),dtype='float32')
        for junction in sorted(self.clusters):
            chromosome,left,right,strand = junction
            inclusions = self.counts[self.junctionIndex[junction],:]
            exclusions = np.zeros(len(self.manifest))
            for excluded in self.clusters[junction]:
                exclusions += self.counts[self.junctionIndex[excluded],:]
            denom = inclusions + exclusions
        psi[self.junctionIndex[junction],:] = np.where(denom > 0, inclusions / denom, np.nan)        
        return psi'''
new = '''    def calculatePsi(self):
        \"\"\" \"\"\"
        psi = np.zeros((len(self.clusters),len(self.manifest)),dtype='float32')
        for junction in sorted(self.clusters):
            chromosome,left,right,strand = junction
            inclusions = self.counts[self.junctionIndex[junction],:]
            exclusions = np.zeros(len(self.manifest))
            for excluded in self.clusters[junction]:
                exclusions += self.counts[self.junctionIndex[excluded],:]
            denom = inclusions + exclusions
            psi[self.junctionIndex[junction],:] = np.where(
                denom > 0,
                np.divide(inclusions, denom, where=denom > 0),
                np.nan
            )
        return psi'''
text = open(path).read()
assert old in text, 'pattern not found!'
open(path, 'w').write(text.replace(old, new))
print('done')
"
```

## run quant



```bash
this_base_dir=/mnt/sd/ex_streamlined_4dd834b_2026.06.17_14.51.32/
mkdir ${this_base_dir}/analysis/quant_nanfix_test/

splicedice quant \
    -m ${this_base_dir}/analysis/quant_manifest.txt \
    -o ${this_base_dir}/analysis/quant_nanfix_test/
```

ran without error!



std out

```bash
Parsing manifest...
        Done [0:00:0.00]
Getting all junctions from 2 files...
        Done [0:00:0.99]
Finding clusters from 206575 junctions...
        Done [0:00:1.42]
Writing cluster file...
        Done [0:00:1.46]
Writing junction bed file...
        Done [0:00:1.25]
Gathering junction counts...
        Done [0:00:1.41]
Writing inclusion counts...
        Done [0:00:1.86]
Calculating PS values...
        Done [0:00:6.38]
Writing PS values...
        Done [0:00:1.97]
All done [0:00:16.72]

```



## compare results

```bash
diff ${this_base_dir}/analysis/_allPS.tsv \
     ${this_base_dir}/analysis/quant_nanfix_test/_allPS.tsv
```

results are identical!

## eyeball the outputs

```bash
root@44e0993ad392:/opt# head ${this_base_dir}/analysis/_allPS.tsv | cut -f1-5
cluster TCGA-86-8074-01A        TCGA-62-8402-01A
chr1:15039-15795:-      1.000   1.000
chr1:15948-16606:-      1.000   0.800
chr1:16311-16606:-      0.000   0.200
chr1:16766-16857:-      1.000   1.000
chr1:17056-17232:-      0.899   0.869
chr1:17056-17605:-      0.062   0.071
chr1:17056-17914:-      0.000   0.011
chr1:17369-17525:-      0.000   0.038
chr1:17369-17605:-      0.859   0.755
root@44e0993ad392:/opt# head ${this_base_dir}/analysis/quant_nanfix_test/_allPS.tsv | cut -f1-5
cluster TCGA-86-8074-01A        TCGA-62-8402-01A
chr1:15039-15795:-      1.000   1.000
chr1:15948-16606:-      1.000   0.800
chr1:16311-16606:-      0.000   0.200
chr1:16766-16857:-      1.000   1.000
chr1:17056-17232:-      0.899   0.869
chr1:17056-17605:-      0.062   0.071
chr1:17056-17914:-      0.000   0.011
chr1:17369-17525:-      0.000   0.038
chr1:17369-17605:-      0.859   0.755
```



# Claude convo

me: Here's how I've been running quant. walk me through entering the docker interactively, changing the code to replace the bare divsion with a nan-safe version and testing if the output is the same as existing output

date time docker run --rm \ -v /mnt/:/mnt \ splicedice_analysis:latest \ splicedice quant -m ${this_base_dir}/analysis/quant_manifest.txt \ -o ${this_base_dir}/analysis/ date ~/alert_msg.sh "quant run complete"



claude

Here's the walkthrough:

**1. Enter the container interactively**

```bash
docker run -it --rm \
    -v /mnt/:/mnt \
    splicedice_analysis:latest \
    bash
```

**2. Find and edit the file**

```bash
grep -n "inclusions / (inclusions + exclusions)" /usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py
```

Note the line number, then open it:

```bash
vi /usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py
```

Navigate to the line and replace:

```python
psi[self.junctionIndex[junction],:] = inclusions / (inclusions + exclusions)
```

with:

```python
denom = inclusions + exclusions
psi[self.junctionIndex[junction],:] = np.where(denom > 0, inclusions / denom, np.nan)
```

**3. Run quant on your mini manifest and save output to a new location**

```bash
splicedice quant \
    -m ${this_base_dir}/analysis/quant_manifest.txt \
    -o ${this_base_dir}/analysis/quant_nanfix_test/
```

**4. Compare output to existing**

```bash
diff ${this_base_dir}/analysis/_allPS.tsv \
     ${this_base_dir}/analysis/quant_nanfix_test/_allPS.tsv
```

If the only differences are NaN where there were previously 0 or inf values, the fix is working correctly. If diff returns nothing, the outputs are identical.

**5. Exit the container**

```bash
exit
```

Note that edits inside the container are lost when you exit — the fix is just for testing. Once confirmed, you'd apply it permanently either by patching the installed file in the Dockerfile, or if you have access to the splicedice source, committing it there.



## me

write a python command to edit the code isntead of using vi



## claude



```bash
python3 -c "
import re
path = '/usr/local/lib/python3.8/site-packages/splicedice/SPLICEDICE.py'
old = 'psi[self.junctionIndex[junction],:] = inclusions / (inclusions + exclusions)'
new = 'denom = inclusions + exclusions\n        psi[self.junctionIndex[junction],:] = np.where(denom > 0, inclusions / denom, np.nan)'
text = open(path).read()
assert old in text, 'pattern not found!'
open(path, 'w').write(text.replace(old, new))
print('done')
"
```

