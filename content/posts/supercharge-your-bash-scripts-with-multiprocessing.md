+++
title = "Supercharge Your Bash Scripts with Multiprocessing"
date = "2021-05-05T17:08:12+03:00"
author = "Yigit Colakoglu"
authorTwitter = "theFr1nge"
cover = ""
tags = ["bash", "scripting", "programming"]
keywords = ["bash", "scripting"]
description = "Bash is a great tool for automating tasks and improving you workflow. However, it is ***SLOW***. Adding multiprocessing to the scripts you write can improve the performance greatly."
showFullContent = false
draft=false
+++

Bash is a great tool for automating tasks and improving you workflow. However,
it is ***SLOW***.  Adding multiprocessing to the scripts you write can improve
the performance greatly.

## What is multiprocessing?

In the simplest terms, multiprocessing is the principle of splitting the
computations or jobs that a script has to do and running them on different
processes. In even simpler terms however, multiprocessing is the computer
science equivalent of hiring more than one
worker when you are constructing a building.

### Introducing "&"

While implementing multiprocessing the sign `&` is going to be our greatest
friend.  It is an essential sign if you are writing bash scripts and a very
useful tool in general when you are in the terminal. What `&` does is that it
makes the command you added it to the end of run in the background and allows
the rest of the script to continue running as the command runs in the
background. One thing to keep in mind is that since it creates a fork of the
process you ran the command on, if you change a variable that the command in the
background uses while it runs, it will not be affected.  Here is a simple
example:

```bash
foo="yeet"

function run_in_background(){
  sleep 0.5
  echo "The value of foo in the function run_in_background is $foo"
}

run_in_background & # Spawn the function run_in_background in the background
foo="YEET"
echo "The value of foo changed to $foo."
wait # wait for the background process to finish
```

This should output:

```
The value of foo changed to YEET.
The value of foo in here is yeet
```

As you can see, the value of `foo` did not change in the background process even though
we changed it in the main function.

## Baby steps...

Just like anything related to computer science, there is more than one way of
achieving our goal. We are going to take the easier, less intimidating but less
efficient route first before moving on to the big boy implementation. Let's open up vim and get to scripting!
First of all, let's write a very simple function that allows us to easily test
our implementation:

```bash
function tester(){
  # A function that takes an int as a parameter and sleeps
  echo "$1"
  sleep "$1"
  echo "ENDED $1"
}
```

Now that we have something to run in our processes, we now need to spawn several
of them in controlled manner. Controlled being the keyword here. That's because
each system has a maximum number of processes that can be spawned (You can find
that out with the command `ulimit -u`). In our case, we want to limit the
processes being ran to the variable `num_processes`. Here is the implementation:

```bash
num_processes=$1
pcount=0
for i in {1..10}; do
  ((pcount=pcount%num_processes));
  ((pcount++==0)) && wait
  tester $i &
done
```

What this loop does is that it takes the number of processes you would like to
spawn as an argument and runs `tester` in that many processes. Go ahead and test it out!
You might notice however that the processes are run int batches. And the size of
batches is the `num_processes` variable. The reason this happens is because
every time we spawn `num_processes` processes, we `wait` for all the processes
to end. This implementation is not a problem in itself, there are many cases
where you can use this implementation and it works perfectly fine. However, if
you don't want this to happen, we have to dump this naive approach all together
and improve our tool belt.

## Real Chads use Job Pools

The solution to the bottleneck that was introduced in our previous approach lies
in using job pools.  Job pools are where jobs created by a main process get sent
and wait to get executed. This approach solves our problems because instead of
spawning a new process for every copy and waiting for all the processes to
finish we instead only create a set number of processes(workers) which
continuously pick up jobs from the job pool not waiting for any other process to finish.
Here is the implementation that uses job pools. Brace yourselves, because it is
kind of complicated.

