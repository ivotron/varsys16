---
title: "The Predictable Container: Towards Reproducing System 
Performance Behavior Across Different Hardware Platforms with OS-level 
Virtualization"
author:
  - name: "Ivo Jimenez and Carlos Maltzahn"
    affiliation: "_UC Santa Cruz_"
    email: "`{ivo,carlosm}@cs.ucsc.edu`"
  - name: "Jay Lofstead"
    affiliation: "_Sandia National Laboratories_"
    email: "`gflofst@sandia.gov`"
  - name: "Adam Moody and Kathryn Mohror"
    affiliation: "_Lawrence Livermore National Laboratory_"
    email: "`{moody20,kathryn}@llnl.gov`"
#  - name: "Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau"
#    affiliation: "_University of Wisconsin-Madison_"
#    email: "`{remzi,dusseau}@cs.wisc.edu`"
abstract: |
  Evaluating experimental results in the field of systems research is 
  a challenging task, mainly due to the many changes in software and 
  hardware that computational environments go through. In this 
  position paper, we propose making use of some isolation features 
  that OS-level virtualiztion offers to mitigate the variability 
  observed when validating application performance accross multiple 
  hardware platforms. In concrete, we make the case for using CPU 
  affinity, CPU usage limitting and memory bandwidth throttling to 
  recreate original hardware conditions. We show promising perliminary 
  results that illustrate the effectiveness of this approach. We 
  additionally outline possible challenges and discuss alternative 
  future solutions.
documentclass: ieeetran
classoption: conference
ieeetran: true
numbersections: true
substitute-hyperref: true
csl: "ieee.csl"
bibliography: "citations.bib"
twocolumn-longtable: true
usedefaultspacing: true
fontfamily: times
conferencename: VarSys2016
copyrightyear: 2016
---

# Introduction

A key component of the scientific method is the ability to revisit and 
reproduce previous experiments. Reproducibility also plays a major 
role in education since a student can learn by looking at provenance 
information, re-evaluate the questions that the original experiment 
answered and thus "stand on the shoulder of giants". In the wide field 
of computer systems research, the issues of _reproducibility_ manifest 
when we validate system performance. In order to validate a claim in 
our field, we need to show that the system performs as stated, 
possibly in a variety of different scenarios.

When evaluating system performance, multiple variables[^unknowns] need 
to be accounted for, among them: source code changes, compilation and 
application configuration, as well as workload properties and hardware 
characteristics. A rule of thumb while executing experiments is: 
across multiple runs, modify only one variable at a time so that 
correlation can be accurately attributed to the right variable. 
Generally, one can control for changes in all the aforementioned 
variables but the latter one (the hardware). In an ideal scenario, we 
would like to reproduce the behavior of the system being evaluated, 
even when going across different hardware platforms.

[^unknowns]: Other _unknown_ variables might affect performance such 
as room temperature that are very hard to account for. In our case, 
the applications under study don't depend on some of these external 
factors (see Discussion section).

In this work we look at OS-level virtualization (a.k.a. container 
technology) as a way of mitigating the variability that presents 
across multiple executions of an application. OS-level virtualization 
offers several features that come handy when reproducing system 
performance. While core affinity and memory/swap size limitations have 
been shown to bring stability across executions in one single system 
[@corepining_2013], the use of CPU and memory bandwidth limitations 
can bring stability for executions across different platforms.

<!-- can we mention here that cpu/mem bandwidth limiation hasn't been 
studied in the context of bringing stability? -->

The rest of the paper is organized as follow. First, we describe the 
features and limitations of OS-level virtualization as a way of 
recreating hardware characteristics among distinct platforms. Then, we 
introduce our proposed methodology and show preliminary results of 
this approach. We discuss general issues and other high-level 
proposals. We close with a brief review of related work and final 
remarks.

# OS-level Virtualization

OS-level virtualization [@soltesz_container-based_2007] is a server 
virtualization method where the kernel of an operating system allows 
for multiple isolated user space instances, instead of just one. Such 
instances (often called containers, virtualization engines (VE), 
virtual private servers (VPS), or jails) may look and feel like a real 
server from the point of view of its owners and users. In addition to 
isolation mechanisms, the kernel often provides resource management 
features to limit the impact of one container's activities on the 
other containers. Container technology is currently employed as a way 
of reducing the complexity of software deployment and portability of 
applications in cloud computing infrastructure. Arguably, containers 
have taken the role that package management tools had in the past, 
where they were used to control upgrades and keep track of change in 
the dependencies of an application [@di_cosmo_package_2008]. In the 
remaining of this paper we focus on Docker [@docker_merkel_2014], 
which employs Linux's cgroups [@corbet_process_2007] and namespaces 
features. While our discussion is centered around cgroups, the overall 
strategies can be applied to any of the other OS-level virtualization 
technology mentioned above.

`cgroups` is a unified interface to the system's resource management 
options, allowing users to specify how the kernel should limit, 
account and isolate usage of CPU, memory, disk I/O and network of a 
collection of processes. In our case, we're initially interested in 
the CPU and memory limiting capabilities.

## CPU Usage Limitation

