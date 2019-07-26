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

###  Writing SLURM scripts ###

To make use of SLURM, you may provide options known as *[sbatch directives](https://slurm.schedmd.com/sbatch.html)*. Here, we'll cover a few basic directives and their use in some example scripts. These directives are provided within SLURM scripts on lines immediately following the shebang line. The directives we'll cover here will be the following:
  * `--time` : for specifying time-to-completion
  * `--job-name` : for naming jobs
  * `--ntasks` : for specifying numbers of *job steps*
  * `--cpus-per-task` : for allocating CPU resources
  * `--array` : for specifying the dimensions of large jobs
  * `--partition` : for specifying a partition

If these look like options that may be passed on the command line, that's because the are. All SLURM directives are equivalent to command line options, and may be passed on the command line when you call a SLURM script with Sbatch. However, this is inconvenient and should be avoided, as it forces you to correctly type each directive every time. Instead, follow this example.

#### Directives, an example: ####

```bash
#!/bin/bash
#SBATCH --job-name=script_test
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=2
#SBATCH --time=00:20:00
#SBATCH --partition=high-moby
#SBATCH --output=/projects/<your_name>/slurm_%A.out
#SBATCH --error=/projects/<your_name>/slurm_%A.err

echo "Hello, this script ran on `hostname -s` on `date --iso`."
```

In this script, we've used a series of directives to effectively delineate how this job is to be performed. Let's touch on each one briefly:

The `job-name` directive ensures that this job's *name* will be distinctive and noticable, for example, in a large list of jobs.

The `ntasks` directive specifies that in order for this job to be considered 'completed', the script must have run this many times on any computer at any time.

The `cpus-per-task` directive specifies that each time this script runs on some machine, it is expected to use this many CPU cores. It is reasonably important to provide an accurate CPU count, as providing too few CPUs will slow it down or even kill it, while providing too many cores may prevent the job from running soon or at all

The `time` directive indicates an estimated deadline by which the job is expected to finish. Like cpus-per-task, it is important for this to be correct, as a job which exceeds this time limit will be liable to be killed by SLURM, while a job which asks for too much time will be liable to wait a long while for other jobs to finish first.

The `partition` directive allows you to specify which subset of the cluster's machines to assign your job to. Partitions are mildly important to assign, as different partitions provide maximum time limits and other constraints. On the other hand, if no partition is specified, your job may run anywhere but will have low priority.

The `output` and `error` directives tell Slurm where to direct the log outputs of your script. **Always** direct the log outputs into globally available directories, either in your project or scratch directory.

[Resource Allocation](LINKHERE) and [Logging](LINKHERE) in Slurm will be covered later in their own respective sections in more detail. In the above example, we have certainly *over-served* this job; as 20 minutes and two cores are far greater than the resources required to execute simple echo, date, and hostname commands.

Note that this does not mean that this script will *actually run* for 20 minutes or *actually use* 2 CPU cores. Instead, these directives are a *prediction* of expected requirements. However, since the predictive nature of these directives is used by SLURM to schedule your job alongside possibly many others in an effort to provide equitable service to all users, SLURM will *de-prioritise* large, expensive jobs in favour of smaller, inexpensive jobs.

What this means is that, in practice, your goal when writing a SLURM script should be to assign the smallest number of CPUs and the smallest amount of time to your job that you can manage, without under-estimating your job's *actual* requirements. The smaller your job's footprint, the sooner it will run, and the faster it will be able to complete. In the following section, we'll provide another example which illustrates the use of the `array` directive, which is peculiarly useful for large jobs, but also generally poorly understood by new users.

#### Array Jobs ####

An **[array job](https://slurm.schedmd.com/job_array.html)** is a special kind of SLURM job which allows you to specify that an identical job script should be exected with some non-identical input or options. In an array job, the `--array` directive is used
