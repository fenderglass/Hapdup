# Hapdup

Hapdup (haplotype duplicator) is a pipeline to convert a haploid Oxford Nanopore assembly into a [dual](http://lh3.github.io/2021/10/10/introducing-dual-assembly) diploid assembly.
The reconstructed haplotypes preserve heterozygous structural variants (in addition to small variants) and
are locally phased.


## Version 0.2

Input requirements
------------------

Hapdup takes as input a long-read assembly, such as produced with [Flye](https://github.com/fenderglass/Flye) or 
[Shasta](https://github.com/chanzuckerberg/shasta). The assembly is assumed to be hapoid, alternative
alleles could be removed prior to running the pipeline using [purge_dups](https://github.com/dfguan/purge_dups).

The first stage is to realign the original long reads
on the assembly using [minimap2](https://github.com/lh3/minimap2). We recommend to use the latest minimap2 release.

```
minimap2 -ax map-ont assembly.fasta reads.fastq | samtools sort -@ 4 -m 4G > lr_mapping.bam
samtools index -@ 4 assembly_lr_mapping.bam
```

Quick start using Docker
------------------------

Hapdup is available on the [Docker Hub](https://hub.docker.com/repository/docker/mkolmogo/hapdup).

If Docker is not installed in your system, you need to set it up first following this [guide](https://docs.docker.com/engine/install/ubuntu/).

Next steps assume that your `assembly.fasta` and `lr_mapping.bam` are in the same directory,
which will also be used for hapdup output. If it is not the case, you might need to bind additional 
directories using the Docker's `-v / --volume` argument. The number of threads (`-t` argument)
should be adjusted according to the available resources.

```
cd directory_with_assembly_and_alignment
HD_DIR=`pwd`
docker run -v $HD_DIR:$HD_DIR -u `id -u`:`id -g` mkolmogo/hapdup:0.2 \
  hapdup --assembly $HD_DIR/assembly.fasta --bam $HD_DIR/lr_mapping.bam --out-dir $HD_DIR/hapdup -t 64
```

Quick start using Singularity
-----------------------------

Alternatively, you can use [Singularity](https://sylabs.io/guides/3.5/user-guide/). First, you will need install
the client as descibed in the manual. One way to do it is through conda:

```
conda install singularity
```

Next steps assume that your `assembly.fasta` and `lr_mapping.bam` are in the same directory,
which will also be used for hapdup output. If it is not the case, you might need to bind additional 
directories using the `--bind` argument. The number of threads (`-t` argument)
should be adjusted according to the available resources.

```
singularity pull docker://mkolmogo/hapdup:0.2
HD_DIR=`pwd`
singularity exec --bind $HD_DIR hapdup_0.2.sif \
  hapdup --assembly $HD_DIR/assembly.fasta --bam $HD_DIR/lr_mapping.bam --out-dir $HD_DIR/hapdup -t 64
```

Output files
------------

The output directory will contain:
* `haplotype_{1,2}.fasta` - final assembled haplotypes
* `phased_blocks_hp{1,2}.bed` - phased blocks coordinates

Haplotypes generated by the pipeline contain homozogous and heterozygous varinats (small and structural).
Becuase the pipeline is only using long-read (ONT) data, it does not achieve chromosome-level phasing.
Fully-phased blocks are given in the the `phased_blocks*` files.


Pipeline overview
-----------------

1. hapdup starts with filtering alignments that are likely originating from the unassembled parts of the genome.
Such alignments may later create false haplotypes if not removed (e.g. if reads from a segmental duplication with two copies
can create four haplotypes).

2. Afterwards, PEPPER is used to call SNPs from the filtered alignment file

3. Then we use Margin to phase SNPs and haplotype reads

4. We then use Flye to polish the initiall assembly with the reads from each of the two
haplotypes independently

5. Finally, we find (heterozygous) breakpoints in long-read alignments and apply
the corresponding structural changes to the corresponding polished haplotypes.
Currently, it allows to recover large heterozygous inversions.

Benchmarks
----------

We evaluated hapdup haplotypes in terms of reconstructed structural variants signatures (heterozygous & homozygous)
using the HG002 for which the [curated set of SVs](https://www.nature.com/articles/s41587-020-0538-8) 
is available. We used the [recent ONT data](https://s3-us-west-2.amazonaws.com/miten-hg002/index.html?prefix=guppy_5.0.7/) 
basecalled with Guppy 5.

Given hapdup haplotypes, we called SV using [dipdiff](https://github.com/fenderglass/dipdiff). We also compare SV
set against hifiasm assemblies, even though they were produced from HiFi, rather than ONT reads.
Evaluated using truvari with `-r 2000` option. GT refers to genotype-considered benchmarks.


| Method         | Precision | Recall | F1-score | GT Precision | GT Recall | GT F1-score |
|----------------|-----------|--------|----------|--------------|-----------|-------------|
| Shasta+Hapdup  |  0.9500   | 0.9551 | 0.9525   | 0.934        | 0.9543    |  0.9405     |
| Sniffles       |  0.9294   | 0.9143 | 0.9219   | 0.8284       | 0.9051    |  0.8605     |
| CuteSV         |  0.9324   | 0.9428 | 0.9376   | 0.9119       | 0.9416    |  0.9265     |
| hifiasm        |  0.9512   | 0.9734 | 0.9622   | 0.9129       | 0.9723    |  0.9417     |

Yak k-mer based evaluations:

| Hap   |  QV  | Switch err | Hamming err |
|-------|------|------------|-------------|
|     1 |  35  |   0.0389   |   0.1862    |  
|     2 |  35  |   0.0385   |   0.1845    |

Given a minimap2 alignment, hapdup runs in ~400 CPUh and uses ~80 Gb of RAM.

Source installation
-------------------

Here are the instructions to build a hapdup docker image locally.
To install directly into the system, please see `Dockerfile` for the command lines.

```
git clone https://github.com/fenderglass/Hapdup
cd Hapdup
git submodule update --init --recursive
docker build -t hapdup .
```

Acknowledgements
----------------

The major parts of the hapdup pipeline are:

* [PEPPER](https://github.com/kishwarshafin/pepper)
* [Margin](https://github.com/UCSC-nanopore-cgl/margin)
* [Flye](https://github.com/fenderglass/Flye)


Authors
-------

The pipeline was developed at [UC Santa Cruz genomics institute](https://ucscgenomics.soe.ucsc.edu/), Benedict Paten's lab.

Pipeline code contributors:
* Mikhail Kolmogorov

PEPPER/Margin/Shasta support:
* Kishwar Shafin
* Trevor Pesout
* Paolo Carnevali

Citation
--------

If you use hapdup in your research, the most relevant papers to cite are:

Kishwar Shafin, Trevor Pesout, Pi-Chuan Chang, Maria Nattestad, Alexey Kolesnikov, Sidharth Goel, Gunjan Baid et al. 
"Haplotype-aware variant calling enables high accuracy in nanopore long-reads using deep neural networks." bioRxiv (2021).
[doi:10.1101/2021.03.04.433952](https://doi.org/10.1101/2021.03.04.433952)


Mikhail Kolmogorov, Jeffrey Yuan, Yu Lin and Pavel Pevzner, 
"Assembly of Long Error-Prone Reads Using Repeat Graphs", Nature Biotechnology, 2019
[doi:10.1038/s41587-019-0072-8](https://doi.org/10.1038/s41587-019-0072-8)

License
-------

hapdup is distributed under a BSD license. See the [LICENSE file](LICENSE) for details.
Other software included in this discrubution is released under either MIT or BSD licenses.


How to get help
---------------
A preferred way report any problems or ask questions is the 
[issue tracker](https://github.com/fenderglass/hapdup/issues). 

In case you prefer personal communication, please contact Mikhail at fenderglass@gmail.com.
