# Abstract

[CRISPR](https://en.wikipedia.org/wiki/CRISPR) screens are powerful experimental tools used to screen entire genomes in search of genes responsible for [phenotypes](https://en.wikipedia.org/wiki/Phenotype) of interest. With this approach, a single experiment can generate several gigabytes of data that, with sequential implementation, can take many hours to process, limiting [the](https://en.wikipedia.org/wiki/The) depth of sequencing and amount of analysis that can feasibly be performed. Here, we apply principles of parallel computing and algorithm design to expedite the data processing and analysis pipeline significantly. In doing so, we create a framework that provides three important features: i) Facilitation of considerably deeper sequencing experiments through parallelized expedition ii) integration of a sequence distance analysis for improved screening results iii) An associated cost-performance analysis for achieving the desired computational power within given financial constraints. 

# Introduction

The advent of technological advancements such as high-throughput sequencing and genome engineering, along with the increase in available computational power, has allowed biologist to adopt experimental approaches that create millions, sometimes even billions of data points per experiment

## Problem Description

The output of the biological experiment for CRISPR genetic screens is two files of DNA sequences:<br>
1) one file contains the DNA sequences from the control population of cells, which we will call the *control file*
2) the second file contains the DNA sequences from cells that were selected for some phenotype of interest, which we will call the *experimental file*

## Existing Pipeline

TO DO

* * *

# Project Design

# Sequential Code Profiling

The sequential code `count_spacers_with_ED.py` was profiled using the `cProfile` Python package. The file was run with a control file of 100 sequences (*Genome-Pos-3T3-Unsorted_100_seqs.txt*) and an experimental file of 100 sequences (*Genome-Pos-3T3-Bot10_100_seqs.txt*). Each of these input files contained 75 sequencing reads that could be perfectly matched to the database of 80,000 guide sequences and 25 sequencing reads that needed an edit distance calculation. This breakdown was representative of the proportion of sequencing reads in the full input files; ~25% of sequencing reads cannot be perfectly matched to one of the 80,000 guide sequences. The exact command that was run was: `python -m cProfile -o 100_seq_stats.profile count_spacers_with_ED.py -g ../data/Brie_CRISPR_library_with_controls_FOR_ANALYSIS.csv -u ../data/Genome-Pos-3T3-Unsorted_100_seqs.txt -s ../data/Genome-Pos-3T3-Bot10_100_seqs.txt -o cProfile_test_output`. This code was run on a Macbook Pro, with a 2.2 GHz Intel Core i7 processor with 6 cores. The profiling information was saved in a file called *100_seq_stats.profile*. The following results are from the `pstats` package.
```python
import pstats

p = pstats.Stats('100_seq_stats.profile'); #read in profiling stats
p.strip_dirs(); #remove the extraneous path from all the module names

#sort according to time spent within each function, and then print the statistics for the top 20 functions. 
p.sort_stats('time').print_stats(20)

```
	Mon Apr 29 16:08:45 2019    100_seq_stats.profile
	         
	         350132307 function calls (350126604 primitive calls) in 537.388 seconds

	   Ordered by: internal time
	   List reduced from 1999 to 20 due to restriction <20>

	   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
	  3981650  462.315    0.000  534.561    0.000 count_spacers_with_ED.py:35(editDistDP)
	333437825   69.985    0.000   69.985    0.000 {built-in method builtins.min}
	  3982102    1.819    0.000    1.819    0.000 {built-in method numpy.zeros}
	        2    1.414    0.707  535.981  267.990 count_spacers_with_ED.py:69(count_spacers)
	7992171/7992125    0.447    0.000    0.447    0.000 {built-in method builtins.len}
	        1    0.215    0.215    0.232    0.232 count_spacers_with_ED.py:7(createDictionaries)
	    81/79    0.153    0.002    0.156    0.002 {built-in method _imp.create_dynamic}
	    20674    0.115    0.000    0.365    0.000 stats.py:3055(fisher_exact)
	      348    0.100    0.000    0.100    0.000 {method 'read' of '_io.FileIO' objects}
	    41950    0.092    0.000    0.092    0.000 {method 'reduce' of 'numpy.ufunc' objects}
	        1    0.075    0.075    0.075    0.075 {method 'dot' of 'numpy.ndarray' objects}
	      348    0.056    0.000    0.155    0.000 <frozen importlib._bootstrap_external>:830(get_data)
	    28062    0.052    0.000    0.052    0.000 {built-in method numpy.array}
	     1604    0.035    0.000    0.035    0.000 {built-in method posix.stat}
	      348    0.034    0.000    0.034    0.000 {built-in method marshal.loads}
	    593/1    0.033    0.000  537.389  537.389 {built-in method builtins.exec}
	        1    0.033    0.033    0.405    0.405 count_spacers_with_ED.py:135(calcGeneEnrich)
	    81/65    0.024    0.000    0.077    0.001 {built-in method _imp.exec_dynamic}
	    21078    0.021    0.000    0.071    0.000 fromnumeric.py:69(_wrapreduction)
	    20675    0.020    0.000    0.020    0.000 {method 'writerow' of '_csv.writer' objects}

