# Getting Started
### TIGRSlurm: the Kimel Slurm cluster

Before using the queue it's useful to know what's available for you to use on the system. The queue consists of groups of computers called **partitions**:

![STICK IN DIAGRAM OF QUEUE](https://slurm.schedmd.com/entities.gif "Obvious placeholder is obvious. Doot.")

When you submit any job to our Kimel cluster, it goes to any one of the available partitions which you need to specify (moby is default). In general:

- high-moby - contains high performance computers
- low-moby - contains mid-tier performance computers
- thunk - contains low performance computers
- moby - contains all the cluster's computers
- cudansha - contains computers which have a [usable GPU](#gres)

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

### <a name="howtouseslurm">How to use Slurm</a>

#### <a name="submittingyourfirstjob">Submitting your first job</a>
In order to run a Slurm job, the first thing you will need is a job script. This can be *in any kind of language* such as Bash, Python, MATLAB, or R. In this section we'll stick with Bash for simplicity. Suppose that we write up a text file called `hello_slurm.sh` with the following contents:

```bash
#!/bin/bash

# Do nothing for a minute
sleep 60

# Output a message
echo "Hello world!" > /scratch/<your_username>/slurm_hello

# Print out which computer is running your job
hostname >> /scratch/<your_username>/slurm_hello

```

This script will write to a file with the path: `/scratch/<your_username>/slurm_hello`. 

We can submit this job to the Slurm queue at Kimel using the [`sbatch`](https://slurm.schedmd.com/sbatch.html) command, by opening up the terminal, going to the directory containing the script and entering:

```bash
sbatch hello_slurm.sh
```

You will see a message saying:

```bash
Submitted batch job <jobid>
```

Where `jobid` is a number.

#### <a name="inspectingyourfirstjob">Inspecting your first job</a>

Once you've submitted a job you can check its status via the [`sacct`](https://slurm.schedmd.com/sacct.html) command in the terminal. `sacct` will only show your jobs and what their statuses are, unless you specify otherwise using the `-u` option with another username. For example you might see something like:
```
       JobID    JobName  Partition    Account  AllocCPUS      State ExitCode
------------ ---------- ---------- ---------- ---------- ---------- --------
152760       hello_slu+       moby    tigrlab          1  COMPLETED      0:0
152760.batch      batch               tigrlab          1  COMPLETED      0:0
152760.exte+     extern               tigrlab          1  COMPLETED      0:0
```

- `JobID` is an ID which uniquely identifies the job
- `JobName` is the name given to the job. By default it is the name of the script but see `['directives'](#directives)` for details on how to customize this
- `Account` will always be tigrlab on our local Kimel cluster
- `AllocCPUS` is how many CPUs were given to the ask
- `State` describes Job state and can either be `COMPLETED`, `RUNNING`, `PENDING`, or possibly `FAILED`
- `Partition` tells you the partition used. By default **moby** is used but you can specify which partition you want to use manually with `sbatch --partition` or even better, using ['directives'](#directives).

Once the job `State` is `COMPLETED` (it'll take a minute due to `sleep`), we can inspect the outputs - in this case in `/scratch/<your_username>/slurm_hello`:

```
cat scratch/<your_username>/slurm_hello

> Hello world!
> bayes.camhres.ca
```

As you can see, a computer from the Kimel lab has run your script for you (it could even have been your own workstation). However, Slurm provides no guarantee that it will run on any *specific* machine unless you request one by name. Instead, Slurm simply provides you the ability to ensure your script runs *somewhere* on our cluster.

A consequence of this is that if you need your job to access a file or directory, **it must be accessible on all computers in the lab!** If it isn't, the job's machine will fail to find the file and the job will crash. See [**Why are my jobs failing?**](#pitfallsanfaq) for more pitfalls! See also [Logging](#logging) for details on output logs.

#### <a name="cancellingyourjob">Cancelling your Job</a>

To cancel any slurm job the command [`scancel`](https://slurm.schedmd.com/scancel.html) is used. Common ways to cancel jobs is to provide the `JobID`:

```
scancel <jobID1> <jobID2> <jobID3> ...
```

Or if you want to cancel *all your jobs*:

```
scancel -u <your_username>
```

Where the `-u` flag indicates `user`, as it does for `sacct` as well.

### <a name="writingslurmscripts">Writing Slurm scripts</a> ###

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

However, this is inconvenient and should be avoided, as it forces you to correctly retype each directive every time. Instead, follow the below example and place the directives inside the script. This allows you to check the directives for correctness, ensures they *remain* correct, and allows review of directives from past successful script runs.

#### <a name="directives">Directives</a> ####

``` bash
#!/bin/bash
#SBATCH --job-name=script_test
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=2
#SBATCH --mem-per-cpu=1024
#SBATCH --time=00:20:00
#SBATCH --partition=high-moby
#SBATCH --output=/projects/<your_username>/slurm_%A.out
#SBATCH --error=/projects/<your_username>/slurm_%A.err

echo "Hello, this script ran on `hostname -s` on `date --iso`."
```

In this script, we've used a series of directives to effectively delineate how this job is to be performed. Let's lay out what they do and why it matters:

| directive          | short     | meaning                         | important                                  |
|:------------------:|:---------:|:-------------------------------:|:------------------------------------------:|
| `--job-name`       | `-J`      | Give your job a name.           | Make it distinctive.                       |
| `--ntasks`         | `-n`      | How many processes it will run. | Useful for jobs with multiple large steps. |
| `--cpus-per-task`  | `-c`      | Allocate it CPU cores.          | How many will it *actually* use?           |
| `--mem-per-cpu`    | n/a       | Allocate it memory.             | How much does it need per CPU?             |
| `--time`           | `-t`      | Allocate it a time limit.       | This is a *maximum* time.                  |
| `--partition`      | `-p`      | Allocate it to a partition.     | It will get higher priority here.          |
| `--output`/`error` | `-o`/`-e` | The location of your log files. | *Always* in your scratch or projects.      |
| `--gres`           | n/a       | Allocate generic resources.     | GPUs and specific amounts of memory.       |

Using [Resource Allocation](#resourceallocation), [Logging](#logging), and [Array Jobs](#arrayjobs) in Slurm will be covered later in their own respective sections in more detail.

#### <a name="arrayjobs">Array Jobs</a> ####

An [**array job**](https://slurm.schedmd.com/job_array.html) is a special kind of Slurm job which allows you to specify that an identical job script should be executed with some non-identical parameter.

In an array job, the `--array` directive is used to specify a numerical range such as `--array=0-249`. This range acts similarly to `--ntasks=250`, except that it provides a distinguishing variable *within* your script called a `SLURM_ARRAY_TASK_ID`, allowing each copy of the script to run on a different datum. For example:

``` bash
#!/bin/bash
#SBATCH --job-name=slurm_array_test
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --time=01:00:00
#SBATCH --partition=low-moby
#SBATCH --array=0-249%10
#SBATCH --output=/projects/<your_username>/logs/%x_%j_%a.out
#SBATCH --error=/projects/<your_username>/logs/%x_%j_%a.err

slurmarray=(`seq 1 250`)

echo "The number ${slurmarray[$SLURM_ARRAY_TASK_ID]} appeared on `hostname -s`"
```

This script generates a bash array called `slurmarray` filled with the numbers 1 through 250, and echoes a different number on each machine the script executes on. Every time Slurm runs this script it provides a unique value for the variable `SLURM_ARRAY_TASK_ID`, indicating a unique datum each time.[^1]

This works for any data. Consider the following:

``` bash
subjects=(`cat $mydirectory/subjects.txt`)

# ... <do some stuff to "${subjects[$SLURM_ARRAY_TASK_ID]}" here> ...

echo "Subject '${subjects[$SLURM_ARRAY_TASK_ID]}' was processed on `hostname -s`"
```

In this example, as above, `"${subjects[$SLURM_ARRAY_TASK_ID]}"` becomes the name of the subject as it appears in `$mydirectory/subjects.txt`. Note that the array begins at 0; this is because in bash arrays begin at zero, so we begin our Slurm array at zero as well.

If we are batch processing an array of 250 subjects, the default behaviour of Slurm is to attempt to run as many subjects as possible, which can be achieved via `--array=0-249`. However, in the above example, the `array` directive includes a per cent sign followed by an integer, as in `--array=0-249%10`.

This is the `Array Task Throttle`. The `Array Task Throttle` allows you to cap the number of Slurm's simultaneous job attempts. Thus, `--array=0-249%10` runs a Slurm script on each of 250 subjects, but never attempts to run more than 10 subjects at a time.

An array job provides one more advantage aside from the convenience of letting you map a script over a bunch of data: *when you submit a job as an array, the job gains a head*. What does this mean? It means that there will be one number, one Job ID which you can use to control *all* jobs in that array. If you don't want to have to systematically blast updates at every job you submitted, an array allows you to update large numbers of tasks at once, simply by addressing the head of the array. Example:

``` bash
squeue
```

``` log
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
    168943_[10-12]  low-moby Antropos     ttan PD       0:00      1 (Resources)
        164857_157  cudansha dmriprep smansour  R    8:03:30      1 higgs
        164857_156  cudansha dmriprep smansour  R    9:00:16      1 darwin
        164857_155  cudansha dmriprep smansour  R    9:01:39      1 bulbasaur
        164857_154  cudansha dmriprep smansour  R   10:21:31      1 hopper
        164857_152  cudansha dmriprep smansour  R   12:44:26      1 hopper
        164857_150  cudansha dmriprep smansour  R   15:31:57      1 hopper
            168942  low-moby CLZ_CMP_   clevis  R    1:24:57      1 cajal
          168943_9  low-moby Antropos     ttan  R       7:30      1 tesla
          168943_8  low-moby Antropos     ttan  R       8:01      1 talairach
          168943_7  low-moby Antropos     ttan  R       8:05      1 nissl
          168943_6  low-moby Antropos     ttan  R      13:17      1 hopper
          168943_5  low-moby Antropos     ttan  R      13:30      1 hopper
          168911_0  low-moby skullstr     ttan  R    3:06:21      1 davinci
          168911_1  low-moby skullstr     ttan  R    3:06:21      1 davinci
          168911_2  low-moby skullstr     ttan  R    3:06:21      1 franklin
          168911_3  low-moby skullstr     ttan  R    3:06:21      1 franklin
          168911_4  low-moby skullstr     ttan  R    3:06:21      1 kandel
          168911_5  low-moby skullstr     ttan  R    3:06:21      1 kandel
          168911_6  low-moby skullstr     ttan  R    3:06:21      1 mansfield
          168911_7  low-moby skullstr     ttan  R    3:06:21      1 mansfield
          168911_8  low-moby skullstr     ttan  R    3:06:21      1 cajal
          168911_9  low-moby skullstr     ttan  R    3:06:21      1 crick
         168911_10  low-moby skullstr     ttan  R    3:06:21      1 crick
         168911_11  low-moby skullstr     ttan  R    3:06:21      1 lovelace
         168911_12  low-moby skullstr     ttan  R    3:06:21      1 milner
         168911_13  low-moby skullstr     ttan  R    3:06:21      1 penfield
```

In this example, *all* job numbers that begin with the same `Job ID`, ex. 168943, can be simultaneously controlled with `scontrol` by issuing instructions to the that Job ID, **so long as they aren't running yet.** In the above example, the next two jobs in `168943_[10-12]`, can be changed together if the user applies the changes to 168943 and not to either one of 168943_10, 168943_11, or 168943_12 individually. For small Slurm arrays, this is less meaningful, but issuing hundreds or thousands of commands is always worse than issuing one command to hundreds or thousands of array indices.

#### <a name="logging">Logging</a> ####

Logging in Slurm is performed via two directives:`--error` and `--output`. These are very important but reasonably simple directives which specify via absolute filepath the name of a file to which error and output are logged in your script.

Note that in this context, output *does not* refer to the data files or other objects created by the pipelines or other software in your Slurm script. Rather, it refers exclusively to the posix standard streams. This is to say, if your script would ordinarily print text output onto the terminal while running, these directives let you accumulate that output in a text file wherever you want.

For instance, if you have to process 40 subjects, one per compute node, and you want your output to go into `/home/<your_username>/joboutputs/<subjectid>.txt`, and you create that directory in your homedir, your job will fail if the script runs on any computer that isn't the current computer you're working at, *unless the script itself makes the `joboutputs` directory*.

In practice, it's simpler and neater to just have every script write to a common place, like your /scratch or /projects. It's what those directories are for.

Job output and error names may have certain special patterns. Above, for instance, we saw the example `%x_%j_%a.out`; these are replacements which expand to, respectively, the `<job_name>_<job_id>_<array_id>`. Below are the meanings of these expansions

| replacement | meaning                      | example |
|:-----------:|:----------------------------:|:-------:|
| `%A`        | The 'head' of a slurm array. |         |
| `%a`        | The slurm array task index.  |         |
| `%J`        | The                          |         |
| `%j`        |                              |         |
| `%N`        |                              |         |
| `%n`        |                              |         |
| `%s`        |                              |         |
| `%t`        |                              |         |
| `%u`        |                              |         |
| `%x`        |                              |         |

Using these, you can cause Slurm to separate job output by node, by subject, by *`job step`*, and so on.

#### <a name="resourceallocation">Resource Allocation</a> ####

For directives which specify resources such as `time` and `cpus-per-task`, it is important that the provided values be approximately accurate, as they effectively limit your job. Allocations such as these are *constraints*, and cannot be exceeded once the job has begun.

Thus, if you allocate *too few* resources to your job, it may fail to work or may be slow. However, the converse is also true; if you allocate *too many* resources, your job could fail or run slowly.

For example, The Slurm scheduler may believe that no machine in the cluster is powerful enough to satisfy the job's requirements, or else it may provide you with a very small group of machines for your job to run on!

Remember that when allocating resources using directives, you are not providing a *total pool of resources* for all subjects; you are providing a resource granulation *for each subject*. For example:

``` bash
#SBATCH --cpus-per-task=4
#SBATCH --array=0-1
```

These directives are *not* providing for four CPU cores to be divided among two subjects; it is providing for *each subject to be processed by four cores apiece*. If each subject needs only two cores, we have just overserved this job by 100%. Correctly, we would want:

``` bash
#SBATCH --cpus-per-task=2
#SBATCH --array=0-1
```

Another example; let's say you ask for the following resources:

``` bash
#SBATCH --cpus-per-task=8
#SBATCH --mem-per-cpu=8G
#SBATCH --time=00:20:00
```

Intuitively, this seems like a fairly large job, and it is. One might *expect* to get access to 16 of the lab's 32 machines, since we asked for 8 cores per job, and **lab machines generally donate half of their resources**, and 16 of the lab's machines have 16 or more CPU cores.

In reality, though, you would only get 8 machines, because via the `mem-per-cpu` directive, you're *also* asking for machines willing to donate *at least* 32GB of RAM, and not all 16 core machines have that much memory. 

The same would implicity hold true if we used the directive `gres`. With:

``` bash
#SBATCH --cpus-per-task=8
#SBATCH --gres=mem:32K
```

We're asking for the same total amount, 32K megabytes i.e. 32GB of RAM, and this will also restrict us to the same 8 machines.

<a name="gres">GRES</a> are *generic resources*; they stand for any countable thing which can be consumed by a Slurm job.</a> Currently our cluster is configured with only two types of GRES: `gpu` and `mem`.

By specifying `--gres=mem:(COUNT)K` where (COUNT) is one of `8`, `16`, `32`, or `128`, you can generally target a job at groups of machines willing to provide an amount >= (COUNT)GB of memory. This allows you to specifically select machines willing to allocate enough memory for a high-memory job, independently of `cpus-per-task`.

`--gres=gpu:(TYPE):(COUNT)` accepts both a (TYPE) and a (COUNT), where TYPE is one of `titanx` or `qm4k`, of which there are 4 `titanx`s in 1 group of 4, and 7 `qm4k`s in 7 groups of 1. If you ask for `titanx`s, you get `hopper`. If you ask for `qm4k`s, you get at least one of `bulbasaur,darwin,higgs,mendel,purkinje,zerbina,zippy`.

Importantly, `gres=gpu` is **explicitly required** for Slurm jobs to make use of a GPU. CUDA jobs which *do not* use `gres` to allocate one or more GPUs **will be unable to use any GPUs**, as Slurm constrains its jobs from using GPU devices they do not expressly call for in their directives.

Furthermore, it is also possible to request invalid resource allocations. For example, if you request the following:

``` bash
#SBATCH --partition=low-moby
#SBATCH --time=2-12:00:00
```

Which is a request for a time limit of 2 1/2 days to run on low-moby, you will in fact *get zero machines and your job will never start*, since, as you can see by running the `sinfo` command, *machines in low-moby are unwilling to run jobs longer than 32h*. In such situations, you'll receive a notification like the following:

``` bash
sbatch: error: Unable to allocate resources: Requested node configuration is not available
```

The rule when writing resource allocation directives is the following:

*For any set of resources specified by directives, you are limiting yourself to machines willing to provide at least all of them.* Therefore, always allocate the smallest amount of resources that can run your job!

Not only will this make more machines eligible to run parts of your job, it will generally allow machines to run *more simultaneous copies* of your job. A machine with 8 CPU cores can accept one job of 6 cores *or* two jobs of 4 cores, potentially doubling the performance of that job on that machine.

### <a name="pitfallsanfaq">Pitfalls: an FAQ</a> ###

[^1]: If you've never used a bash array before, here's a didactic instance:

``` bash
$ basharray=(itemzero itemone itemtwo)

$ echo "${basharray[1]}"

> itemone
```
