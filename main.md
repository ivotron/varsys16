---
title: "Predicting Cross-Platform Performance Variability Using 
OS-level Virtualization"
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
  - name: "Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau"
    affiliation: "_University of Wisconsin-Madison_"
    email: "`{remzi,dusseau}@cs.wisc.edu`"
abstract: |
  Evaluating experimental results in the field of systems research is 
  a challenging task, mainly due to changes and differences in 
  software and hardware in computational environments. In this paper, 
  we propose making use of isolation features that OS-level 
  virtualization offers to mitigate the performance variability 
  observed when validating application performance across multiple 
  hardware platforms. In concrete, we make the case for using CPU 
  bandwidth limitations to reduce the variability of a CPU-bound 
  application between different hardware platforms. We introduce an 
  architecture-independent way of quantifying variability within and 
  across machines. Our results show that the variability can be 
  reduced by up to an order of magnitude, depending on the opcode mix 
  of an application, as well as generational and architectural 
  differences between two hardware platforms.
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
to be accounted for, among them are source code changes, compilation 
and application configuration, as well as workload properties and 
hardware characteristics. A rule of thumb while executing experiments 
is: across multiple runs, modify only one variable at a time so that 
correlation can be accurately attributed to the right variable. 
Generally, it is easy to account for all experimental variables except 
the hardware. However, ideally we want to control all aspects of the 
behavior of the system being evaluated in a predictable manner, even 
across different hardware.

[^unknowns]: Other _unknown_ variables might affect performance such 
as room temperature that are very hard to account for. In our case, 
the applications under study don't depend on some of these external 
factors (see Discussion section).

In this work we investigate OS-level virtualization as a way of 
mitigating performance variability across multiple executions of an 
application. OS-level virtualization offers several features for 
reproducing system performance. While core affinity and memory/swap 
size limitations have been shown to bring stability across executions 
in one single system [@beyer_benchmarking_2015], the use of CPU 
bandwidth limitation can reduce the variability for executions across 
different platforms. Our experiments show that using CPU bandwidth 
limitation reduces performance variability by up to 8x, depending on 
the opcode mix of an application, as well as generational and 
architectural differences between two hardware platforms.

<!-- can we say that, we don't only show that by limiting bandwidth we 
reduce variabilty but we also bring predictability in the sense that 
if an app $a$ runs faster than $b$ on system $S_1$, then the same app 
will run faster on system $S_2$ after throttling both? Does results 
show this? If it does, then we can make the claim that our approach is 
equivalent to the issue of flash-vs-disk in the sense that we are 
controlling for bottlenecks moving between CPU subsystems.
-->

<!-- can we mention here that cpu/mem bandwidth limiation hasn't been 
studied in the context of bringing stability? -->

The rest of the paper is organized as follow. First, we briefly 
introduce OS-level virtualization and describe the features that are 
of interest for our purposes, as well as their fundamental 
limitations. We introduce an architecture-independent way of 
quantifying variability within and across machines. Then, we present 
our proposed methodology and show preliminary results of this 
approach. Lastly, we close with a brief review of related work and 
final remarks.

# OS-level Virtualization

OS-level virtualization [@soltesz_containerbased_2007] is a server 
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
the dependencies of an application [@dicosmo_package_2008]. In the 
remaining of this paper we focus on Docker [@merkel_docker_2014], 
which employs Linux's cgroups and namespaces features to provide 
OS-level virtualization. While our discussion is centered around 
cgroups, the overall strategies can be applied to any of the other 
technology mentioned above.

`cgroups` [@corbet_notes_2007] is a unified interface to the operating 
system's resource management options, allowing users to specify how 
the kernel should limit, account and isolate usage of CPU, memory, 
disk I/O and network for a collection of processes. In our case, we're 
interested in the CPU and memory limiting capabilities.

## CPU Bandwidth Throttling

`cgroups` exposes parameters for the Completely Fair Scheduler (CFS) 
[@molnar_modular_2007]. The allocation of CPU for a group can be given 
in relative (`shares`) or absolute (`period` and `quota`) values. 
Figure 1 shows the effect that multiple values for `quota` have on the 
execution of a CPU-bound process.

![Boxplots of runtimes of the `crafty` benchmark for multiple values 
of cpu quota (with a fixed period of 100 microseconds). This 
illustrates the effectiveness of the CFS scheduler for limiting CPU 
access for a single-threaded process executing the crafty benchmark. 
Every boxplot summarizes 10 executions. As the figure shows, the 
interquartile box is tight and it overlaps with the 
median.](figures/cgroups.png)

