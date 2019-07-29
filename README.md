# Getting Started
### TIGRSlurm: the Kimel Slurm cluster

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

Which gives you an idea of which partitions are available and more specifically the number of computers as well as their identities. For (a lot) more details, check out the Slurm official [Quick Start User Guide](https://slurm.schedmd.com/quickstart.html).

### How to use Slurm

#### Submitting your first job
In order to run a Slurm job, the first thing you will need is a job script. This can be *in any kind of language* such as Bash, Python, MATLAB, or R. In this section we'll stick with Bash for simplicity. Suppose that we write up a text file called `hello_slurm.sh` with the following contents:

```bash
#!/bin/bash

# Do nothing for a minute
sleep 60

# Output a message
echo "Hello world!" > /scratch/<your_name>/slurm_hello

# Print out which computer is running your job
hostname >> /scratch/<your_name>/slurm_hello

```

This script will write to a file with the path: `/scratch/<your_name>/slurm_hello`. 

We can submit this job to the Slurm queue at Kimel by opening up the terminal, going to the directory containing the script and entering:

```bash
sbatch hello_slurm.sh
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
- `Partition` tells you the partition used. By default **moby** is used but you can specify which partition you want to use manually with `sbatch --partition` or even better, using ['directives'](#directives:).

Once the job `State` is `COMPLETED` (it'll take a minute due to `sleep`), we can inspect the outputs - in this case in `/scratch/<your_name>/slurm_hello`:

```
cat scratch/<your_name>/slurm_hello

> Hello world!
> bayes.camhres.ca
```

As you can see, a computer from the Kimel lab has run your script for you (it could even have been your own). However, Slurm provides no guarantee that it will run on any *specific* machine unless you request one by name. Instead, Slurm simply provides you the ability to ensure your script runs *somewhere* on our cluster.

A consequence of this is that if you need your job to use a file, or touch a file in a directory, **it must be accessible by all computers in the lab!**. If it isn't, the job's machine will fail to find the file and the job will crash. See **WHY IS MY JOB FAILING** for more pitfalls!

#### Cancelling your Job

To cancel any slurm job the command `scancel` is used. Common ways to cancel jobs is to provide the `JobID`:

```
scancel <jobID1> <jobID2> <jobID3> ...
```

Or if you want to cancel *all your jobs*:

```
scancel -u <your_name>
```

Where the `-u` flag indicates `user`, as it does for `sacct` as well.

### Writing Slurm scripts ###

To make good use of Slurm, you may provide options known as [*sbatch directives*](https://slurm.schedmd.com/sbatch.html). Here, we'll cover a few basic directives and their use in some example scripts. These directives are provided within Slurm scripts on lines immediately following the shebang line. The directives we'll cover here will be the following:
  * `--job-name` : for naming jobs
  * `--time` : for specifying time-to-completion
  * `--ntasks` : for specifying numbers of *job steps*
  * `--cpus-per-task` : for allocating CPU resources
  * `--mem-per-cpu` : for allocating memory
  * `--partition` : for specifying a partition
  * `--gres` : for special resource requirements
  * `--array` : for specifying the dimensions of large jobs

If these look like options that may be passed on the command line, that's because they are. All Slurm directives are equivalent to command line options, and may be passed on the command line when you call a Slurm script with `sbatch`, as in:

```bash
sbatch --partition=low-moby my-script.sh
```

However, this is inconvenient and should be avoided, as it forces you to correctly retype each directive every time. Instead, follow this example and place the directives inside the script. This allows you to check the directives for correctness, keeps them correct once they are correct, and allows you to look back over directives you used in past successful runs of some script.

#### Directives: ####

``` bash
#!/bin/bash
#SBATCH --job-name=script_test
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=2
#SBATCH --mem-per-cpu=1024
#SBATCH --time=00:20:00
#SBATCH --partition=high-moby
#SBATCH --output=/projects/<your_name>/slurm_%A.out
#SBATCH --error=/projects/<your_name>/slurm_%A.err

echo "Hello, this script ran on `hostname -s` on `date --iso`."
```

In this script, we've used a series of directives to effectively delineate how this job is to be performed. Let's lay out what they do and why:

| directive         | meaning                         | important                             |
|:-----------------:|:-------------------------------:|:-------------------------------------:|
| `--job-name`      | Give your job a name.           | Make it distinctive.                  |
| `--ntasks`        | How many times should it run?   | Helpful for repetitive jobs.          |
| `--cpus-per-task` | Allocate it CPU cores.          | How many will it *actually* use?      |
| `--mem-per-cpu`   | Allocate it memory.             | How much does it need per CPU?        |
| `--time`          | Allocate it a time limit.       | This is a *maximum* time.             |
| `--partition`     | Allocate it to a partition.     | It will get higher priority here.     |
| `--output/error`  | The location of your log files. | *Always* in your scratch or projects. |

For directives which specify resources such as `time` and `cpus-per-task`, it is important that these directives be approximately accurate, as they effectively limit your job. Allocations such as these are *constraints*, and cannot be exceeded once the job has started.

Therefore, if you allocate *too few* resources to your job, it may fail to work or may be slow. However, the same is true of allocating *too many* resources; if your job is too large, the Slurm scheduler may believe that no machine in the cluster is ready to satisfy its requirements, or else it may provide you with a highly restricted range of machines across which your job may run!

For example, let's say you ask for the following resources:

``` bash
#SBATCH --cpus-per-task=8
#SBATCH --gres=mem:32K
#SBATCH --time=00:20:00
```

In this case, one might *expect* to get access to 16 of the lab's 32 machines, since since we asked for 8 cores per job, and lab machines generally donate approximately half of their resources, and at least 16 of the lab's machines have 16 or more CPU cores.

In reality, though, you would only get 8 machines, because via the `gres` directive, you're also asking for machines willing to donate **at least** 32,000 megabytes (i.e. 32GB) of RAM, and not all 16 core machines have that memory much to give.

The same would be implicity true if we used the directive `mem-per-cpu`. With:

``` bash
#SBATCH --cpus-per-task=8
#SBATCH --mem-per-cpu=4G
```

We're asking for the same total amount (32GB of RAM), and this will restrict us to the same 8 machines.

Furthermore, if you request the following:

``` bash
#SBATCH --partition=low-moby
#SBATCH --time=2-12:00:00
```

Which is a requested time limit of 2 1/2 days, you will in fact *get zero machines and your job will never start*, since, as you can see by running the `sinfo` command, *machines in low-moby are unwilling to run jobs longer than 32h*. In such situations, you'll receive a notification like the following:

``` bash
sbatch: error: Unable to allocate resources: Requested node configuration is not available
```

Because the cluster as a whole does not possess any subset of nodes satisfying all of your provided directives.

The rule when writing these directives is thus the following:

**For any set of resources specified by directives, you are limiting yourself to machines willing to provide AT LEAST ALL OF THEM!**

*Ask for the smallest amount of resources that works!*

Using [Resource Allocation](LINKHERE), [Logging](LINKHERE), and [GRES](LINKHERE) in Slurm will be covered later in their own respective sections in more detail.

#### Array Jobs ####

An [**array job**](https://slurm.schedmd.com/job_array.html) is a special kind of Slurm job which allows you to specify that an identical job script should be executed with some non-identical parameter. In an array job, the `--array` directive is used to specify a numerical range, as in `--array=0-249`. This array works similarly to `--ntasks=250`, except that it provides a distinguishing variable *within* your script called a `SLURM_ARRAY_TASK_ID`, allowing each copy of the script to run on a different datum. For example.

``` bash
#!/bin/bash
#SBATCH --job-name=slurm_array_test
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --time=01:00:00
#SBATCH --partition=low-moby
#SBATCH --array=0-249%10
#SBATCH --output=/projects/<your_name>/logs/%x_%A_%a.out
#SBATCH --error=/projects/<your_name>/logs/%x_%A_%a.err

slurmarray=(`seq 1 250`)

echo "The number ${slurmarray[$SLURM_ARRAY_TASK_ID]} appeared on `hostname -s`"
```

This script generates an array called `slurmarray` filled with the numbers 1 through 250, and echoes a different number on each machine the script executes on. Every time Slurm runs this script it provides a unique value for the variable `SLURM_ARRAY_TASK_ID`. This number allows you to uniquely substitute any datum you might want from the array.

This works for anything. Consider the following:

``` bash
subjects=(`cat $mydirectory/subjects.txt`)

# ... <do some stuff to "${subjects[$SLURM_ARRAY_TASK_ID]}" here> ...

echo "Subject '${subjects[$SLURM_ARRAY_TASK_ID]}' was processed on `hostname -s`"
```

In this example, as above, `"${subjects[$SLURM_ARRAY_TASK_ID]}"` becomes the name of the subject as it appears in `$mydirectory/subjects.txt`. Note also, that the array as specified in `--array` begins with 0, rather than 1. This is because in bash arrays begin at zero; thus we begin our Slurm array at zero as well.

In these examples, our `array` directive has included a per centage sign followed by a number, as in `--array=0-249%10`. This is the `Task Array Throttle`. If we are batch processing an array of 250 subjects, the default behaviour of Slurm is to attempt to run as many subjects as possible.

The Task Array Throttle allows you to cap the number of Slurm's simultaneous job attempts. Thus, `--array=0-249%10` executes a Slurm script once for each of 250 subjects, but never attempts to process more than 10 subjects at a time.

#### Logging: ####

#### Resource Allocation: ####

#### GRES: ####
