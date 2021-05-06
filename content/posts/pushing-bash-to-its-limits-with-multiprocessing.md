+++
title = "Pushing Bash to its Limits with Multiprocessing"
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

Bash is a great tool for automating tasks and improving you workflow. However, it is ***SLOW***.
Adding multiprocessing to the scripts you write can improve the performance greatly.

## What is multiprocessing?

In the simplest terms, multiprocessing is the principle of splitting the computations
or jobs that a script has to do and running them on different processes. In even simpler
terms however, multiprocessing is the computer science equivalent of hiring more than one
worker when you are constructing a building.

### Introducing "&"

While implementing multiprocessing the sign `&` is going to be out greatest friend.
It is an essential sign if you are writing bash scripts and a very useful tool in
general when you are in the terminal. What `&` does is that it makes the command
you added it to the end of run in the background and allows the rest of the script
to continue running as the command runs in the background. One thing to keep in mind
is that since it creates a fork of the process you ran the command on, if you change a
variable that the command in the background uses while it runs, it will not be affected.
Here is a simple example:

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

Just like anything related to computer science, there is more than one way of achieving our
goal. We are going to take the easier, less intimidating but less efficient route first
before moving on to the big boy implementation. Let's open up vim and get to scripting!  
First of all, let's write a very simple function that allows us to easily test our
implementation:
```bash
function tester(){ 
  # A function that takes an int as a parameter and sleeps
  echo "$1"
  sleep "$1"
  echo "ENDED $1"
}
```


