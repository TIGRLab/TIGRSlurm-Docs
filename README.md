# Getting Started
### TIGRSLURM: the Kimel SLURM cluster

Before using the queue it's useful to know what's available for you to use on the system. The queue consists of groups of computers called **partitions**:

```
STICK IN DIAGRAM OF QUEUE
```

When you submit any job to our Kimel cluster, it goes to any one of the available partitions which you need to specify (moby is default). In general:

- high-moby - contains high performance computers
- low-moby - contains mid-tier performance computers
- thunk - contains low performance computers
- moby - contains all the cluster's computers
- cudansha - contains computers which have a usable GPU

You can inspect which computers are available in each partition by opening up a terminal emulator and using [`sinfo`](https://slurm.schedmd.com/sinfo.html) which will display something like:

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

Which gives you an idea of which partitions are available and more specifically the number of computers as well as their identities. For (a lot) more details, check out the official [SLURM Quick Start User Guide](https://slurm.schedmd.com/quickstart.html).

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

We can submit this job to the SLURM queue at Kimel by opening up the terminal, going to the directory containing and the script and entering:

```bash
sbatch ./hello_slurm.sh
```

You will see a message saying:

```shell
Submitted batch job <jobid>
```

Where `jobid` is a number.

#### Inspecting your first job

Once you've submitted a job you can check its status via the [`sacct`](https://slurm.schedmd.com/sacct.html) command in the terminal. `sacct` will only show your jobs and what their statuses are, unless you specify otherwise using the `-u` option with another username. For example you might see something like:
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
- `State` describes Job state and can either be `COMPLETED`, `RUNNING`, `PENDING`, or possibly `FAILED`
- `Partition` tells you the [**partition**](#tigrslurm:-the-kimel-slurm-cluster) used. By default **moby** is used but you can specify which partition you want to use manually with `sbatch --partition` or even better, using ['directives'](#directives:).

Once the job `State` is `COMPLETED` (it'll take a minute), we can inspect the outputs - in this case in `/scratch/<your_user>/slurm_hello`:

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

Where the `-u` flag indicates `user`, as it does for `sacct` as well.

###  Writing SLURM scripts ###

To make use of SLURM, you may provide options known as [*sbatch directives*](https://slurm.schedmd.com/sbatch.html). Here, we'll cover a few basic directives and their use in some example scripts. These directives are provided within SLURM scripts on lines immediately following the shebang line. The directives we'll cover here will be the following:
  * `--time` : for specifying time-to-completion
  * `--job-name` : for naming jobs
  * `--ntasks` : for specifying numbers of *job steps*
  * `--cpus-per-task` : for allocating CPU resources
  * `--array` : for specifying the dimensions of large jobs
  * `--partition` : for specifying a partition
  * `--gres` : for special resource requirements

If these look like options that may be passed on the command line, that's because the are. All SLURM directives are equivalent to command line options, and may be passed on the command line when you call a SLURM script with `sbatch`, as in `sbatch --partition=low-moby my-script.sh`. However, this is inconvenient and should be avoided, as it forces you to correctly type each directive every time. Instead, follow this example and place the directives inside the script.

#### Directives: ####

``` bash
#!/bin/bash
#SBATCH --job-name=script_test
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=2
#SBATCH --gres=mem:8K
#SBATCH --time=00:20:00
#SBATCH --partition=high-moby
#SBATCH --output=/projects/<your_name>/slurm_%A.out
#SBATCH --error=/projects/<your_name>/slurm_%A.err

echo "Hello, this script ran on `hostname -s` on `date --iso`."
```

In this script, we've used a series of directives to effectively delineate how this job is to be performed. Let's lay out what they do and why:

| directive         | meaning                         | important                                |
|:-----------------:|:-------------------------------:|:----------------------------------------:|
| `--job-name`      | Give your job a name.           | Make it distinctive.                     |
| `--ntasks`        | How many times should it run?   | Helpful for repetitive jobs.             |
| `--cpus-per-task` | Allocate it CPU cores.          | How many will it *actually* use?         |
| `--gres`          | Request special resources.      | Does your job have special requirements? |
| `--time`          | Allocate it a time limit.       | This is a *maximum* time.                |
| `--partition`     | Allocate it to a partition.     | It will get higher priority here.        |
| `--output/error`  | The location of your log files. | *Always* in your scratch or projects.    |

For directives which specify resources such as `time` and `cpus-per-task`, it is important that these allocations be approximately accurate, as they effectively limit your job. Directives such as these are *caps*, and cannot be exceeded.

Therefore, if you allocate *too few* resources to your job, it may fail or work may be slow. However, if on the other hand you allocate *too many* resources, it may fail *even to start*, as the resources you requested might be unavailable.

A good rule when writing these directives is this:
**For any set of resources specified by directives, you are limiting yourself to machines possessing all of them.**

Ask for too many CPUs, too much RAM, or a GPU, and you're shrinking the number of machines eligible to run your job.

[Resource Allocation](LINKHERE) and [Logging](LINKHERE) in SLURM will be covered later in their own respective sections in more detail.

#### Array Jobs ####

An [**array job**](https://slurm.schedmd.com/job_array.html) is a special kind of SLURM job which allows you to specify that an identical job script should be exected with some non-identical parameter. In an array job, the `--array` directive is used to specify a numerical range, as in `--array=1-250`. This array works similarly to `--ntasks=250`, except that it provides a distinguishing variable *within* your script called a `SLURM_ARRAY_TASK_ID`, allowing each copy of the script to run on a different datum. For example.

``` bash
#!/bin/bash
#SBATCH --job-name=slurm_array_test
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --time=01:00:00
#SBATCH --partition=low-moby
#SBATCH --array=1-250%10
#SBATCH --output=/projects/<your_user>/logs/%x_%A_%a.out
#SBATCH --error=/projects/<your_user>/logs/%x_%A_%a.err

slurmarray=(`seq 1 250`)

echo "The number ${slurmarray[$SLURM_ARRAY_TASK_ID]} appeared on `hostname -s`"
```

This script generates an array called `slurmarray` filled with the numbers 1 through 250, and echoes a different number on each machine the script executes on. Every time Slurm runs this script, on whichever compute node, it provides a unique value for the variable `SLURM_ARRAY_TASK_ID`. By placing this variable into a
