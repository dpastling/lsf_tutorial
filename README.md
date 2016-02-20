
# Working with the LSF Job Scheduler

When you log onto the Tesla server, you access the head node which is an underpowered computer node. The head node is linked to an array of more powerfull nodes that are built for carrying out the big jobs. The head node is a place to edit scripts, move files around, and submit jobs to the queue.

## The Interactive Shell

To gain direct access to one of the compute nodes you can use the `qlogin`command. Once logged into one of the compute nodes, you can run scripts interactivly. For example this is a great way to do data exploration and analysis in R. The node should be able to handle the memory and computational power needed.


    [astling@amc-tesla ~]$ qlogin  
    Job <72216> is submitted to queue <interactive>.  
    <<Waiting for dispatch ...>>  
    <<Starting on compute07>>  
    [astling@compute07 ~]$

Note the change in my prompt. I am no longer in the amc-tesla head node and am now working from the compute07 node. It's important to understand that while I am working from a different node, I am still in the same working directory and still have access to the same files. The underlying filesystem remains the same, but the computation happens elsewhere.

When finished with the session:

    [astling@compute07 ~]$ exit
    logout   
    [astling@amc-tesla ~]$ exit
    logout
    Connection to amc-tesla closed.
    [dastling@laptop ~]$ 


## Submitting Simple Jobs to the Queue

The downside to the `qlogin` command is that any running processeses are killed as soon as you log out. For example if your internet drops out in the middle of a session, you have to log back in and start over from scratch. To execute a long running job, you will need to submit it to the queue with the `bsub` command. In this example we will create a script that pauses for 60 seconds (long enough for us to watch it in the queue), and have it print a simple output.

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

The `bjobs` command tells us that the job is sitting in the queue with the status of Pending. Note that the EXEC_HOST field is blank which means it has not yet been assigned to a node. We can see the time the job was submitted which is useful in determining how long the job has been sitting in the queue. The other piece of information that is useful is the JOBID which let's us modify or kill the job after it has been submitted.

Let's check on it again:

    [astling@amc-tesla ~]$ bjobs  
    JOBID   USER    STAT  QUEUE      FROM_HOST   EXEC_HOST   JOB_NAME   SUBMIT_TIME  
    72221   astling RUN   normal     amc-tesla   compute05   *script.pl Feb 19 13:49  

Let's try again after 60 seconds  

    [astling@amc-tesla ~]$ bjobs  
    No unfinished job found

Let's check on the output

    [astling@amc-tesla ~]$ cat output.txt  
    Hello World  

Congrats! You have run your first job!


## Logging standard output and error messages

If you try running the script without piping the output to a file, you may get an email with the output. Tesla is configured to email the job report once completed if the output is not captured in some other way. 

    [astling@amc-tesla ~]$ rm output.txt
    [astling@amc-tesla ~]$ bsub bsub "perl some_script.pl"

Without the email you may wonder if our script ran correctly or exited with an error. To capture the results of the run, you will need to pass along the `-e <stderr file>` and `-o <stdout file>` flags.

We will add a line of code to the script to generate an error message.

    [astling@amc-tesla ~]$ echo 'warn "This is an error message\n";' >> some_script.pl  

Now let's test the log files:

    [astling@amc-tesla ~]$ bsub -e script.err -o script.out "perl some_script.pl"

Look at output of each:

    blah  
    blah


If you run the script multiple times, the log files will get overwritten each time. To keep a record of each run, you can append the job ID to the end of the file by using the `%J` variable.

    bsub -J better_name -o stdout_%J.out -e stderr_%J.err "some_script.pl"

These log files can add up if you're doing a lot of runs, so it's nice to put the log files in a separate folder.

    mkdir logs
    bsub -J good_name -o logs/stdout_%J.out -e logs/stderr_%J.err "perl some_script.pl"


## Bailing on a Job

If I have made some mistake and need to cancel the job I can use the `bkill` command using the JOBID. An example is if the job is hung (e.g. in an infinate loop) and is taking *much* longer than expected.

    bkill 72221  


## Naming Jobs

    bsub -J good_name "perl some_script.pl"


## How to set the number of processes

    bsub -n 12 "some_script.pl"

## How to set the RAM requirements

    bsub -R "???" "some_script.pl"

## How to see which nodes are available

    bqueues
    lshosts

## How to submit to a specific queue

    bsub -q idle "some_script.pl"


## Putting it all together

Let's put all the bsub arguments in one line.

    [astling@amc-tesla ~]$ bsub -J good_name -n 12 -R "??" -o logs/stdout_%J.out -e logs/stderr_%J.err "perl some_script.pl"

This is a bit cumbersome to type each time. Also can lead to problems later on if you try to reproduce the run and don't remember which arguments you used or which script you ran. It is better to put your job submission in a script and pass it along to bsub.  Fire up your favorite text editor and create the following:

    #!/usr/bin/env bash
    #BSUB -J good_name
    #BSUB -n 12
    #BSUB -R ???
    #BSUB -o logs/stdout_%J.out
    #BSUB -e logs/stderr_%J.err
    
    set ???
    
    perl some_script.pl
        

You can submit this job like so

    bsub < script.sh

This way you have a record of how you submitted the run.

## Submitting multiple jobs

- config script
- job array