The majority of runtime is spent with `editDistDP` function. 534 of the 537 seconds, which accounts for 99.4% of the runtime, are spent calculating the edit distance between 50 sequencing reads and 80,000 guides. Generally, the input files contain ~10M sequencing reads, and about 25% of the sequences cannot be matched perfectly to one of the 80,000 guides. Thus for two input files of ~10M sequencing reads (~20M reads total), there are ~4-5M sequencing reads for which the edit distance calculations must be performed. If this code was run sequentially, this would require 10,000 hours of runtime. Therefore, we need to parallelize this portion of the code.

The edit distance calculation is currently nested within the function `count_spacers`, which matches each sequencing read from the input files to one of the 80,000 guides. For 200 sequencing reads provided as input, 1.4 seconds are spent performing the matching. This is only 0.007 seconds per sequencing read (using the 1.4 seconds from the *tottime* column since the *cumtime* takes into account the edit distance calculation). This number grows large if we have 20M sequencing reads - it would take $\dfrac{0.007\text{seconds/read} \cdot 20\text{M reads}}{3600\text{seconds/hour}} = 39\text{hours}$. Thus, the entire matching process of our workflow needs to be parallelized.

We want to parallelize this matching process by using a Spark cluster to have access to as many cores as possible to perform both the matching process and edit distance calculation (if needed). We will partition each input file into many tasks, and each task will run on a single core of the Spark cluster. A single core will perform both the matching process and edit distance for the sequencing reads in a partition. From what we have determined, there is not an easy way to parallelize the edit distance calculation algorithm itself. However, for a given sequencing read, we should be able to parallelize the 80,000 edit distance calculations that need to be performed between the sequencing read and the guides by using Python multi-threading.

# Overheads 

Since we do not know for which sequences we will need to perform edit distance calculations, load-balancing is the main overhead we anticipate dealing with because we do not want one or two cores slowed down with having to compute too many edit distance calculations. We would like to spread the number of edit distance calculations out evenly between cores by tuning the number of Spark tasks. It may be good to shuffle the order of the sequencing reads because sometimes many sequences that require edit distance calculations are adjacent to each other in the input file.

If we try to use a GPU to perform the 80,000 edit distance calculations in parallel, memory-transfer (input/output) to the GPU would also be an overhead. For a single sequencing read, multiple transfers would need to be performed as we would not be able to perform the 80,000 calculations in parallel, since we are limited by the number of cores on the GPU. Currently, we do not have a good way of mitigating this GPU overhead.

# Scaling

The sequence matching and edit distance portion of our code accounts for 99.4% of the runtime in our small example. With larger problem sizes, this percentage should only increase because the number of operations performed after the sequence matching and edit distance section is constant.

Amdahl's Law states that potential program speedup $S_t$ is defined by the fraction of code $c$ that can be parallelized, according to the formula
$$
S_t = \dfrac{1}{(1-c)+\frac{c}{p}}
$$
where $p$ is the number of processors/cores. In our case, $c=0.994$ and the table below shows the speed-ups for $2$, $4$, $8$, $64$, and $128$ processors/cores:

|processors|speed-up|
|----------|--------|
|2|1.98x|
|4|3.93x|
|8|7.68x|
|64|46.44x|
|128|72.64x|

Thus, our strong-scaling is almost linear when $p$ is small, but we observe that this begins to break down because we only get 73x speed-up if we were to use 128 processors/cores.

Gustafson's Law states larger systems should be used to solve larger problems because there should ideally be a fixed amount of parallel work per processor. The speed-up $S_t$ is calculated by
$$
S_t = 1 - c +c\cdot p
$$
where $p$ is the number of processors/cores. In our case, $c=0.994$ and the table below shows the speed-ups for $2$, $4$, $8$, $64$, and $128$ processors/cores:

|processors|speed-up|
|----------|--------|
|2|1.994x|
|4|3.98x|
|8|7.96x|
|64|63.62x|
|128|127.23x|

Thus, we almost achieve perfect weak-scaling because we can split up larger problem-sizes (which would be larger input files in our case) over more processors to achieve about the same runtime.


## Data

TO DO

## Pipeline

TO DO

## Speedup Algorithm

TO DO


## Infrastructure

TO DO

* * *

# Usage 

TO DO

* * *

# Results

## Performance Evaluation

TO DO

## Optimizations and Overheads

TO DO

* * *

# Discussion

TO DO

# Future Work

* * *

# References

1. Pusapati GV, Kong JH, Patel BB, Krishnan A, Sagner A,
Kinnebrew M, Briscoe J, Aravind L, Rohatgi R: CRISPR screens
uncover genes that regulate target cell sensitivity to the
morphogen sonic hedgehog. Dev Cell 2018, 44:113-129 e118.