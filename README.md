# slurm-pipeline

A Python class for scheduling [SLURM](http://slurm.schedmd.com/)
([wikipedia](https://en.wikipedia.org/wiki/Slurm_Workload_Manager)) jobs.

## Operation

This repo contains a Python script, `bin/slurm-pipeline.py` that can
schedule programs and/or scripts to be run in an organized pipeline
fashion. I.e., with an understanding of dependencies.

A pipeline run is schedule according to a specification file in JSON
format. A couple of these can be seen in the `examples` directory of the
repo. Here's an example,  `examples/word-count/specification.json`:

```json
{
    "steps": [
        {
            "name": "one-per-line",
            "script": "scripts/one-word-per-line.sh"
        },
        {
            "dependencies": ["one-per-line"],
            "name": "long-words",
            "script": "scripts/long-words-only.sh"
        },
        {
            "collect": true,
            "dependencies": ["long-words"],
            "name": "summarize",
            "script": "scripts/summarize.sh"
        }
    ]
}
```

This example contains almost everything you need to know to set up your own
pipeline specification.

### Steps

A pipeline run is organized into a series of conceptual *steps*. These are
scheduled in the order they appear in the specification file. The actual
running of the programs involved will occur later, in an order that will
depend on how long earlier programs take to complete and on dependencies.

Each step must contain a `name` key. Names must be unique.

Each step must also contain a `script` step. This gives the file name of a
program to be run. The file must exist at the time of scheduling and must
be executable.

Optionally, a step may have a `dependencies` key which gives a list of step
names upon which the step depends. Any dependency named must have already
been defined in the specification.

Optionally, a script may specify `collect` with a `true` value. This will
cause the script to be run when all the *tasks* (see below) started by
earlier dependent steps have completed.

Optionally, a script may specify a `cwd` key with a value that is a
directory that the script should be invoked from. If no directory is given,
the script will be run in the directory where you invoke
`slurm-pipeline.py`.

### Tasks

A step may choose to emit one or more *task*s. It does this by writing
lines such as `TASK: xxx 39749 39750` to its standard output (this may
change). This indicates that a task named `xxx` has been scheduled and that
two SLURM jobs (with ids `39749` and `39750`) will be run to complete
whatever work is needed for the task.

Any other output from a script is ignored (it is however stored in the
specification - see TODO, below).

When a step depends on an earlier step (or steps), its `script` will be
called with a single argument: the name of each task emitted by the earlier
script (or scripts). If a step depends on multiple earlier steps that each
emit the same task name, the script will only be called once with that task
name but after all the tasks from all the dependent steps have finished.

So, for example, a step that starts the processessing of 10 FASTA files
could emit the file names, to trigger ten downstream invocations of a
dependent step's script.

If a script emits `TASK: xxx` with no job ids, the named task will be
passed along the pipeline but dependent steps will not need to wait for any
SLURM job to complete before being invoked. This is useful when a script
can do some work synchronously (i.e., without scheduling via SLURM).

If a step uses `"collect": true`, its script will only be called once. The
script's arguments will be the names of all tasks emitted by the steps it
depends on.

Steps that have no dependencies are called with whatever command-line
arguments are given to `slurm-pipeline.py` (apart from the specification
file, of course).

So for example, given the specification above,

```sh
$ cd examples/word-count
$ slurm-pipeline.py -s specification.json texts/*.txt
```

will cause `one-word-per-line.sh` from the first specification step to be
called with the files matching `texts/*.txt`. The script of the second
script (`long-words-only.sh`) will be invoked multiple times (once for each
`.txt` file). The script will be invoked with the task names emitted by
`one-word-per-line.sh` (the names can be anything desired, and in this case
they are the file names without the `.txt` suffix). The (`collect`) script
of the third step (`summarize.sh`) will be called with the names of *all*
the tasks emitted by its dependency (the second script).

The standard input of invoked scripts is closed.

#### Script environment variables

Two environment variables are set when a step's script is exectued:

* `SP_ORIGINAL_ARGS` contains the (space-separated) list of arguments
  originally passed to `slurm-pipeline.py`. Most scripts will not need to
  know this information, but it might be useful. Scripts with no
  dependencies get these arguments on the command line too.
* `SP_DEPENDENCY_ARG` contains a string that must be used when the script
  invokes `sbatch` to guarantee that the execution of the script does not
  begin until after the tasks from all dependent steps have finished
  successfully.

The canonical way to use `SP_DEPENDENCY_ARG` in a step script is as
follows:

```sh
jobid=`sbatch -n 1 $SP_DEPENDENCY_ARG submit.sh $task | cut -f4 -d' '`
echo "TASK: $task $jobid"
```

This calls `sbatch` with the dependency argument (if any) and
simultaneously gets the job id from the `sbatch` output (`sbatch` prints a
line like `Submitted batch job 3779695`) and the `cut` in the above pulls
out just the job id. The task name is then emitted, along wtih the job id.

A task name may be emitted multiple times by the same script.


### Separation of concerns

`slurm-pipeline.py` doesn't interact with SLURM at all. Actually, the
*only* thing it knows about SLURM is how to construct a dependency argument
for `sbatch` (so it could in theory be generalized to support other
workload managers).

To use `slurm-pipeline.py` you need to make a specification file such as
the one above to indicate the steps in your pipeline, their scripts, and
their dependencies.

Secondly, you need to write the scripts and arrange for them to produce
output and find their inputs. How you do this is totally up to you.

If your pipeline will process a set of files, it will probably make sense
to have an initial script emit the file names (or their basenames) as task
names. That will make it simple for later scripts to find the files from
earlier processing and to make file names to hold their own output.

You can confirm that `slurm-pipeline.py` doesn't even require that SLURM is
installed on a machine by running the examples in the `examples`
directory. The scripts in the examples all do their work synchronously
(they emit fake SLURM job ids using the Bash shell's `RANDOM` variable,
just to make it look like they are submitting jobs to `sbatch`).

### Examples

There are some simple examples in the `examples` directory:

#### Word counting

`examples/word-count` (whose `specification.json` file is shown above) ties
together three scripts to find the most commonly used long words in three
texts (see the `texts` directory). The scripts are in the `scripts`
directory, and each of them reads and/or writes to files in the `output`
directory. You can run the example via

```sh
$ cd examples/word-count
$ make
rm -f output/*
../../bin/slurm-pipeline.py -s specification.json texts/*.txt
```

This example is more verbose in its output than would be typical. Here's
what the `output` directory contains after `slurm-pipeline.py` completes:

```sh
$ ls -l output/
total 104
-rw-r--r--  1 terry  staff  2027 Oct 24 01:02 1-karamazov.long-words
-rw-r--r--  1 terry  staff  3918 Oct 24 01:02 1-karamazov.words
-rw-r--r--  1 terry  staff  3049 Oct 24 01:02 2-trial.long-words
-rw-r--r--  1 terry  staff  7904 Oct 24 01:02 2-trial.words
-rw-r--r--  1 terry  staff  1295 Oct 24 01:02 3-ulysses.long-words
-rw-r--r--  1 terry  staff  2529 Oct 24 01:02 3-ulysses.words
-rw-r--r--  1 terry  staff   129 Oct 24 01:02 MOST-FREQUENT-WORDS
-rw-r--r--  1 terry  staff   168 Oct 24 01:02 long-words-1-karamazov.out
-rw-r--r--  1 terry  staff   163 Oct 24 01:02 long-words-2-trial.out
-rw-r--r--  1 terry  staff   166 Oct 24 01:02 long-words-3-ulysses.out
-rw-r--r--  1 terry  staff   169 Oct 24 01:02 one-word-per-line.out
-rw-r--r--  1 terry  staff   318 Oct 24 01:02 summarize.out
```

The `MOST-FREQUENT-WORDS` file is the final output (produced by
`scripts/summarize.sh`). The `.out` files contain output from the
scripts. The `.words` and `.long-words` files are intermediates made by
scripts. The intermediate files of one step would normally be cleaned up
after being used by a subsequent step.

If you want to check the processing, run `make sh` to see the same thing
done in a shell pipeline. The output should be identical to that in
`MOST-FREQUENT-WORDS`.

#### Simulated BLAST

`examples/blast` simulates the running of
[BLAST](https://blast.ncbi.nlm.nih.gov/Blast.cgi) on a FASTA file. This is
done in 5 steps:

1. Write an initial message and timestamp to a log file (`pipeline.log`).
1. Split the FASTA file into smaller pieces.
1. Run a simulated BLAST script on each piece.
1. Collect the results, sort them on BLAST bit score, and write them to a file (`BEST-HITS`).
1. Write a final message to the log file.

All the action takes place in the same directory and all intermediate files
are removed along the way (uncomment the clean up `rm` line in
`2-run-blast.sh` and `3-collect.sh` and re-run to see the 200 intermediate
files).

#### A more realistic example

Another example can be seen in another of my repos,
[eske-pipeline-spec](https://github.com/acorg/eske-pipeline-spec). You wont
be able to run that example unless you have various bioinformatics tools
installed. But it may be instructive to look at the specification file and
the scripts. The scripts use `sbatch`, unlike those (described above) in
the `examples` directory.

## TODO

* `slurm-pipeline.py` should print the updated specification following scheduling.
  This contains script output, task information, SLURM job ids, scheduling
  timestamps, etc.
* Make `slurm-pipeline.py` be able to accept an in-progress specification
  so it can report on output, tasks, job ids, completion, timing, etc.
* Make it possible to pass the names of the two environment variables to
  the `SlurmPipeline` constructor.
* Make it possible to pass a regex for matching task name and job id lines
  in a script's stdout to the `SlurmPipeline` constructor (instead of
  using the currently hard-coded `TASK: name jobid1 jobid2`).
* Allow for steps to be error-processing ones, that handle failures in
  their dependencies. Or allow a step to have both success and failure
  scripts.
* Better treatment of `stdout` and `stderr` from scripts.
