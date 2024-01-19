---
layout: post
title: GSoC 2023 - Implementation of parallel analysis in MDAnalysis
---

As you might know from my [previous posts](https://marinegor.github.io/year-archive/), during the summer of 2023 I've been working on MDAnalysis during Google Summer of Code. Here I'll summarize what I've done, how others can use it, and what changes will follow that in the MDAnalysis codebase in the near future.


## Goals of the project
One sentence: introduce parallel execution of analysis runs in [MDAnalysis](https://github.com/MDAnalysis/mdanalysis) library. Somewhat good introduction I also gave [here](https://marinegor.github.io/posts/2023/05/gsoc-proposal/) when writing a proposal for the project.

In more technical details, MDAnalysis library (as of v2.6.0) contains around 30 different subclasses that can perform various molecular trajectory analysis tasks, like calculating RMSD, RMSF, various contacts, density analysis, and more advanced tasks. 24 of these subclasses are children of `AnalysisBase` class. This base class is written in a way that allows subclass authors care about implementing only 3 methods:

 1. `_prepare()` -- how to initialize attributes for the analysis
 2. `_single_frame()` -- how to do analysis of a single molecular trajectory frame
 3. `_conclude()` -- how to transform intermediate results into final ones

With only these 3 methods implemented, by inheritance subclasses get `run()` method that will take care of reading the trajectory, storing the results, logging, etc.

The main goal was to re-write `AnalysisBase.run()` so that it can run in parallel on multiple processes, but make these changes invisible from the subclasses, i.e. not re-write any of their code.

## What I did
Altogether, the `AnalysisBase.run()` has changed by addition of the following methods:

 - `_setup_computation_groups()`: split frames into multiple parts for separate analysis
 - `_compute()`: run `_single_frame` on a list of frames, but without running `_conclude`
 - `_get_aggregator()`: get an object to aggregate the run results with, making them compatible with subsequent `_conclude`
 - class property `supported_backends()`: get tuple of `str` values that describe available backends for a given subclass

I've also added `BackendBase` and `ResultsGroup` classes that abstract away parallel execution and results aggregation, respectively. And finally, I added `multiprocessing` and `dask` backends that reportedly speed up the analysis!

## The current state
Currently, changes to the `AnalysisBase` are almost finalized. One thing that holds it back is some CI/CD issues causing tests to timeout, but all `AnalysisBase`-related tests run both locally and on CI/CD system.

## What's left to do
An optional thing suggested within my proposal was to actually add parallelization to the subclasses, and update tests accordingly. It turned out to be a tedious task, but finally the mechanism is finalized and described in `MDAnalysisTests/analysis/conftest.py`, generating appropriate fixtures for testing subclasses with all supported (for the subclass) and installed (in the environment) backends.

Since it is a big change for the library, essentially changing the API for the "analysis" part of the MDAnalysis, it requires a lot of time to merge into the develop branch, and wasn't merged within the GSoC timeframe. But the feature is expected by [version 3.0](https://www.mdanalysis.org/2023/10/25/towards_3.0/) in the core library 🚀

## What code got merged (or not) upstream
The main changes are summarized in the [main pull-request](https://github.com/MDAnalysis/mdanalysis/pull/4162) of the project. They mostly involve changes to `package/analysis/base.py`, as well as installation and CI/CD configuration files. Also, there are example changes in `package/analysis/rms.py` introducing parallelization into RMSD and RMSF subclasses, showcasing the changes to be made in order to add parallelization to a certain class.


## How can you use it?

Let's imagine we've just learned that MDAnalysis now supports parallelization, and want to calculate RMSD of our large trajectory faster using multiple cores on our machine. Imagine we have a 16-core CPU Linux workstation with an SSD drive and a 1 us trajectory of lysozyme in water. Like this:


```python
import MDAnalysis as mda
from MDAnalysis.analysis import rms

prefix = "./large_data/"
traj, top = f"{prefix}/md_0_1.xtc", f"{prefix}/md_0_1.gro"

u = mda.Universe(top, traj)
```

First we want to get a reference structure by taking the average of all frames. Like this:

```python
from MDAnalysis.analysis.align import AverageStructure

avg = AverageStructure(mobile=u).run(backend='multiprocessing', n_workers=16)
```

but we get this:

```python
ValueError                                Traceback (most recent call last)
Cell In[11], line 2
      1 from MDAnalysis.analysis.align import AverageStructure
      2 avg = AverageStructure(mobile=u).run(backend='multiprocessing', n_workers=16)
...
ValueError: backend=multiprocessing is not in self.available_backends=('serial',) for class AverageStructure
```

which basically says we can use only `backend='serial'` (which means "current process processing data in serial fashion") for the `AverageStructure`. Ok, let's do that, but with a large step to save time:


```python
avg = AverageStructure(mobile=u).run(step=100)
ref = avg.results.universe
```

and start our analysis run -- RMSD for multiple selections to later compare them between each other:


```python
groupselections = ("protein", "backbone", "name CA")

R = rms.RMSD(
    u,  # universe to align
    ref,  # reference universe or atomgroup
    groupselections=groupselections,
    select="backbone",  # group to superimpose and calculate RMSD
)
```

If we start it with `R.run()`, we won't even know when the run would finish. Luckily, we can add some verbosity with `R.run(verbose=True)` and see a nice `tqdm` progressbar that shows us that ETA of the whole analysis is around 4 minutes:

```python
>>> R.run(verbose=True)
5%|▌         | 5062/100001 [00:13<04:15, 371.20it/s]
```

Let's try to speed it up now. Which backends do we have available?

```python
>>> rms.RMSD.available_backends
('serial', 'multiprocessing', 'dask')
```

Let's try a built-in `multiprocessing` first:

```python
>>> R.run(backend='multiprocessing', n_workers=4)
# CPU times: user 153 ms, sys: 74.2 ms, total: 227 ms
# Wall time: 1min 14s
```

OK, this is roughly 4 times faster! Amazing, roughly as we expected.
Spoiler though -- if we do it with 16 workers, we'll see the total time around 40 seconds, so improvement saturates at some point.


## Conclusion
We've now gone through the essence of the MDAnalysis parallelization project, and learned how to use it in your analysis either by simply adding `backend='multiprocessing', n_workers=...`, or using `dask` if you want to have more flexible serialization.

Hopefully, this project will grow further and include all existing subclasses, as well as improving the speed (which, as we saw, saturates) and memory efficiency.

If you want to contribute to the project, stay tuned for the new issues on MDAnalysis [github](https://github.com/MDAnalysis/mdanalysis) -- there definitely will be some parallelization-related things for the future described there!


-- @marinegor
