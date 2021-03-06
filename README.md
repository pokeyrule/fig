fig
===
Percy Liang
Last updated Nov. 24, 2012.

General-purpose collection of Java libraries and tools to faciliate writing
research code and managing experiments.  The main features include:

1) Command-line options: your algorithm probably has several free parameters,
which you'd like to specify dynamically from the command line.  Simply define
variables and get them automatically populated.
  
In your code (examples/Sample.java):

    @Option public int N = 5;
    @Option public Random random = new Random(1);

On the command-line (run this from examples after typing make):

    java -cp .:../fig.jar Sample -N 10 -random 42

2) Hierarchical logging: avoid messy log files by organizing them into a
hierarchy.

In your code:

    LogInfo.begin_track("Simulating");
    for (int i = 0; i < N; i++) {
      double x = SampleUtils.sampleGaussian(random);
      LogInfo.logs("Sample %d: %f", i, x);
      stats.add(x);
    }
    LogInfo.end_track();

    LogInfo.begin_track("Statistics");
    LogInfo.logs("Mean: %f", stats.mean());
    LogInfo.logs("Stddev: %f", stats.stddev());
    LogInfo.end_track();

What's printed:

    main() {
      Computing {
        Sample 0
        Sample 1
        Sample 2
      }
      Statistics {
        Mean: -0.431150
        Stddev: 0.493404
      }
    }

Logging can get quite expensive, both in terms of disk space and time spent
writing, and often one has to log selectively.  You can automatically subsample
based on amount of output you want per second.  For example:

    java -cp .:../fig.jar Sample -N 1000000 -msPerLine 1000

will log only once every second, so you get output like this instead of 1000000
lines (if each sample takes longer, then more lines will be printed):

    main() {
      Simulating {
        Sample 0: -0.668791
        Sample 1: -0.880499
        Sample 2: 0.255840
        Sample 3002: -1.525517
        Sample 18323: -0.741311
        Sample 63567: 0.524057
        Sample 188210: -0.886195
        Sample 424063: 0.799244
        Sample 742668: -1.128977
        ... 999991 lines omitted ...
      } [2.5s, cum. 2.5s]
      Statistics {
        Mean: 0.001096
        Stddev: 1.000084
      }
    } [2.5s]

3) Executions:

Suppose you are running your program with 100 different parameters.  How do you
keep track of all the outputs?  Each time you execute a fig program, a new
directory is created which will store all the outputs of that execution along
with the parameters.

    mkdir -p state/execs
    java -cp .:../fig.jar Sample -execPoolDir state/execs

This will create state/execs/0.exec with the following files:

    info.map  log  options.help  options.map  output.map  samples  time.map

The map files contain a set of tab-separated key-value pairs, one per line
containing various statistics of the run, some auto-generated by fig, others
user generated.
 - log: copy of whatever's printed to stdout.
 - options.map: contains all the options processed by the OptionsParser
   (defined by @Option).
 - output.map: Execution.putOutput(key, value) prints to this file.
 - samples: created in this particular code (Execution.getFile("samples") gives
   us the path).

If you run the command again, it will write to state/execs/1.exec, then
state/execs/2.exec, etc.

4) Servlet:

Once you get up to 1000.exec, you'll want a systematic way of viewing the
executions.  fig provides a servlet which allows you to browse executions,
group them, and comment on them.  Note that the executions don't even have to
be generated by fig.  The servlet simply reads tab-separated files from
execution directories named 0.exec, 1.exec, etc., which could be generated by
any program.

The first time, type the following to download Apache Tomcat:

    ./tomcat

By default, the fig servlet is password protected, so you will need to add the
following line into conf/tomcat-users.xml in order to access it:

    <user username="YOUR_USERNAME" password="YOUR_PASSWORD" roles="fig-user"/>

Now, start the server:

    ./tomcat start

Go to http://localhost:8080/fig to see the page.

General points:
 - You will see a table representing the root of a hierarchy.  Each row denotes
   a child item, which you can descend into.
 - Use vi keys ('k' for up, 'j' for down, 'h' for left, 'r' for right) to move
   around.  Type 'o' to open a child item, 'u' to go back to the parent.
 - To reduce server lag, the page does not refresh automatically.  Therefore
   you must refresh manually (press 'R').  You may have to do this several
   times as the first 'R' just triggers asynchronous reloading.

Go into 'domains', then 'examples', then 'execs', then '(all)'.  Each row will
correspond to a different execution, and the columns are the fields, which have
been read from the map files.  You can make notes about executions by going to
the 'note' field of a row, press 'E' (or double click on the cell).  Type
something and press enter.  This will directly save the note to the map file.

To change the columns which are displayed, navigate to 'domains | examples |
fieldSpecs | default' (press 'u' twice, 'o' on fieldSpecs, 'o' on 'default').
Each row here is a description of the field to be displayed.  The 'data' column
here for 'N' is '$options.map:Sample.N', which means that the value displayed
for an execution will be gotten by reading the 'Sample.N' key from the
options.map file.

To add an entry for 'random', type 'N' to create a new item (and enter 'random'
for its name).  Set 'data' to '$options.map:Sample.random'.  You can move the
new row up by pressing 'K' (and 'J' to move down).  These movements are not
recorded, so you must type 'as' to save (alternatively, select the action from
the dropdown box).  To view your changes, go back to 'domains | examples |
execs | (all)'.

You can select items by typing 'x'.  To purge executions, type 'ap'.
Currently, this does not actually delete executions from disk, but just moves
them away to 0.exec.purged, etc. so it doesn't show up in the servlet.

You can monitor the progress of the executions by refreshing the page.  You can
kill any executions by selecting them and typing 'ak' (this just creates a
'kill' file in the execution directory and waits for the process to kill
itself).

5) Java libraries:

fig contains a suite of miscellaneous Java libraries.  Here's a few of them:

 - fig.prob.SampleUtils: sample from various distributions
 - fig.basic.StopWatch: timing pieces of your code
 - fig.basic.ListUtils, fig.basic.MapUtils, fig.basic.NumUtils
 - fig.basic.LispTree: allows parsing/writing of Lisp-like S-expressions.

6) Command-line utilities:

 - bin/q: simple workqueue system for running jobs remotely.
 - bin/execrunner.rb: allows you to manage complex sets of parameters and
   automatically generate all the combinations.
