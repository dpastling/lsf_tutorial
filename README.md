
Working with the LSF Job Scheduler
============================================================

The LSF job scheduler is a tool for distributing batch jobs among the available computational resources on a cluster. Jobs submitted to the queue are run in the background without requiring interaction from the user. It is useful for parallelizing large projects as well as prioritizing jobs from multiple users.

It is important to note that when you log onto the Tesla server, you access the head node which is responsible for farming out the jobs to the other nodes on the cluster. The head node is an underpowered computer node that cannot cannot handle much computation requiring RAM and CPU's. The head node is a place to edit scripts, move files around, and submit jobs to the queue.


The Interactive Shell
------------------------------------------------------------

To gain direct access to one of the compute nodes you can use the `qlogin` command. Once logged into the node, you can run scripts interactively. For example this is a great way to do data exploration and analysis in R. The node will be able to use more memory and CPU cycles than the head node.

    [astling@amc-tesla ~]$ qlogin  
    Job <72216> is submitted to queue <interactive>.  
    <<Waiting for dispatch ...>>  
    <<Starting on compute07>>  
    [astling@compute07 ~]$

Note the change in my prompt. I am no longer in the amc-tesla head node and am now working from the compute07 node. It's important to understand that while I am working from a different node, I am still in the same working directory and still have access to the same files as before. The underlying filesystem remains the same, but the computation happens elsewhere.

When finished with the session:

    [astling@compute07 ~]$ exit
    logout   
    [astling@amc-tesla ~]$ exit
    logout
    Connection to amc-tesla closed.
    [astling@laptop ~]$ 


Submitting Jobs to the Queue with *bsub*
------------------------------------------------------------

The downside to the `qlogin` command is that any running processes are killed as soon as you log out. If your internet connection drops out in the middle of a session, you have to log back in and start over from scratch. To run a job in the background, you will need to submit it to the queue with the `bsub` command. In the example below we will create a script that sleeps for 60 seconds (long enough for us to watch it in the queue), and have it print a simple output.

Let's create an example script  

    [astling@amc-tesla ~]$ echo '#!/usr/bin/env perl' > some_script.pl  
    [astling@amc-tesla ~]$ echo 'sleep 60;' >> some_script.pl  
    [astling@amc-tesla ~]$ echo 'print "Hello World\n";' >> some_script.pl  

Submit the script to the queue  

    [astling@amc-tesla ~]$ bsub "perl some_script.pl > output.txt"  
    Job <72221> is submitted to default queue <normal>.  

Check on the status of the job  

    [astling@amc-tesla ~]$ bjobs  
    JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME  
    72221   astling PEND  normal     amc-tesla               *script.pl Feb 19 13:49  

The `bjobs` command tells us that the job is waiting in the queue for the next available node with the status of PEND. Note that the EXEC_HOST field, which tells us where the script is being executed, is blank. We can see the time the job was submitted which is useful later on when determining how long the job has been running. The other piece of information that is useful is the JOBID which let's us modify `bmod` or kill `bkill` the job after it has been submitted.

Let's check on it again:

    [astling@amc-tesla ~]$ bjobs  
    JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME  
    72221   astling RUN   normal     amc-tesla   compute05   *script.pl Feb 19 13:49  

The job is currently running and has been assigned to compute node #5. Let's check again after 60 seconds  

    [astling@amc-tesla ~]$ bjobs  
    No unfinished job found

Let's check on the output

    [astling@amc-tesla ~]$ cat output.txt  
    Hello World  

It worked!


Naming Jobs
------------------------------------------------------------

In the example above, the job name only lists the last nine characters of the quoted command. It is sometimes helpful to provide a short name to manage multiple jobs.

    [astling@amc-tesla ~]$ bsub -J ShortName "perl some_script.pl"
    JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME  
    72222   astling RUN   normal     amc-tesla   compute11   ShortName  Feb 19 13:55  

If your job name is longer than nine characters, you can display the whole thing using the `bsub -w` or *wide* option. 


Logging standard output and error messages
------------------------------------------------------------

By default, Tesla will email a report when the job has finished if no output has been specified. Note that in the example standard output from the script is captured in the `output.txt` file. If we did not pipe the output, the result would have ended up in the email. Without the email you may wonder if our script ran correctly. A better way to capture a report from the run is to pass along the `-e <stderr file>` and `-o <stdout file>` flags. This pipes the standard output and error into specific files.

Let's add a line of code to our example script to generate an error message.

    [astling@amc-tesla ~]$ echo 'warn "This is an error message\n";' >> some_script.pl  

