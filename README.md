# Getting Started
### The Kimel SLURM system

Before using the queue it's useful to know what's available for you to use on the system. The queue consists of groups of computers called **partitions**:

```
STICK IN DIAGRAM OF QUEUE 
```

When you submit any job to our Kimel cluster, it goes to any one of the available partitions which you need to specify (moby is default). In general:

- High moby - contains high performance computers 
- Low moby - contains mid-tier performance computers
- Thunk - contains low performance computers
- Moby - uses both High and Low moby computers
- Cudansha - contains computers which have a usable GPU

You can inspect which computers are available in each partition by opening up terminal and using `sinfo` which will display something like:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
thunk        up    1:00:00      4   idle golgi,hebb,laplace,mrsdti
cudansha     up   infinite      3    mix bulbasaur,darwin,hopper
cudansha     up   infinite      5   idle higgs,mendel,purkinje,zerbina,zippy
low-moby     up 1-08:00:00     10    mix bulbasaur,cajal,darwin,davinci,franklin,hopper,kandel,mansfield,nissl,talairach
low-moby     up 1-08:00:00     11   idle crick,higgs,lovelace,mendel,milner,penfield,purkinje,strangelove,tesla,zerbina,zippy
high-moby    up   infinite      6    mix bayes,borg,deckard,hawking,noether,ogawa
high-moby    up   infinite      1   idle downie
```

Which gives you an idea of which partitions are available and more specifically the number of computers as well as their identities. For more details check out the SLURM documentation (LINK HERE). 


### How to use SLURM

#### Submitting your first job
In order to run a SLURM job, the first thing you will need is a job script. This can be *in any kind of language* such as Bash, Python, MATLAB, or R. In this section we'll stick with Bash for simplicity. Suppose that we write up a text file called `hello_slurm.sh` with the following contents:

```bash
#!/bin/bash

# Do nothing for a minute 
sleep 60

#Output a message
echo "Hello world!" > /scratch/<your_name>/slurm_hello

#Print out which computer is running your job
hostname >> /scratch/<your_name>/slurm_hello

```

This script will write to a file with the path: `/scratch/<your_name>/slurm_hello`. 

We can submit this job to the SLURM queue at Kimel by opening up terminal, going to the directory containing and the script and entering:

```bash
sbatch ./hello_slurm.sh
```

You will see a message saying:

```shell
Submitted batch job <jobid>
```

Where `jobid` is a number. 

#### Inspecting your first job

Once you've submitted a job you can check its status via the `sacct` command in terminal. `sacct` will only show your jobs and what their statuses are. For example you might see something like:
```
       JobID    JobName  Partition    Account  AllocCPUS      State ExitCode
------------ ---------- ---------- ---------- ---------- ---------- --------
152760       hello_slu+       moby    tigrlab          1  COMPLETED      0:0
152760.batch      batch               tigrlab          1  COMPLETED      0:0
152760.exte+     extern               tigrlab          1  COMPLETED      0:0
```

- `JobID` is an ID which uniquely identifies the job
- `JobName` is the name given to the job. By default it is the name of the script but see `point to directives` for details on how to customize this
- `Account` will always be tigrlab on our local Kimel cluster
- `AllocCPUS` is how many CPUs were given to the ask
- `State` describes Job state and can either be `COMPLETED`, `RUNNING`, or `PENDING` 
- `Partition` tells you the **partition**(LINK) used. By default **moby** is used but you can specify which partition you want to use manually with `sbatch --partition` or even better, using Directives (LINK)

Once the job `State` is completed (it'll take a minute), we can inspect the outputs - in this case in `/scratch/<your_user>/slurm_hello`:

```
cat scratch/<your_user>/slurm_hello

> Hello world!
> bayes.camhres.ca
```

As you can see, a computer from the Kimel lab has run your script for you (it could be your own too!). 

A consequence of this is that if you need your job to use a file, **it must be accessible by all computers in the lab!**. If isn't the other computer will fail to find the file and the job will crash. See **WHY IS MY JOB FAILING** for more pitfalls!

#### Cancelling your Job

To cancel any slurm job the command `scancel` is used. Common ways to cancel jobs is to provide the `JobID`:

```
scancel <jobID1> <jobID2> <jobID3> ...
```

Or if you want to cancel *all your jobs*:

```
scancel -u <your_user>
```

Where the `-u` flag indicates `user`.