## Limitations of Throttling

While absolute CPU bandwidth limitations work well to isolate 
processes within a single system, they are not guaranteed to be as 
effective while going across multiple platforms and multiple 
applications. The main reason being that fundamental differences 
between two machines (e.g. CPU features, memory bandwidth, etc.) make 
it practically impossible to find a single value for `quota` that 
works for _every_ application. One can tune the quota/period to 
reproduce results for a particular application, but these values won't 
work for another application with a different opcode mix.

# Quantifying Performance Variability Ranges

In order to evaluate the effectiveness of CPU bandwidth throttling in 
controlling variability, we need a way of visualizing the range of 
performance behavior across machines. Ideally, we would like to have 
programs that execute the same opcode over and over again so that we 
could characterize their performance at the level of the OS (e.g. the 
amount of CPU utilization over time). Since we can't have programs 
that execute one single operation, an alternative is to create 
synthetic microbenchmarks that get as close as possible to exercising 
particular features of a system.

[stress-ng](https://github.com/ColinIanKing/stress-ng) is a tool that 
is used to "stress test a computer system in various selectable ways. 
It was designed to exercise various physical subsystems of a computer 
as well as the various operating system kernel interfaces". There are 
multiple stressors for CPU, CPU cache, memory, OS, network and the 
filesystem. Since we focus on CPU bandwidth, we look at the `cpu` 
"stressor", which is a routine that loops a function (termed CPU 
_methods_) multiple times and reports the rate of iterations executed 
for a determined period of time (referred to as 
`bogo-ops-per-second`). There are 111 CPU methods in total that range 
from bitwise operations, to floating, integer operations on multiple 
word sizes, among many others.

Using this battery of CPU stressors, one can obtain a profile of CPU 
performance for a machine. When this profile is normalized against the 
profile of another machine, we obtain a _variability profile_ that 
characterizes the speedups/slowdowns of a machine $B$ w.r.t. another 
one $A$. We refer to this profile as the variability profile for $B/A$ 
(or the $B/A$ profile for brevity). Figure 2 shows a histogram (in 
green) of the variability profile of machines _base_ and $T_3$ from 
Table 1 (i.e. the $T_3$|_base_ profile). The purple histogram is 
discussed in _Section V_, so our interest is on the green one for now. 
We look closely to the _range_ of the histogram of the variability 
profile. As mentioned earlier, our goal is to reduce this range so 
that an application has less wiggle room for performance variations.

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

![Histograms for two variability profiles. The green histogram is the 
$T_3$|_base_ and is described in this section. The purple histogram is 
described in _Section V.B_. Each data point in a histogram corresponds 
to the performance speedup/slowdown of a stress-ng CPU method that a 
machine has w.r.t. another one. For example, in the $T_3$|_base_ 
histogram (green), the speedup caused by the architectural 
improvements of machine $T_3$ causes 7 stressors to have a speedup 
within the `[1.75, 2)` range over machine 
_base_.](figures/with_and_without_limits.png)

[^porta]: An implementation of this methodology for Docker containers 
is available at <https://github.com/systemslab/porta>. **Note to 
reviewers**: the repo will be made public if this article gets 
accepted.

## Tuning The Target Machine

Finding the values of CPU bandwidth (step 2) is done via program 
auto-tunning [@ansel_opentuner_2014] for one or more microbenchmarks 
that characterize the performance of the underlying hardware. At every 
execution step, a docker container is instantiated and constrained 
with a value for CPU quota. It is reasonable to assume that the 
performance of CPU corresponds to a monotonically decreasing function 
(as in Figure 1), thus, we can select a random value within the valid 
tunable range (or alternatively the highest) and climb/descend until 
we get to the desired target performance for the microbenchmark(s). 
When multiple microbenchmarks are executed, their results need to be 
aggregated (e.g. by taking a weighted average of a speedup metric).

The tuning methodology assumes that the machine where an application 
is being ported to (system $B$) is relatively more powerful that the 
one were an application originally ran[^cantspeeduphardware]. When 
this assumption does not hold, one can resort to constraining the 
original execution (i.e. generating a $C_a$ for $A$).

[^cantspeeduphardware]: If machine $B$ is slower than $A$, there's no 
way one can speedup $A$. Overclocking may be an alternative but we 
think it's easier to just slowdown machine $A$ instead, so that a 
common-denominator between the two can be found.

# Results

In this section we show the effectiveness of our proposed methodology 
(_Section V.B_) by obtaining the variability profile for multiple 
machines with respect to a baseline machine; we do so by visualizing 
the reduction of the variability range when we apply the mapping 
methodology (Section IV) to the targets. We then study the effects of 
this reduction by executing a variety of benchmarks on the same 
platforms (_Section V.C_).

\begin{table}[ht]
\caption{Components of base and target machines used in our study. All 
machines run Ubuntu Server 14.04.3 and Linux 3.13. Complete details 
about these systems can be found in the article's github repository.}

\scriptsize
\centering
\begin{tabular}{@{} c c c c @{}}
\toprule

Machine ID & CPU Model              & Memory BW & Release Date \\\midrule
base       & Opteron 2212 @2.0GHz   & 4x2GB DDR2   & Q3-2006 \\
$T_1$         & Core i7-930 @2.8GHz    & 6x2GB DDR3   & Q1-2010 \\
$T_2$         & Core i5-2400 @3.1GHz   & 2x4GB DDR3   & Q1-2011 \\
$T_3$         & Xeon E5-2630 @2.3GHz   & 8x8GB DDR3   & Q1-2012 \\
$T_4$         & Opteron 6320 @2.8GHz   & 8x8GB DDR3   & Q3-2012 \\
$T_5$         & Xeon E5-2660v2 @2.2GHz & 16x16GB DDR4 & Q3-2013 \\

\end{tabular}
\end{table}

<!--
base - piha
t1 - nibbler
t2 - dwill
t3 - proliant
t4 - rackform
t5 - clemson
-->

## Hardware Setup

The list of servers used in our study is shown in Table 1. The reason 
for selecting a relatively old machine as our baseline is two-folded. 
First, by picking an old machine we ensure that all the target 
machines, when unconstrained in CPU bandwidth, can outperform the base 
machine in every test of stress-ng. In other words, the base machine 
serves as a lower-common denominator that all the targets can match or 
surpass. Secondly, having an old computer as part of the our study 
resembles the scenario that many researchers face while trying to 
reproduce results found in the literature. In this case, we recreate a 
hypothetical situation where 5 researchers (one per each machine) try 
to replicate an study done somewhere between 2007-2009 (using a 
machine model from 2006).

## Reduction of Variability Range

Comparing the range of two histograms illustrates the differences in 
performance variability for a pair of machines (or the same machine 
without and with constraints). Perfect performance reproducibility of 
results would result in having the performance of every benchmark to 
be in $x=1.0$ (+/- a reasonable delta, for example 5%). As mentioned 
before (_Section II.B_), fundamental differences between two machines 
such as CPU, memory, micro-controllers, BIOS configuration, etc. make 
it practically impossible to have perfect reproducibility.

Yet, reducing the performance variability (shrinking the range around 
$x=1.0$) is an attainable goal. The green histogram in Figure 2 
corresponds to variability profile $T_3$|_base_. The purple one 
corresponds to the variability profile of $T_3$ after being 
constrained (w.r.t. _base_) using the tuning methodology from _Section 
IV_. We denote this profile as $T_3'$|_base_. In this particular case, 
tuning resulted in a CPU quota of 6372 microseconds for a period of 
10000 microseconds. When these bandwidth limitations are in place, the 
variability range is reduced from $[1.6-13.4]$  to $[0.7,2.2]$, i.e. 
from a range of length 11.8 to one of size 1.5, a ~1/8x reduction. Due 
to space constraints, we omit the results for the other 4 systems but 
we note that the reduction in range varies from 1/2 to 1/10 in some 
cases.

We make two main observations about Figure 2. First, more than 50% of 
the data points cluster around the $[0.8-1.2]$ range, while 30% around 
the $[0.95-1.05]$ range (not shown) for the limited case (purple 
histogram). In the unconstrained case (green), the median is 2.67 with 
a long tail towards the higher speedup values. Secondly, while 6372 
represents more than ~50% of CPU time, the range shrinks only by ~12%. 
As shown in Figure 1, this is mainly due to the non-linear behavior of 
CPU performance under different loads. An open question is whether the 
same performance variability would be observed with dynamic frequency 
scaling.

## Empirical Validation

Assuming stress-ng's distinct CPU methods represent a realistic 
coverage of the multiple physical features of a processor, we can 
reasonably assume that the performance of applications under 
constrained CPU bandwidth will land within this range. In order to 
corroborate this hypothesis, we executed 30 benchmarks with CPU 
bandwidth limitations on the 5 target systems shown in Table 1. Due to 
space constrains we omit the description of each benchmark 
does[^paperepo]. Every benchmark was executed on the base and target 
systems in docker containers. While docker makes binary 
reproducibility possible, the compiler options used at the time that 
an image is built can significantly influence the behavior, depending 
on which machine is used to build it, as well as where a container is 
executed. In order to minimize the variability that might originate 
from distinct compiler optimizations, and as a way of making 
meaningful comparisons, we disabled compiler optimizations (`gcc`'s 
`-O0` flag) for all of the evaluated applications. Also, as mentioned 
previously, these are single-threaded processes running in uncontended 
systems.

![Histogram for $T_3'$|_base_ and $T_3$|_base_ profiles. The data 
points come from the following benchmarks: STREAM, cloverleaf-serial, 
comd-serial, sequoia (amgmk, crystalmk, irsmk), c-ray, crafty, 
unixbench, stress-ng (string, matrix, memory and cpu-cache). Vertical 
lines denote the limits of the predicted variability range (Figure 2), 
obtained from executing stress-ng CPU stressors. The rightmost limit 
for the unconstrained (green) histogram is not shown to improve the 
readability of the figure (it lies on the 14x bin). Points outside the 
predicted line correspond to STREAM, a memory-bound 
workload.](figures/benchmarks.png)

Figure 3 shows the results of our tests for $T_3$ for both constrained 
(purple) and unconstrained (green) scenarios. Each point on a 
histogram corresponds to one benchmark. The two vertical lines denote 
the variability range obtained from Figure 2. For the constrained case 
(purple), with the exception of one point, all executions land within 
the predicted range. The point out of the range corresponds to the 
STREAM benchmark which measures memory throughput. We also observe 
that the highest value of the range obtained in the previous section 
(rightmost vertical line) is within the 1.9x bin, the performance of 
28 out of 30 never go above the smaller $[0.6-1.6]$ range. In the case 
of executions without limits (green histogram), we observe 2 points 
going out of the predicted range, the one at $[1.5-1.6]$ and another 
(not shown) at 14x, both corresponding to memory-bound benchmarks 
(STREAM and stress-ng-memory).

From the analysis of the variability profiles for these 30 benchmarks, 
we can conclude that the set of stress-ng microbenchmarks are good 
representatives of the CPU performance of a machine. Also, for 
CPU-bound workloads, the variability profile seems to be a good 
performance predictor, i.e. an execution lies within the determined 
speedup/slowdown range.

[^paperepo]: For a complete description benchmarks, as well as 
detailed hardware and software configuration please refer to the 
repository of this article at <https://github.com/ivotron/varsys16>. 
**Note to reviewers**: the repo is private but access can be granted 
upon request.

<!--
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
-->

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

In [@collberg_repeatability_2015] the authors took 613 articles 
published in 13 top-tier systems research conferences and found that 
25% of the articles are reproducible (under their reproducibility 
criteria). The authors did not analyze performance. In our case, we 
are interested not only in being able to compile the original program, 
but also in binary reproducibility (the ability to run the same binary 
as in the original setting) and, more importantly, predicting the 
variability of the system being evaluated.

The closest work to our approach is Fracas [@buchert_accurate_2010]. 
Fracas emulates CPU frequency for the same machine. As stated in that 
work, accurately emulating CPU frequencies is a challenging task, even
within the same system. Instead, we take the performance profiles as 
our baseline and quantify the performance variability of reproducing 
in other target systems, irrespective of the differences between 
frequencies.

Architecture-independent characterization of workloads 
[@hoste_microarchitectureindependent_2007] and performance 
[@marin_crossarchitecture_2004] has been extensively researched in the 
past. In our case, working at the OS virtualization level imposes new 
challenges. A way of overcoming these is by using a comprehensive list 
of microbenchmarks that can accurately characterize the performance of 
the underlying system.

# Conclusion

Characterizing the runtime performance variability of a machine helps 
in the interpretation of results obtained when attempting to reproduce 
performance across distinct hardware platforms. In the future we will 
work in relaxing the constrains that we impose in order to support 
multi-threaded applications and collocated workloads.

**Acknowledgements:** Work performed under auspices of U.S. Department 
of Energy by Lawrence Livermore National Laboratory under Contract 
DE-AC52-07NA27344 LLNL-CONF-681457-DRAFT. Sandia National Laboratories 
is managed and operated by Sandia Corporation, for the U.S. Department 
of Energy's National Nuclear Security Administration under contract 
DE-AC04-94AL85000.

# References

<!-- hanged biblio -->

\noindent
\vspace{-2em}
\setlength{\parindent}{-0.26in}
\setlength{\leftskip}{0.2in}
\setlength{\parskip}{8pt}
