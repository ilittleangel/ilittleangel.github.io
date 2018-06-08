---
layout: post
title:  "Joining lists with python3.6"
date:   2018-06-08 22:05:51 +0200
categories: python iterables
---

Few days ago I was wondering what is the best way to perform a "join" with two different list, using Python3.6. The point is to try dont make a cartesian product and filter after that. The third solution seems the best way in performing but it is not the clearest way.

Here it is the three approaches for joining lists with Python:

#### 1. Nested loops
```python
def join(jobids, stages):
    for jobid in jobids:
        for stage in stages:
            stageids = jobid[1]
            if stage['stageId'] in stageids:
                stage['jobid'] = jobid[0]
```
> complexity -> **O(n^2)**
  

#### 2. Nested loops and `itertools` package
```python
def join(jobids, stages):
    for jobid, stage in itertools.product(jobids, stages):
        if stage['stageId'] in jobid[1]:
            stage['jobId'] = jobid[0]
```
> complexity -> **O(n^2)**
  

#### 3. Reverse index and `toolz` package
```python
def join(jobids, stages):
    idx = {stage: job for job, stages in jobids for stage in stages}
    return [toolz.dicttoolz.assoc(stage, 'jobId', idx[stage['stageId']]) for stage in stages]
```
> complexity -> **O(n)**
  

#### Data

```python
jobids = [(76, [75, 77, 88]), (75, [74]), (74, [73])]

stages = [{'status': 'COMPLETE', 'stageId': 75, ...},
          {'status': 'COMPLETE', 'stageId': 74, ...}]
```