`cgroups` exposes parameters for the Completely Fair Scheduler (CFS) 
[@corbet_cfs_2007]. The allocation of CPU for a group can be given in 
relative (`shares`) or absolute (`period` and `quota`) values. Figure 
1 shows the effect that multiple values for `quota` have on the 
execution of a CPU-bound process.

![Boxplot of crafty runtimes for multiple values of cpu quota (with a 
fixed period of 100 microseconds). This illustrates the effectiveness 
of the CFS scheduler for limiting CPU access for a single-threaded 
process executing the crafty benchmark. Every boxplot summarizes 10 
executions. As the figure shows, the interquartile box overlaps with 
the median.](figures/cgroups.png)

## Memory Bandwidth Throttling

With `cgroups` one can specify constraints on the amount of memory and 
swap space that a group has, a useful feature that prevents 
applications from greedily using all its available memory. However, 
cgroups does not allow to impose limits on memory bandwidth in the 
same way that CPU usage can be constrained. To achieve this, we make 
use of memguard [@yun_memguard_2013], a kernel module that imposes 
absolute and relative limitations on memory bandwidth, in a similar 
way in which the CFS scheduler manages CPU usage. Figure 2 shows the 
effect of distinct memory bandwidth limitation values on a 
memory-bound workload.

![Boxplot of bandwidth test results for the STREAM (copy) benchmark 
for multiple values of cpu bandwidth limits. The figure shows the 
effectiveness of `memguard` for limiting available memory bandwidth to 
the single-threaded process executing the benchmark. Every point 
summarizes 10 executions of the benchmark. The interquartile box 
overlaps with the median.](figures/memguard.png)

# Proposed Methodology

We now present a methodology that leverages CPU and memory bandwidth 
limitations to enable the reproduction of performance across distinct 
hardware platforms[^porta]. When reproducing performance of an 
application on a target system $B$ that originally ran on a base 
system $A$, we propose the following mapping methodology:

  1. Execute microbenchmarks on $A$ that characterize the underlying 
     hardware platform.
  2. Manipulate absolute values for CFS and memguard on target system 
     $B$ in such a way that the performance of microbenchmarks is 
     reproduced; a configuration $C_b$ is obtained.
  3. Recreate the resource allocation of base system $A$ by applying 
     configuration $C_b$ on system $B$ and execute the application, 
     which should observe similar performance of when it executed on 
     $A$.

If the original execution of the application on the base system $A$ 
was itself being constrained, then this configuration $C_a$ should be 
applied in step 1. While we believe this methodology applies to many 
scenarios, we currently have tested it on single-threaded, non-DMA and 
non-collocated workloads (see _Preliminary Results_ section).

[^porta]: An implementation of this methodology for Docker containers 
is available at <https://github.com/systemslab/porta>.

## Tuning The Target System

Finding the values of CPU and memory bandwidth (step 2) is done via 
program auto-tunning [@ansel_opentuner_2014] for one or more 
microbenchmarks. Since we assume non-DMA accesses, the constrain of 
CPU bandwidth directly influences the effects of memory bandwidth 
limits[^membw], thus we first tune the value for CPU quota. It is 
reasonable to assume that the performance of CPU and memory correspond 
to a monotonically increasing function (as shown in Figures 1 and 2), 
thus, we can select a random value within the validate tunable range 
(or alternatively the highest) and climb/descend until we get to the 
desired target performance. Once the value for `cpu_quota` is found, 
we tune one or more memory-bound benchmarks in order to find the 
target value for memory bandwidth (expressed in MB/s in this case).

[^membw]: Memory bandwidth limits, while expressed in MB/s are 
translated into a quota of memory-related operations that a process 
group can execute per period, thus, the lower the quota for _any_ CPU 
operation, the lower the number of memory-related operations.

Our methodology assumes that the system where an application is being 
ported to (system $B$) is in general more powerful that the one were 
an application originally ran. When this assumption does not hold, one 
can resort to constraining the original execution (i.e. generating a 
$C_a$ for $A$).

It is important to note that the goal of our approach is to influence 
the performance of end-to-end applications rather than performance of 
a specific subsystem. For the latter, microbenchmarking exists and 
should be employed. In other words, we envision microbenchmarks as the 
tuning units while complete-stack applications as the targets.

# Preliminary Results

