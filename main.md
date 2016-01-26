---
title: "Reducing The Variability Range Across Distinct Hardware 
Platforms with OS-level Virtualization"
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
  hardware that computational environments go through. In this paper, 
  we propose making use of some isolation features that OS-level 
  virtualiztion offers to mitigate the variability observed when 
  validating application performance accross multiple hardware 
  platforms. In concrete, we make the case for using CPU affinity and 
  bandwith limitations to reduce the variability associated to the 
  execution of a CPU-bound application between different hardware 
  platforms. Our results show that the variability can be reduced by 
  up to an order of magnitude, depending on its opcode mix. We outline 
  remaining challenges in this topic and also discuss issues related 
  to the state of the practice.
documentclass: ieeetran
classoption: conference
ieeetran: true
numbersections: true
substitute-hyperref: true
csl: "ieee.csl"
bibliography: "citations.bib"
links-as-notes: true
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
when we validate system performance. In order to validate a claim, we 
need to show that the system performs as stated, possibly in a variety 
of different scenarios.

When evaluating system performance, multiple variables[^unknowns] need 
to be accounted for, among them: source code changes, compilation and 
application configuration, as well as workload properties and hardware 
characteristics. A rule of thumb while executing experiments is: 
across multiple runs, modify only one variable at a time so that 
correlation can be accurately attributed to the right variable. 
Generally, one can control for changes in all the aforementioned 
variables but the latter one (the hardware). In an ideal scenario, we 
would like to control the behavior of the system being evaluated in a 
predictable manner, even when going across different hardware 
platforms.

[^unknowns]: Other _unknown_ variables might affect performance such 
as room temperature that are very hard to account for. In our case, 
the applications under study don't depend on some of these external 
factors (see Discussion section).

In this work we look at OS-level virtualization (a.k.a. container 
technology) as a way of mitigating the variability that presents 
across multiple executions of a system. OS-level virtualization offers 
several features that come handy when reproducing system performance. 
While core affinity and memory/swap size limitations have been shown 
to bring stability across executions in one single system 
[@corepining_2013], the use of CPU bandwidth limitation can bring 
stability for executions across different platforms.

<!-- can we mention here that cpu/mem bandwidth limiation hasn't been 
studied in the context of bringing stability? -->

The rest of the paper is organized as follow. First, we briefly 
introduce OS-level virtualization and describe the features that are 
of interest for our purposes, as well as their fundamental 
limitations. Then, we introduce our proposed methodology and show 
preliminary results of this approach. Subsequently, we discuss general 
issues and other high-level proposals. Lastly, we close with a brief 
review of related work and final remarks.

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

`cgroups` is a unified interface to the operating system's resource 
management options, allowing users to specify how the kernel should 
limit, account and isolate usage of CPU, memory, disk I/O and network 
of a collection of processes. In our case, we're initially interested 
in the CPU and memory limiting capabilities.

## CPU Bandwidth Throttling

`cgroups` exposes parameters for the Completely Fair Scheduler (CFS) 
[@corbet_cfs_2007]. The allocation of CPU for a group can be given in 
relative (`shares`) or absolute (`period` and `quota`) values. Figure 
1 shows the effect that multiple values for `quota` have on the 
execution of a CPU-bound process.

## Limitations of Throttling

While absolute CPU bandwidth limitations work well to isolate groups 
within a single system, they don't work well while going across 
multiple platforms. The fundamental differences between distinct 
machines (e.g. CPU features, memory bandwidth, etc.) make it 
practically impossible to find a single value for `quota` that works 
for _every_ application. One can tune the quota/period to reproduce 
results for a particular application, but these values won't work for 
another application with different opcode mix.

![Boxplot of crafty runtimes for multiple values of cpu quota (with a 
fixed period of 100 microseconds). This illustrates the effectiveness 
of the CFS scheduler for limiting CPU access for a single-threaded 
process executing the crafty benchmark. Every boxplot summarizes 10 
executions. As the figure shows, the interquartile box overlaps with 
the median.](figures/cgroups.png)

# Quantifying CPU Variability Ranges

