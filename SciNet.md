# Running Slurm Jobs on SciNet Niagara #
## Getting Set Up ##

Now that you know a bit about how the Kimel Lab Slurm cluster works (if not see [here](https://github.com/TIGRLab/TIGRSlurm-Docs/blob/master/README.md)), we can discuss some of the differences between the local lab cluster and the SciNet Niagara cluster.

All of our publically available datasets are processed, archived, and stored on SciNet. You should do all of your computing on SciNet for all of the following studies:

- HBN (Healthy Brain Network dataset)
- HCP (Human Connectome Project dataset)
- PNC (Philedelphia Neurodevelopmental Cohort dataset)
- POND (Province of Ontario Neurodevelopmental Disorders dataset)

To start with, we'll assume you have a Compute Canada account associated with the lab's RAC. If not, or if you don't know, visit [FAKELINKHERE](SciNet.onboarding.docs "This will be a link to SciNet onboarding documentation."). To access SciNet, can `ssh` into it using your Compute Canada account username and password:

```
$ ssh <your_CC_username>@niagara.scinet.utoronto.ca

> <your_CC_username>@niagara.scinet.utoronto.ca's password:
```

Once you type your password, you will be on a login node. There are seven such and you'll get routed to one of them, and it doesn't really matter which one.

The primary difference between SciNet’s compute cluster and our local cluster, from the individual user’s perspective, is that unlike in our local cluster, or even the SCC, SciNet **does not allow users to request single processors.** No less than *one entire 80 processor node* will be allocated to any job, and it is up to users to adequately apportion the full resources of these machines. This is *very important* as Compute Canada will complain at you, via e-mail, quite vociferously, if you request a machine that you go on to substantially under-utilise. It is thus crucial to your usage of SciNet as a resource that you structure your jobs sensibly to maximise your use of these nodes.

On our local cluster, as mentioned in the relevant documentation, often the most efficient way to utilise resources is with an [array job](https://github.com/TIGRLab/TIGRSlurm-Docs/blob/master/README.md#arrayjobs). Briefly, an array job looks like this:

``` sh
#!/bin/bash
#SBATCH --array=1-10

echo "$SLURM_ARRAY_TASK_ID (a number) was found on $SLURMD_NODENAME (a hostname)"
```

In this example, when our array script is run using the `sbatch` command, 10 copies of it will be split across the local cluster, each one yielding a different value for `$SLURM_ARRAY_TASK_ID` and possibly for `$SLURMD_NODENAME`, with `$SLURM_ARRAY_TASK_ID` being a number, 1 through 10, and `$SLURMD_NODENAME` being the name of the machine this script ran on. In brief, an array is an ordered sequence of numbers (indices) with associated values:

| index | 0     | 1     | 2     | 3     | 4      | 5     | 6     | 7     | 8     | 9     |
|:-----:|:-----:|:-----:|:-----:|:-----:|:------:|:-----:|:-----:|:-----:|:-----:|:-----:|
| value | val.a | val.b | val.c | val.d | vald.e | val.f | val.g | val.h | val.i | val.j |

	by this token, a slurm job array, like that below:

``` sh
#!/bin/bash
#SBATCH --array=0-4

echo "$SLURM_ARRAY_TASK_ID (a number) was found on $SLURMD_NODENAME (a hostname)"
```

	has the following structure
		(where `I` is the value of SLURM_ARRAY_TASK_ID, equal to the index, and)
	    (where `N` is the value of SLURMD_NODENAME as selected from the cluster):

| index | 0           | 1           | 2           | 3           | 4           |
|:-----:|:-----------:|:-----------:|:-----------:|:-----------:|:-----------:|
| alloc | alloc[N]    | alloc[N]    | alloc[N]    | alloc[N]    | alloc[N]    |
|:-----:|:-----------:|:-----------:|:-----------:|:-----------:|:-----------:|
| value | script[I,N] | script[I,N] | script[I,N] | script[I,N] | script[I,N] |
|       |             |             |             |             |             |
               ↑             ↑             ↑             ↑             ↑
  _____________|_____________|_____________|_____________|_____________|
 |
 `echo "$SLURM_ARRAY_TASK_ID (a number) was found on $SLURMD_NODENAME (a hostname)"`

Slurm allocates N many sets of [requested resources](https://github.com/TIGRLab/TIGRSlurm-Docs/blob/master/README.md#resourceallocation) from the cluster/partition/nodelist, assigns indices to them, and runs a copy of the attached script, with appropriate values for each of the slurm relevant variables (`SLURM_ARRAY_TASK_ID` or `SLURMD_NODENAME`, for example).

The temptation of users new to high performance computing is to process multiple objects (subject directories, text files, scans, etc.) in the simplest way they understand how, which is to use a loop:

``` shell
#!/bin/bash
#SBATCH --array=0-4
#SBATCH --ntasks-per-node=1

SUBJECTS="/path/to/my/scans.csv"

for i in $(cat $SUBJECTS)
do process ${i} --output /scratch/myname/${i}/out
done
```

The problem with this may not necessarily be obvious, until we consider the structure of this visually:

| index | 0        | 1        | 2        | 3        | 4        |
|:-----:|:--------:|:--------:|:--------:|:--------:|:--------:|
| alloc | alloc[N] | alloc[N] | alloc[N] | alloc[N] | alloc[N] |
|:-----:|:--------:|:--------:|:--------:|:--------:|:--------:|
| value | loop     | loop     | loop     | loop     | loop     |
|       |          |          |          |          |          |
             ↑           ↑          ↑         ↑          ↑
  ___________|___________|__________|_________|__________|
 |
 ``` sh
 SUBJECTS="/path/to/my/scans.csv"

 for i in $(cat $SUBJECTS)
 do process ${i} --output /scratch/myname/${i}/out
 done
 ```

The loop body, which processes all subjects by itself; is run independently on each allocated node, thus duplicating the processing of all subjects. If the output of the `process` program is located in a cluster-wide accessible filesystem, such as `/scratch/myname`, the result is that the work duplicated on each node will go to the same files in the same directories, either overwriting or blocking.

They key function of HPC parallelisation is to divide a large job (a processing job that needs to hit hundreds of subjects), and divide that job into the smallest independent units - these may then be spread across the HPC cluster. As such the right way to write a Slurm script for SciNet is to write the script *as though you only want to process one subject*.
