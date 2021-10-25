+++
title = "STM CTF 2021 Ear To Ear Coding Writeup"
date = "2021-10-25T09:49:22+02:00"
author = "Yigit Colakoglu"
authorTwitter = "theFr1nge"
cover = ""
tags = ["ctf", "coding", "numpy", "stmctf2021"]
keywords = ["ctf", "cybersecurity", "coding", "python"]
description = "The writeup to the Ear to Ear coding challenge on STM CTF 2021"
showFullContent = false
+++

For the challenge, we are given a web server that provides the json data:

```json
{
  "word": "hidden",
  "sequence": "938658411126141947251193886"
}

```

The server also has an endpoint where you can submit an answer and the server
would respond with the flag.  We are also given a file, called *vectors.txt*
that contains one word, and a set of signed decimal numbers.  You can download
the file [here](https://yeetstore.s3.eu-central-1.amazonaws.com/vectors.txt)

OK, that is a cryptic question, thankfully the creators of the question
realized this and provided a hint.  

> Each vector represents the corresponding word in a vector space where the
> similar words are closer to each other. You can use the cosine distances
> between vectors to find the nth most similar words.  You are given a sequence
> of numbers and a word as a starting point.  To find the flag, you need to
> traverse the words using the nth most similar words, based on the given
> sequence.
> 
> Dummy Example (Similar words are chosen randomly, for demonstration): Word:
> Apple Sequence: 213 The 2nd most similar word of 'apple' -> 'pie' The 1st
> most similar word of 'pie' -> 'cake' The 3rd most similar word of 'cake' ->
> 'chocolate' Send chocolate to api to retrieve the flag.

Now that we know what the question is about, it is actually pretty easy. All we
need to do is save each vector on a numpy array, and compare the vector of the
word that is provided to us on the web server with every single vector and get
the n+1th vector that has the smallest distance with the word(because the
closest neighbour to the word would be itself). Lucky for us, we can calculate
the cosine distance if each vector in two matrices using the
(scipy.spatial.distance)[https://docs.scipy.org/doc/scipy/reference/spatial.distance.html]
and numpy's
(argsort)[https://numpy.org/doc/stable/reference/generated/numpy.argsort.html].
Here is the function that return's n'th closest neighbour to a vector in a vector space.

```py
def getnclosest(v,n):
    distances = distance.cdist([v], vectors, 'cosine')
    p = np.argsort(distances)
    return p[0][n-1]
```

Now that we have the basic function, we can easily write some wrapper code that will allow us to
find the resulting word. Here is the full code that should do that:

```py
from scipy.spatial import distance
from tqdm import tqdm
import numpy as np

def vectortostr(vec):
    vs = ""
    for i in range(len(vec)):
        vs += str(vec[i])
        if i != len(vec) - 1:
            vs += " "
    return vs


f = open("vectors.txt", "r")

l1 = f.readline()

y = int(l1.split(" ")[0])
x = int(l1.split(" ")[1])

vectors = [None]*y

wordvectormap = {}

vectorwordmap = {}

print("[INFO]: Loading vectors from file.")
for i in tqdm(range(y)):
    newlist = [None]*(x)
    l = f.readline().split(" ")
    for j in range(1,x+1):
        newlist[j-1] = float(l[j])

    vectors[i] = newlist
    wordvectormap[l[0]] = newlist
    vectorwordmap[vectortostr(newlist)] = l[0]

vectors = np.array(vectors)

print("[INFO]: Done loading vectors. Dropping you into a shell")

def getnclosest(v,n):
    distances = distance.cdist([v], vectors, 'cosine')
    np.fill_diagonal(distances, np.inf)
    return p[0][n-1]

cmd = 0

while cmd != "exit":
    try:
        cmd = input(">> ")
        cmdparts = cmd.split(" ")
        if cmdparts[0] == "search":
            vector = [None]*(x)
            for i in range(1, x+1):
                vector[i-1] = float(cmdparts[i])

            dist, index = tree.query(vector, k=[int(cmdparts[-1])])
            v = tree.data[index][0]
            print(vectorwordmap[vectortostr(v)])

        elif cmdparts[0] == "eartoear":
            w = cmdparts[1]
            vector = np.array(wordvectormap[w])
            sequence = cmdparts[2]
            print("START: " + w)
            for i in sequence:
                cindex = getnclosest(vector, int(i)+1)
                vector = vectors[cindex]
                w = vectorwordmap[vectortostr(vector)]
            print("END: " + w)


    except KeyboardInterrupt:
        break
    except Exception as e:
        print(str(e))

print("\nBye Bye")
```

When we run the code, here is the result:

```
[INFO]: Loading vectors from file.
100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1193514/1193514 [00:10<00:00, 112756.85it/s]
[INFO]: Done loading vectors. Dropping you into a shell
>> eartoear hidden 938658411126141947251193886
START: hidden
brings, 9
gives, 3
could, 8
n't, 6
if, 5
know, 8
n't, 4
think, 1
n't, 1
think, 1
know, 2
mean, 6
know, 1
n't, 4
think, 1
either, 9
unless, 4
because, 7
means, 2
everything, 5
something, 1
everything, 1
means, 9
unless, 3
matter, 8
difference, 8
reasons, 6
END: reasons
>>
```

And after we submit, we get the flag: `STMCTF{Je0pardy_w@ts0n}`. Nice!