In order to validate our approach, we need a way of visualizing the 
CPU variability range between two machines. Ideally, we would like to 
have programs that execute the same opcode over and over again and we 
would then characterize their performance, at the level of the OS 
(e.g. the amount of CPU utilization over time). Since we can't have 
programs that execute one single operation, we need to have 
microbenchmarks that get as close as possible to exercising distinct 
features of a CPU.

[stress-ng](https://github.com/ColinIanKing/stress-ng) is a tool that 
is used to "stress test a computer system in various selectable ways. 
It was designed to exercise various physical subsystems of a computer 
as well as the various operating system kernel interfaces". The `cpu` 
"stressor" is a routine that loops a function (or `cpu-method`) 
multiple times and reports the rate of iterations executed for a 
determined period of time (referred to as `bogo-ops-per-second`). 
There are 68 CPU methods in total that range from bitwise operations, 
to floating and integer operations on multiple word sizes.

![Histogram of speedups. There are 68 points in total. Each point 
corresponds to the performance speedup of an cpu method of stress-ng 
when executed on a target machine w.r.t. a base one. For example, the 
speedup caused by improvements in the hardware of the target machine 
causes 7 stressors to have a speedup within the `[1.75, 2)` 
range.](figures/unconstrained_range.png)

Using this battery of cpu stressors, one can obtain a "spectrum" of 
CPU performance. When compared against the execution of stress-ng on 
another machine, one can obtain a variability range. Figure 2 shows 
the histogram whose range correspond to the variability range of 
stress-ng CPU stressors for two machines (see Table 1). As mentioned 
earlier, our goal is to reduce this range so that an application has 
less wiggle room for performance variations.

# Proposed Methodology

We now present a methodology that leverages the CPU bandwidth 
limitation feature of OS-level virtualization to reduce the 
variability range of performance across distinct hardware 
platforms[^porta]. When reproducing performance of an application on a 
target machine $B$ that originally ran on a base machine $A$, we 
propose the following mapping methodology:

  1. Execute microbenchmarks on $A$ that characterize the underlying 
     hardware platform.
  2. Manipulate absolute cgroup values for the CFS on target machine 
     $B$ in such a way that the performance of microbenchmarks is as 
     close as possible to the results from machine $A$; a 
     configuration $C_b$ is obtained.
  3. Apply the configuration $C_b$ on machine $B$ and execute the 
     application, which should observe reduced performance variability 
     when compared against the unconstrained execution on $B$.

If the original execution of the application on the base machine $A$ 
was itself being constrained, then this configuration $C_a$ should be 
applied in step 1. While we believe this methodology applies to many 
scenarios, we currently have tested it on single-threaded and 
non-collocated workloads (see _Results_ section).

[^porta]: An implementation of this methodology for Docker containers 
is available at <https://github.com/systemslab/porta>.

## Tuning The Target Machine

Finding the values of CPU bandwidth (step 2) is done via program 
auto-tunning [@ansel_opentuner_2014] for one or more microbenchmarks 
that characterize the performance of the underlying hardware. It is 
reasonable to assume that the performance of CPU corresponds to a 
monotonically decreasing function (as in Figure 1), thus, we can 
select a random value within the validate tunable range (or 
alternatively the highest) and climb/descend until we get to the 
desired target performance for the microbenchmark(s). When multiple 
microbenchmarks are executed, their results need to be aggregated 
(e.g. by taking a weighted average of a speedup metric).

Our methodology assumes that the machine where an application is being 
ported to (system $B$) is relatively more powerful that the one were 
an application originally ran[^cantspeeduphardware]. When this 
assumption does not hold, one can resort to constraining the original 
execution (i.e. generating a $C_a$ for $A$).

[^cantspeeduphardware]: If machine $B$ is slower than $A$, there's no 
way one can speedup $A$. Overclocking may be an alternative but we 
think it's easier to just slowdown machine $A$ instead, so that a 
common-denominator between the two can be found.

# Results

We show the effectiveness of our proposed methodology by evaluating it 
on multiple machines. We also empirically corroborate our findings on 
multiple applications. Table 1 shows the hardware setup for our 
experiments. Both machines run Ubuntu Server 14.04.3.

\begin{table}[ht]
\caption{Components of base and target machines used in our study.}

\scriptsize
\centering
\begin{tabular}{@{} c c c @{}}
\toprule

Component & Base & Target \\\midrule

CPU         & AMD 2212 @2.0GHz & Intel E5-2630 @2.3GHz \\
Memory      & DDR2@   & DDR3@ \\
Year        & 2006    & 2014

\end{tabular}
\end{table}

## Reduction of Variability Range

Figure 3 illustrates the reduction in the variability range for the 
two machines from Table 1. The tuning for the target machine resulted 
in a cpu quota of `6372` microseconds for a period of `10000` 
microseconds. When these bandwidth limitations are applied, the range 
reduces to the one shown on the histogram in Figure 3. As we can see, 
the variability range is reduced from \[1.5-14\] (in Figure 2) to 
\[0.7,2\], i.e. from a range of size 12.5 to one of size 1.3. Assuming 
stress-ng's distinct CPU methods represent a realistic coverage of the 
multiple physical features of a processor, we can reasonably assume 
that the performance of applications under constrained CPU bandwidth 
will land within this range. In order to prove this hypothesis, we 
conducted an empirical study.

![Histogram of speedups for constrained executions. When compared 
against the histogram in Figure 2, we see how the range has been 
reduced.](figures/constrained_range.png)

## Empirical Validation

To validate our approach, we executed 25 application with and without 
CPU bandwidth limitations on the target system. Due to space 
constrains we omit the description of what each application 
does[^paperepo]. Every application was executed on the base and target 
systems (Table 1) in docker containers. While docker makes binary 
reproducibility possible, the compiler options used at the time that a 
docker image is created can significantly influence the behavior, 
depending on where it was built, as well as where it is executed. In 
order to discard the variability that might originate from distinct 
compiler optimizations, and as a way of making meaningful comparisons, 
we disabled compiler optimizations (`gcc`'s `-O0` flag) for all of the 
evaluated applications. Also, as mentioned previously, these are 
single-threaded processes running in uncontended systems.

Table 2 shows the results of our tests. As we can see, no application 
goes out of the (reduced) validation range. We also observe that, 
although the highest value from stress-ng in the unconstrained 
scenario is approximately 14x the baseline, the performance of 
applications in this study never exceed 4x. We conjecture that this is 
due to the fact that no application in our evaluation resembles the 
opcode mix for the method in bin 14, which happens to be an `ackerman` 
stressor, executing ...

[^paperepo]: For a complete description of each machine, as well as 
detailed software configuration and workloads used please refer to the 
repository of this article at <https://github.com/ivotron/varsys16>.

\begin{table}[ht]
\caption{Results.}

\scriptsize
\centering
\begin{tabular}{@{} c c c c c c @{}}
\toprule

Benchmark   & without & limits & without & limits \\\midrule

CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\
CloverLeaf  & 1.878   & 1.026 & 1.878 & 1.026 \\
SILT        & 3.306   & 0.642 & 3.306 & 0.642 \\
Redis (get) & 3.093   & 0.589 & 3.093 & 0.589 \\
CoMD        & 1.635   & 0.992 & 1.635 & 0.992 \\

\bottomrule
\end{tabular}
\end{table}

# Discussion

## Why bother?

It would seem that trying to reproduce performance is only a 
time-consuming task without a tangible, productive output. If we have 
been doing research like this for 50 years, why bother? Also, the rate 
of change is hard to keep up with.

## Community-maintained Hardware Configuration Guidelines

Other aspects of the CPU that have an effect on performance and that 
we haven't modified in our case: dynamic frequency scaling, 
hyperthreading, Intel's turboboost, Linux runtime configuration, among 
others. All these influence the outcome of system performance. We 
consider two possible solutions. (1) make all this information part of 
performance results, so that others can have access to it and (2)
create a community maintained list of guidelines that should be 
followed (and automate its applications, e.g. via CloudLab/PRoBe 
system images).

## A New Category of Experimental Evaluation

We also see how having new categories for reporting performance 
results can be helpful. In real-world scenarios, we are interested in 
executing applications without any constraint so that they can take 
advantage of all the resources available to them. Distinguishing 
between "controlled" and "real-world" tests can be of help. In the 
former, a great amount of effort is devoted to ensure that important 
factors are accounted for. In the latter, we could have what we 
currently see in the literature but with sound statistical analysis 
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