We show preliminary results for 4 applications. While Docker makes 
binary reproducibility possible, the compiler options used at the time 
that the docker image was created can influence the behavior, 
depending on where it was built, and where it is executed. Thus, to 
provide meaningful comparisons we have disabled compiler optimizations 
(`gcc`'s `-O0` flag) for all of the evaluated applications. Also, as 
mentioned previously, these are single-threaded, running in an 
uncontended system.

\begin{table}[ht]
\caption{Components of original and reproduced environments of the scalability experiment.}

\scriptsize
\centering
\begin{tabular}{@{} c c c @{}}
\toprule

Component & Original & Reproduced \\\midrule

CPU         & AMD 2212 @2.0GHz      & Intel E5-2630 @2.3GHz \\
Disk drive  & Seagate ST3250620NS   & HP 6G 658071-B21 \\
Disk BW     & 58 MB/s               & 120 MB/s (15 MB/s limit) \\
Linux       & 2.6.9                 & 3.13.0 \\
Ceph        & commit from 2005      & 0.87.1 \\
Storage     & 26 nodes              & 12 nodes \\
Clients     & 20 nodes              & 1 node \\
Network     & Netgear GS748T        & Same as original \\
Network BW  & 1400 MB/s             & 110 MB/s \\

\bottomrule
\end{tabular}
\end{table}

## Applications and Hardware Setup

We use the following applications:

  * CoMD. A reference implementation of classical molecular dynamics 
    algorithms and workloads as used in materials science.
  * CloverLeaf. A hydrodynamics mini-app to solve the compressible 
    Euler equations in 2D, using an explicit, second-order method.
  * SILT [@lim_silt_2011]. A memory-efficient, high-performance 
    key-value store system based on flash storage that scales to serve 
    billions of key-value items on a single node.
  * RedisBench. The redis-benchmark utility that simulates running 
    commands done by N clients at the same time sending M total 
    queries.

All applications were executed on the base and target systems. The 
characteristics of every platform are described in Table 1[^paperepo]. 
Every application runs as a single-threaded process within a (possibly 
constrained) container.

[^paperepo]: For a complete description of each machine, as well as 
detailed software configuration, please refer to the repository of 
this article at <https://github.com/ivotron/varsys16>.

## Results And Analysis

Table 2 shows the results of our tests.

\begin{table}[ht]
\caption{Results.}

\scriptsize
\centering
\begin{tabular}{@{} c c c @{}}
\toprule

Benchmark   & without & limits \\\midrule

CoMD        & 1.635   & 0.992 \\
CloverLeaf  & 1.878   & 1.026 \\
SILT        & 3.306   & 0.642 \\
Redis (get) & 3.093   & 0.589 \\

\bottomrule
\end{tabular}
\end{table}

The reason why the memory-bound applications observe slowdowns is due 
to the way memguard throttles memory bandwidth. memguard uses "busy" 
throttling, which means that whenever a process goes above its 
allocated memory bandwidth quota, a real-time thread begins to make 
use of the CPU in order to limit its usage. This directly conflicts 
with the way cgroups and the CFS scheduler works.

# Discussion

## Why bother?

It would seem that trying to reproduce performance is only a 
time-consuming task without a tangible, productive output. If we have 
been doing research like this for 50 years, why bother? Also, the rate 
of change is hard to keep up with.

## Community-maintained Hardware Configuration Guidelines

Other aspects of the CPU that have an effect on performance and that 
we haven't modified: dynamic frequency scaling, hyperthreading. One 
possible solution is to create a community maintained list of 
guidelines that should be followed.

## A New Category of Experimental Evaluation

We also see how having new categories for reporting performance 
results can be helpful. Having "controlled" and "real-world" tests, 
where in the former a great amount of effort is devoted to ensure that 
important factors are accounted for. In the latter, we could have what 
we currently see in the literature but with sound statistical analysis 
[@hoefler_scientific_2015]

## Providing Microbenchmarks Alongside Experimental Results

This allows to preserve the characteristics of the underlying hardware 
and make it possible to reproduce the behavior in the future.

# Related Work

The challenging task of evaluating experimental results in applied 
computer science has been long recognized [@ignizio_establishment_1971 
; @crowder_reporting_1979]. This issue has recently received a 
significant amount of attention from the computational research 
community [@freire_computational_2012], where the focus is more on 
numerical reproducibility rather than performance evaluation. 
Similarly, efforts such as _The Recomputation Manifesto_ 
[@gent_recomputation_2013] and the _Software Sustainability Institute_ 
[@crouch_software_2013] have reproducibility as a central part of 
their endeavour but leave runtime performance as a secondary problem. 
In systems research, runtime performance _is_ the subject of study, 
thus we need to look at it as a primary issue.

In [@collberg_measuring_2014] the authors took 613 articles published 
in 13 top-tier systems research conferences and found that 25% of the 
articles are reproducible (under their reproducibility criteria). The 
authors did not analyze performance. In our case, we are interested 
not only in binary reproducibility (the ability to run the same binary 
as in the original setting) but also in reproducing performance 
behavior.

The closest work to our approach is Fracas [@fracas]. Fracas emulates 
CPU frequency for a single system. The problem is that frequency is 
not a reliable baseline. Instead, we take the performance of an 
application on the original system as our baseline and try to 
reproduce in other target systems, irrespective of the differences 
between frequencies. By relying on wall-clock time usage 
(period/quota) instead of tyring to map frequencies, we can get better 
results. Additionally, we consider memory bandwidth, which, as shown 
in [@fracas], has to be considered in order to be able to replicate 
performance on the same system.

# Conclusion

In the future we will like to include more applications and relax the 
constrains that we impose such as supporting multi-threaded 
applications and collocated workloads.

# References

<!-- hanged biblio -->

\noindent
\vspace{-2em}
\setlength{\parindent}{-0.26in}
\setlength{\leftskip}{0.2in}
\setlength{\parskip}{8pt}