```bash
job_pool_end_of_jobs="NO_JOB_LEFT"
job_pool_job_queue=/tmp/job_pool_job_queue_$$
job_pool_progress=/tmp/job_pool_progress_$$
job_pool_pool_size=-1
job_pool_nerrors=0

function job_pool_cleanup()
{
        rm -f ${job_pool_job_queue}
        rm -f ${job_pool_progress}
}

function job_pool_exit_handler()
{
        job_pool_stop_workers
        job_pool_cleanup
}

function job_pool_worker()
{
        local id=$1
        local job_queue=$2
        local cmd=
        local args=

        exec 7<> ${job_queue}
        while [[ "${cmd}" != "${job_pool_end_of_jobs}" && -e "${job_queue}" ]]; do
                flock --exclusive 7
                IFS=$'\v'
                read cmd args <${job_queue}
                set -- ${args}
                unset IFS
                flock --unlock 7
                if [[ "${cmd}" == "${job_pool_end_of_jobs}" ]]; then
                      echo "${cmd}" >&7
                else
                      { ${cmd} "$@" ; }
                fi

        done
        exec 7>&-
}

function job_pool_stop_workers()
{
        echo ${job_pool_end_of_jobs} >> ${job_pool_job_queue}
        wait
}

function job_pool_start_workers()
{
        local job_queue=$1
        for ((i=0; i<${job_pool_pool_size}; i++)); do
        job_pool_worker ${i} ${job_queue} &
        done
}

function job_pool_init()
{
        local pool_size=$1
        job_pool_pool_size=${pool_size:=1}
        rm -rf ${job_pool_job_queue}
        rm -rf ${job_pool_progress}
        touch ${job_pool_progress}
        mkfifo ${job_pool_job_queue}
        echo 0 >${job_pool_progress} &
        job_pool_start_workers ${job_pool_job_queue}
}

function job_pool_shutdown()
{
        job_pool_stop_workers
        job_pool_cleanup
}

function job_pool_run()
{
        if [[ "${job_pool_pool_size}" == "-1" ]]; then
        job_pool_init
        fi
        printf "%s\v" "$@" >> ${job_pool_job_queue}
        echo >> ${job_pool_job_queue}
}

function job_pool_wait()
{
        job_pool_stop_workers
        job_pool_start_workers ${job_pool_job_queue}
}
```

Ok... But that the actual fuck is going in here???

### fifo and flock

In order to understand what this code is doing, you first need to understand two
key commands that we are using, `fifo` and `flock`. Despite their complicated
names, they are actually quite simple. Let's check their manpages to figure out
their purposes, shall we?

#### man fifo

fifo's manpage tells us that:

```

NAME
       fifo - first-in first-out special file, named pipe

DESCRIPTION
       A  FIFO  special  file (a named pipe) is similar to a pipe, except that
       it is accessed as part of the filesystem.  It can be opened by multiple
       processes for reading or writing.  When processes are exchanging data
       via the FIFO, the kernel passes all data internally without writing it
       to the filesystem.  Thus, the FIFO special file has no contents on the
       filesystem; the filesystem entry merely serves as a reference point so
       that processes can access the pipe using a name in the filesystem.
```

So put in **very** simple terms, a fifo is a named pipe that can allows
communication between processes. Using a fifo allows us to loop through the jobs
in the pool without having to delete them manually, because once we read them
with `read cmd args < ${job_queue}`, the job is out of the pipe and the next
read outputs the next job in the pool. However the fact that we have multiple
processes introduces one caveat, what if two processes access the pipe at the
same time? They would run the same command and we don't want that. So we resort
to using `flock`.

#### man flock

flock's manpage defines it as:

```
 SYNOPSIS
        flock [options] file|directory command [arguments]
        flock [options] file|directory -c command
        flock [options] number

 DESCRIPTION
        This utility manages flock(2) locks from within shell scripts or from
        the command line.

        The  first  and  second of the above forms wrap the lock around the
        execution of a command, in a manner similar to su(1) or newgrp(1).
        They lock a specified file or directory, which is created (assuming
        appropriate permissions) if it does not already exist.  By default, if
        the lock cannot be immediately acquired, flock waits until the lock is
        available.

        The third form uses an open file by its file descriptor number.  See
        the examples below for how that can be used.
```

Cool, translated to modern english that us regular folks use, `flock` is a thin
wrapper around the C standard function `flock` (see `man 2 flock` if you are
interested). It is used to manage locks and has several forms. The one we are
interested in is the third one. According to the man page, it uses and open file
by its **file descriptor number**. Aha! so that was the purpose of the `exec 7<>
${job_queue}` calls in the `job_pool_worker` function. It would essentially
assign the file descriptor 7 to the fifo `job_queue` and afterwards lock it with
`flock --exclusive 7`. Cool. This way only one process at a time can read from
the fifo `job_queue`

## Great! But how do I use this?

It depends on your preference, you can either save this in a file(e.g.
job_pool.sh) and source it in your bash script.  Or you can simply paste it
inside an existing bash script. Whatever tickles your fancy. I have also
provided an example that replicates our first implementation. Just paste the
below code under our "chad" job pool script.

```bash
function tester(){
  # A function that takes an int as a parameter and sleeps
  echo "$1"
  sleep "$1"
  echo "ENDED $1"
}

num_workers=$1
job_pool_init $num_workers
pcount=0
for i in {1..10}; do
  job_pool_run tester "$i"
done

job_pool_wait
job_pool_shutdown
```

Hopefully this article was(or will be) helpful to you. From now on, you don't
ever have to write single threaded bash scripts like normies :)