Now let's test the log files:

    [astling@amc-tesla ~]$ bsub -e script.err -o script.out "perl some_script.pl"

Look at output of each:

    blah  
    blah


If you use the script routinely, the log files will get overwritten each time. To keep a record of each run, you can append the job ID to the end of the file by using the `%J` variable. This way you can link up the log file to each run.

    bsub -o stdout_%J.out -e stderr_%J.err "some_script.pl"

These log files can really add up if you're doing a lot of runs, so it's nice to put the log files in a separate folder.

    mkdir logs
    bsub -o logs/stdout_%J.out -e logs/stderr_%J.err "perl some_script.pl"


Wrapping the submission in a bash script
------------------------------------------------------------

The `bsub` arguments in the above example are a bit cumbersome to type each time. Also can lead to problems later on if you try to reproduce the run and don't remember which arguments you used or which script you ran. It is better to put your job submission in a script and pass it along to `bsub`.  Fire up your favorite text editor and create the following:

    #!/usr/bin/env bash
    #BSUB -J ShortName
    #BSUB -n 2
    #BSUB -R "select[mem>1] rusage[mem=1] span[hosts=1]"
    #BSUB -o logs/stdout_%J.out
    #BSUB -e logs/stderr_%J.err
    
    perl some_script.pl
        

You can submit this job like so

    bsub < script.sh

This way you have a record of each run.


Bailing on a Job
------------------------------------------------------------

If you realize you made some mistake in your code (or your job is running much longer than expected) and need to cancel the job, you can use the `bkill` command. You'll need to figure out the JOBID from `bjobs` and pass that along to `bkill`.

    [astlingd@amc-tesla ~]$ bkill 72221
    Job <72221> is being terminated  


How to request resources
------------------------------------------------------------

Mismanaging resources is a good way to run afoul of other users on the cluster. If your jobs are small or if the queue is not heavily used, jobs can be dispatched in an organic fashion, taking up whatever space is needed. However if the queue is full and you don't specify enough resources, your job and others running on that node can go into suspend mode. On the other hand, if you ask for too many resources, other people will be waiting unnecessary waiting around in the queue for yours to finish. It's good to keep an eye on your jobs to see how many resources they are using and adjust as necessary.

To request CPUs use the `-n` parameter. The command specifies the number of processors required to run the job. In the example below, if 8 of 12 CPUs are being used, the job will remain in the queue until 6 CPUs become available

    bsub -n 6 "some_script.pl"

To reserve RAM, use the `-R` parameter. The `rusage[mem=10]` parameter sets the desired RAM needed to 10 GB, and the `select[mem>10]`, requests that more than 10 GB available to run (rather than just the bare minimum).

    bsub -R "select[mem>10] rusage[mem=10]" "some_script.pl"

To prevent your job from running on multiple nodes use `span`.

    bsub -R "select[mem>10] rusage[mem=10] span[hosts=1]" "some_script.pl"

If you need to adjust the resource limits on a running job, use the `bmod` command. For example suppose you requested 10 GB of RAM and it turns out your job needs only needs 1 GB of RAM. For this we will need to know our JOBID

    bmod 72221 "select[mem>1] rusage[mem=1]"


How to see what resources are available
------------------------------------------------------------

    [astling@amc-tesla ~]$ lshosts
	HOST_NAME      type    model  cpuf ncpus maxmem maxswp server RESOURCES
	amc-tesla    X86_64 Intel_EM  60.0     8    23G    14G    Yes (mvapich mpich2 mg openmpi)
	amc-uriel.c  X86_64 Intel_EM  60.0    12    63G    49G    Yes (mvapich mpich2 mg openmpi)
	nfsmanager0  X86_64 Intel_EM  60.0     8    23G    20G    Yes (mvapich mpich2 openmpi)
	nfsmanager0  X86_64 Intel_EM  60.0     8    23G    20G    Yes (mvapich mpich2 openmpi)
	compute04    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute07    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute06    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute02    X86_64 Intel_EM  60.0    12    79G    24G    Yes (mvapich mpich2 openmpi)
	compute00    X86_64 Intel_EM  60.0    12   189G    24G    Yes (mvapich mpich2 openmpi)
	compute05    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute03    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute01    X86_64 Intel_EM  60.0    12    47G    24G    Yes (mvapich mpich2 openmpi)
	compute08    X86_64 Intel_EM  60.0    16   505G    39G    Yes (mvapich mpich2 openmpi)
	compute10    X86_64 Intel_EM  60.0    12    94G    24G    Yes (mvapich mpich2 openmpi)
	compute13    X86_64 Intel_EM  60.0    24   189G    24G    Yes (mvapich mpich2 openmpi)
	compute12    X86_64 Intel_EM  60.0    12    94G    24G    Yes (mvapich mpich2 openmpi)
	compute11    X86_64 Intel_EM  60.0    12    94G    24G    Yes (mvapich mpich2 openmpi)
	compute14    X86_64 Intel_EM  60.0    24   189G    24G    Yes (mvapich mpich2 openmpi)
	compute09    X86_64 Intel_EM  60.0    12    94G    24G    Yes (mvapich mpich2 openmpi)
	compute15    X86_64 Intel_EM  60.0    32   757G    20G    Yes (mvapich mpich2 openmpi)
	

