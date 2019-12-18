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

The primary difference between SciNet’s compute cluster and our local cluster, for the individual user’s perspective, is that unlike in our local cluster, or even the SCC, SciNet **does not allow users to request single processors.** No less than one entire 80 processor node will be allocated to any job, and it is up to users to adequately apportion the full resources of these machines. This is *very important*, Compute Canada will complain at you, via e-mail, quite vociferously, if you request a machine that you go on to substantially under-utilise. It is thus crucial to your usage of SciNet as a resource that you structure your jobs sensibly to maximise your use of these nodes.

On our local cluster, as mentioned in the documentation there, often the most efficient way to utilise resources is with an [array job](https://github.com/TIGRLab/TIGRSlurm-Docs/blob/master/README.md#arrayjobs). Briefly, an array job looks like this:

``` sh
#!/bin/bash
#SBATCH --array=1-200

echo "$SLURM_ARRAY_TASK_ID (a number) was found on $SLURMD_NODENAME (a hostname)"
```

In this example, when our array script is run using the `sbatch` command, 200 copies of this script will be spawned across the local cluster, each one yielding a different value for `$SLURM_ARRAY_TASK_ID` and for `$SLURMD_NODENAME`, with `$SLURM_ARRAY_TASK_ID` being a number, 1 through 200, and `$SLURMD_NODENAME` being the name of the machine this script ran on.
