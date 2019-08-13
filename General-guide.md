# Introduction
This guide will help you understand and use the benchmark tool found in [https://github.com/potassco/benchmark-tool](https://github.com/potassco/benchmark-tool).
This guide will teach you about the xml configuration file, how to use that file to generate benchmarks and lastly, how to run and evaluate those benchmarks.

 # XML Runscript
 Understanding the xml runscript  is the most important part of using the benchmark tool. It controls every aspect of the process. So, it is very important to know how the various components relate to each other and how they form the end benchmark.

## Example Runscript
In the next few sections, I will explain how the example file [example file](https://github.com/potassco/benchmark-tool/blob/master/runscripts/runscript-example.xml) works and how to adapt this to your own purposes.

### Folder structure
When running the runscript (we will see how later) a folder structure is created. The name of the first folder in the tree is given by the first line in the script:
```<runscript  output="output-folder">```
In this case, the name of the of the folder is "output-folder". It is recommended that the name should be changed to something meaningful and representative of the benchmark you are running.

the second folder in the structure is given by the information in line 3:
```<machine  name="zuse"  cpu="24x8xE5520@2.27GHz"  memory="24GB"/>```
The values of *cpu* and *memory* are irrelevant for the naming. The second folder will be named  using the value in *name*. In this case, "zuse".

### Configuration
Line 5 is where we first start setting up how the benchmarks will actually run:
```<config  name="seq-generic"  template="templates/seq-generic.sh"/>```
This line states how a **configuration** will run. The *name* is just in ID that we can reference later. the value on *template* refers to a template file that shows how to run each instance and configuration. 

### The run template

Taking a look at [templates/seq-generic.sh](https://github.com/potassco/benchmark-tool/blob/master/templates/seq-generic.sh) it can seem complicated. The first relevant line, line 4, is just a bash script that will uncompress your instance file if it is compressed. If the file is just a regular text file it will just print it out.
The next relevant line, line 10, is where we actually run the instance with clingo. The first part of the line ```[[ -e .finished ]]``` just checks to see if a .finished file exists. If it exists, the script is done executing. If it doesn't exist, it proceed with running the instance.
```
$CAT "{run.file}" | "{run.root}/programs/runsolver-3.3.4" \
-M 20000 \
-w runsolver.watcher \
-o runsolver.solver \
-W {run.timeout} \
"{run.root}/programs/{run.solver}" {run.args}
```
```{run.file}```refers to the name of the instance. 
```{run.root}``` is the path of the benchmark-tool folder
```{run.timeout}``` is the walltime for this run
```{run.solver}``` is the solver to be used for this run
```{run.args}``` are the arguments to be used by the solver

The script has a default memory limit of 20 000 MB which can be changed by modifying the value of the ```-M``` argument. 
If you don't have the specific runsolver used in the script, you can simply change its name to match the one you have.

### Run settings
The run settings are given in the example from line 7 to line 11. line 7 starts describing how the *system* looks like. The values of *name* and *version* work together to describe the name of a **bash script** named **{name}-{version}** and is found in the **programs** folder of the benchmark-tool. So, in the case of the example, the solver given to the template file described in the configuration is *programs/clingo-4.5.4*. Since this is a regular bash script you can put whatever you want in here.

The value in *measures* describes how the results will be evaluated once the benchmark is done and you have the results. We will go into detail on this later on.

The value of *config* refers to which configuration should be used to run this solver. This is where the value of *name* in line 5 comes into play. 

Small tip: If your instances have an "#include" directive they may not work unless they have **absolute** paths. An easy fix for this is to not pipe the text of the instance and instead put the instance as a regular argument to the solver:
```
"{run.root}/programs/runsolver-3.3.4" \
-M 20000 \
-w runsolver.watcher \
-o runsolver.solver \
-W {run.timeout} \
"{run.root}/programs/{run.solver}" {run.file} {run.args} 
```
### Run arguments
The arguments that are going to be used by clingo are given in line 9.
```<setting  name="setting-1"  cmdline="'--stats --quiet=1,0'"  tag="basic" />```
The value of *name* is an identifier for the arguments given in *cmdline* and will appear as the name of this configuration when evaluating the results. The value of *cmdline* can be any valid string that can be given to the solver you are using. Finally, the value of *tag* is also an identifier for this configuration except that this is the one that is used to reference it withing this runscript.

If you are looking to run more than one configuration, you can just copy and paste the line and edit the values as necesary. Keep in mind that *name* has to be unique *name*, values of tag can be the same and should be the same for configurations that are meant to be run in the same group.

### Defining Jobs
A job is basically where you set the parameters on how long a single clingo run should last and how long all runs should last. In the example file this is defined in two places, line 13 and 15. We will start with line 13.
```<seqjob  name="seq-gen"  timeout="900"  runs="1"  script_mode="timeout"  walltime="50:00:00"  parallel="1"/>```
The benchmark tool can generate two types of jobs. Seqjobs that are meant to be run a on regular machine and pbsjobs that are meant to be run on a cluster with SLURM. Line 13 defines a seqjob, as seen at the beginning of the line. As usual, it has a *name* which is an identifier for this particular job. *timeout* is the maximum amount of time in seconds a single run can take, *runs* is how many runs will be done for each instance. *walltime* is the maximum amount of time that running all instances can take(format is HH:MM:SS).

Line 15 defines a pbsjob:
```<pbsjob  name="pbs-gen"  timeout="1200"  runs="1"  script_mode="timeout"  walltime="23:59:59"  cpt="4"/>```
Most values are the same as with seqjobs. *cpt* defines how many cores can be used per run. *cpt* stands for "cores per task".

For pbsjobs, there is an interaction between *timeout* and *walltime*. When creating the benchmarks, the walltime will always be respected. So, if its not possible to run all instances sequentially inside the walltime specified, they will be split into as many jobs as necessary so that the walltime is never surpassed. For example, if the walltime is 25 hours and you have 100 instances with a timeout of 1 hour each, there will be 4 jobs, each with 25 instances which will be run in parallel.

### Benchmarks - Instance location
Lines 17 to 19 indicate where the instances are located. The value in *name* is the same as usual. The value of path is, of course, the path of the folder containing the instances. The folder can also contain folders in it. If there are several folders with instances, the folder where the instances are located is taken a "domain" and there will be an additional separation of the results using this "domains". 
If there are folders that should not be included in the benchmark instances, the ignore tag can be used to define a *prefix* which will be ignored.

### Projects
Projects are where all of the above elements come together to define a complete benchmark. We will focus on the project defined in lines 25 to 27:
```
<project name="clingo-pbs-job" job="pbs-gen">
<runtag machine="zuse" benchmark="no-pigeons" tag="basic"/>
</project>
```
Previously, all of the components from before where independant of each other. When defining a project we pick the components we want in order to make the benchmark. The values of *job*, *machine* *benchmark* and *tag* refers to previously defined components. The *tag* refers to all settings that have that same tag. There is also a special value that *tag* can take, \*ALL\*, that refers to all settings.

The project in lines 21 to 23 is defined in the same manner. The only difference is that it will use a seqjob instead of a pbsjob.

## Running the script
Running the script is very simple. Simply go to the benchmark-tool folder and run:
```
$ ./bgen path/to/runscript.xml
```
where runscript.xml is the runscript that you want to use. This will create a folder structure using the values given in the script.


## Evaluating
Once the benchmarks have finished running it is time to evaluate. To evualte we have to do 2 things. First, we have to retrieve the results and save them into an xml file. This is done with the "beval" bash script:
```
$ ./beval path/to/runscript.xml > benchmark-evaluated.xml
```
The script writes the results with an xml format to the standard output. We save this output into a file with a name that you choose.

Once we have the evaluated benchmarks we can convert them into a formatted csv file. We do this with the "bconv" script:
```
$ ./bconv benchmark-evaluated.xml results.csv -m time:t

The run time will now be found in a csv named results.csv. 
```