To take a look at other users in the queue, use `bjobs -u all` (y'all)

    [astling@amc-tesla]$ bjobs -u all
    JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME
    68932   fred    SSUSP normal     amc-tesla   compute06   blastn     Jan 05 02:37
    72200   mary    RUN   normal     amc-tesla   compute06   bowtie     Feb 19 09:01
    72201   joe     RUN   normal     amc-tesla   compute09   cufflinks  Feb 19 10:48
    72205   sally   PEND  normal     amc-tesla               gsnap[1]   Feb 19 14:02
    72205   sally   PEND  normal     amc-tesla               gsnap[2]   Feb 19 14:02
    72205   sally   PEND  normal     amc-tesla               gsnap[3]   Feb 19 14:02
    72205   sally   PEND  normal     amc-tesla               gsnap[4]   Feb 19 14:02

You can restrict the output to just the running jobs with `-r`, just the pending jobs `-p`, or display any suspended jobs with `-s`.


The types of queues available
------------------------------------------------------------

You can see which queues are available by using the `bqueues` command. You can also see how many jobs are in each queue.

    [astling@amc-tesla ~]$ bqueues
	QUEUE_NAME      PRIO STATUS          MAX JL/U JL/P JL/H NJOBS  PEND   RUN  SUSP 
	priority         62  Open:Active       -    -    -    -     0     0     0     0
	interactive      50  Open:Active      48    4    -    -     0     0     0     0
	fast             45  Open:Active       -  144    -    -     0     0     0     0
	night            40  Open:Inact        -    -    -    -     0     0     0     0
	short            35  Open:Active       -   48    -    -     0     0     0     0
	bigmem           30  Open:Active       -    -    -    -     0     0     0     0
	normal           28  Open:Active       -  244    -    -     2     0     2     0
	test             28  Open:Active       -    -    -    -     0     0     0     0
	idle             20  Open:Active     144  144    -    -     0     0     0     0
	gzip              5  Open:Inact      200    4    -    -     0     0     0     0
	

- **normal**: The default queue
- **gzip**: This is for jobs with heavy IO usage such as gzipping a large number of files. The gzip queue is optimized for the NFS manager to run more efficiently  
- **fast**: This is for quick running jobs shorter than 5 minutes. These are given a higher priority than normal jobs so they can be pushed through much more quickly. However if the job takes longer than 5 minutes, it will be killed.  
- **short**: like the fast queue, but with a limit of 15 minutes. The priority is slightly lower than the fast queue, but still higher than the normal queue.
- **night**: This is for non-urgent jobs that can be run at night when supposedly the load on the sustem is much lower (play nice with other users). These jobs get the benefit of a higher priority at night.
- **idle**: This is another *play nice* queue for non-urgent jobs that can be run when tesla is not in use. These jobs will give priority to other users on the system and run when demand is low.
- **bigmem**  : This queue is for jobs that consume very large amounts of memory. Special permission is needed to use this queue.
- **test**: This is dedicated to the compute15 node which has a high number of CPUs and RAM. Like bigmem, this is a good place to submit big, long running jobs. Jobs in the test queue can run separately from the normal queue

 
    bsub -q idle "some_script.pl"


Submitting a job array
------------------------------------------------------------

    #!/usr/bin/env bash
    #BSUB -J ShortName[1-10]
    #BSUB -e logs/test_%J.log
    #BSUB -o logs/test_%J.out
    #BSUB -P Collaborators_name
    
    SAMPLES=(
    apple
    banana
    pear
    orange
    strawberry
    kiwi
    starfruit
    blueberry
    raspberry
    peach
    )
    
    fruit=${SAMPLES[$(($LSB_JOBINDEX - 1))]}
    
    echo "Mmmm..." $fruit
    

If you have a large number of jobs to run and/or they will consume significant resources, it's a good idea to limit the number of jobs that can run at once by appending a `%n` to the end of the job name like so `-J ShortName[1-10]%3`. This will allow only three jobs to run at a time. The others will wait in the queue until it is their turn.
