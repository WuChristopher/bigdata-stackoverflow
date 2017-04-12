# [bigdata-stackoverflow](https://www.coursera.org/learn/scala-spark-big-data/programming/FWGnz/stackoverflow-2-week-long-assignment)
To start, first download the assignment: [stackoverflow.zip](http://alaska.epfl.ch/~dockermoocs/bigdata/stackoverflow.zip). For this assignment, you also need to download the data (170 MB):

http://alaska.epfl.ch/~dockermoocs/bigdata/stackoverflow.csv

and place it in the folder: 𝚜𝚛𝚌/𝚖𝚊𝚒𝚗/𝚛𝚎𝚜𝚘𝚞𝚛𝚌𝚎𝚜/𝚜𝚝𝚊𝚌𝚔𝚘𝚟𝚎𝚛𝚏𝚕𝚘𝚠 in your project directory.

The overall goal of this assignment is to implement a distributed k-means algorithm which clusters posts on the popular question-answer platform StackOverflow according to their score. Moreover, this clustering should be executed in parallel for different programming languages, and the results should be compared.

The motivation is as follows: StackOverflow is an important source of documentation. However, different user-provided answers may have very different ratings (based on user votes) based on their perceived value. Therefore, we would like to look at the distribution of questions and their answers. For example, how many highly-rated answers do StackOverflow users post, and how high are their scores? Are there big differences between higher-rated answers and lower-rated ones?

Finally, we are interested in comparing these distributions for different programming language communities. Differences in distributions could reflect differences in the availability of documentation. For example, StackOverflow could have better documentation for a certain library than that library's API documentation. However, to avoid invalid conclusions we will focus on the well-defined problem of clustering answers according to their scores.

Note: for this assignment, we assume you recall the K-means algorithm introduced during Parallel Programming part of the specialization. You may refer back to the [K-means assignment text](http://alaska.epfl.ch/~dockermoocs/bigdata/kmeans/kmeans.html) for an overview of the algorithm!

## The Data

You are given a CSV (comma-separated values) file with information about StackOverflow posts. Each line in the provided text file has the following format:


1
<postTypeId>,<id>,[<acceptedAnswer>],[<parentId>],<score>,[<tag>]
A short explanation of the comma-separated fields follows.


1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
<postTypeId>:     Type of the post. Type 1 = question, 
                  type 2 = answer.
                  
<id>:             Unique id of the post (regardless of type).
<acceptedAnswer>: Id of the accepted answer post. This
                  information is optional, so maybe be missing 
                  indicated by an empty string.
                  
<parentId>:       For an answer: id of the corresponding 
                  question. For a question:missing, indicated
                  by an empty string.
                  
<score>:          The StackOverflow score (based on user 
                  votes).
                  
<tag>:            The tag indicates the programming language 
                  that the post is about, in case it's a 
                  question, or missing in case it's an answer.
You will see the following code in the main class:


1
2
3
4
5
  val lines   = sc.textFile("src/main/resources/stackoverflow
      /stackoverflow.csv")  
  val raw     = rawPostings(lines)  
  val grouped = groupedPostings(raw)  
  val scored  = scoredPostings(grouped)  
  val vectors = vectorPostings(scored)
It corresponds to the following steps:

𝚕𝚒𝚗𝚎𝚜: the lines from the csv file as strings
𝚛𝚊𝚠: the raw Posting entries for each line
𝚐𝚛𝚘𝚞𝚙𝚎𝚍: questions and answers grouped together
𝚜𝚌𝚘𝚛𝚎𝚍: questions and scores
𝚟𝚎𝚌𝚝𝚘𝚛𝚜: pairs of (language, score) for each question
The first two methods are given to you. You will have to implement the rest.

## Data processing

We will now look at how you process the data before applying the kmeans algorithm.

Grouping questions and answers

The first method you will have to implement is 𝚐𝚛𝚘𝚞𝚙𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜:


1
  val grouped = groupedPostings(raw)
In the 𝚛𝚊𝚠 variable we have simple postings, either questions or answers, but in order to use the data we need to assemble them together. Questions are identified using a "𝚙𝚘𝚜𝚝𝚃𝚢𝚙𝚎𝙸𝚍" == 1. Answers to a question with "𝚒𝚍" == QID have (a) "𝚙𝚘𝚜𝚝𝚃𝚢𝚙𝚎𝙸𝚍" == 2 and (b) "𝚙𝚊𝚛𝚎𝚗𝚝𝙸𝚍" == QID.

Ideally, we want to obtain an RDD with the pairs of (𝚀𝚞𝚎𝚜𝚝𝚒𝚘𝚗, 𝙸𝚝𝚎𝚛𝚊𝚋𝚕𝚎[𝙰𝚗𝚜𝚠𝚎𝚛]). However, grouping on the question directly is expensive (can you imagine why?), so a better alternative is to match on the QID, thus producing an 𝚁𝙳𝙳[(𝚀𝙸𝙳, 𝙸𝚝𝚎𝚛𝚊𝚋𝚕𝚎[(𝚀𝚞𝚎𝚜𝚝𝚒𝚘𝚗, 𝙰𝚗𝚜𝚠𝚎𝚛))].

To obtain this, in the 𝚐𝚛𝚘𝚞𝚙𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜 method, first filter the questions and answers separately and then prepare them for a join operation by extracting the QID value in the first element of a tuple. Then, use one of the 𝚓𝚘𝚒𝚗 operations (which one?) to obtain an 𝚁𝙳𝙳[(𝚀𝙸𝙳, (𝚀𝚞𝚎𝚜𝚝𝚒𝚘𝚗, 𝙰𝚗𝚜𝚠𝚎𝚛))]. Then, the last step is to obtain an 𝚁𝙳𝙳[(𝚀𝙸𝙳, 𝙸𝚝𝚎𝚛𝚊𝚋𝚕𝚎[(𝚀𝚞𝚎𝚜𝚝𝚒𝚘𝚗, 𝙰𝚗𝚜𝚠𝚎𝚛)])]. How can you do that, what method do you use to group by the key of a pair RDD?

Finally, in the description we made QID, Question and Answer separate types, butin the implementation QID is an 𝙸𝚗𝚝 and both questions and answers are of type 𝙿𝚘𝚜𝚝𝚒𝚗𝚐. Therefore, the signature of 𝚐𝚛𝚘𝚞𝚙𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜 is:


1
2
def groupedPostings(postings: RDD[/* Question or Answer */ Posting]): 
    RDD[(/*QID*/ Int, Iterable[(/*Question*/ Posting, /*Answer*/ 
        Posting)])]
This should allow you to implement the 𝚐𝚛𝚘𝚞𝚙𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜 method.

## Computing Scores

Second, implement the 𝚜𝚌𝚘𝚛𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜 method, which should return an RDD containing pairs of (a) questions and (b) the score of the answer with the highest score (note: this does not have to be the answer marked as "𝚊𝚌𝚌𝚎𝚙𝚝𝚎𝚍𝙰𝚗𝚜𝚠𝚎𝚛"!). The type of this scored RDD is:


1
val scored: RDD[(Posting, Int)] = ???
For example, the 𝚜𝚌𝚘𝚛𝚎𝚍 RDD should contain the following tuples:


1
2
3
4
5
((1,6,None,None,140,Some(CSS)),67)
((1,42,None,None,155,Some(PHP)),89)
((1,72,None,None,16,Some(Ruby)),3)
((1,126,None,None,33,Some(Java)),30)
((1,174,None,None,38,Some(C#)),20)
Hint: use the provided 𝚊𝚗𝚜𝚠𝚎𝚛𝙷𝚒𝚐𝚑𝚂𝚌𝚘𝚛𝚎 given in 𝚜𝚌𝚘𝚛𝚎𝚍𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜.

## Creating vectors for clustering

Next, we prepare the input for the clustering algorithm. For this, we transform the 𝚜𝚌𝚘𝚛𝚎𝚍 RDD into a 𝚟𝚎𝚌𝚝𝚘𝚛𝚜 RDD containing the vectors to be clustered. In our case, the vectors should be pairs with two components (in the listed order!):

Index of the language (in the 𝚕𝚊𝚗𝚐𝚜 list) multiplied by the 𝚕𝚊𝚗𝚐𝚂𝚙𝚛𝚎𝚊𝚍 factor.
The highest answer score (computed above).
The 𝚕𝚊𝚗𝚐𝚂𝚙𝚛𝚎𝚊𝚍 factor is provided (set to 50000). Basically, it makes sure posts about different programming languages have at least distance 50000 using the distance measure provided by the 𝚎𝚞𝚌𝚕𝚒𝚍𝚎𝚊𝚗𝙳𝚒𝚜𝚝 function. You will learn later what this distance means and why it is set to this value.

The type of the 𝚟𝚎𝚌𝚝𝚘𝚛𝚜 RDD is as follows:


1
val vectors: RDD[(Int, Int)] = ???
For example, the 𝚟𝚎𝚌𝚝𝚘𝚛𝚜 RDD should contain the following tuples:


1
2
3
4
5
(350000,67)
(100000,89)
(300000,3)
(50000,30)
(200000,20)
Implement this functionality in method 𝚟𝚎𝚌𝚝𝚘𝚛𝙿𝚘𝚜𝚝𝚒𝚗𝚐𝚜 and by using the given the 𝚏𝚒𝚛𝚜𝚝𝙻𝚊𝚗𝚐𝙸𝚗𝚃𝚊𝚐 helper method.

(Idea for test: 𝚜𝚌𝚘𝚛𝚎𝚍 RDD should have 2121822 entries)

## Kmeans Clustering


1
 val means = kmeans(sampleVectors(vectors), vectors)
Based on these initial means, and the provided variables 𝚌𝚘𝚗𝚟𝚎𝚛𝚐𝚎𝚍 method, implement the K-means algorithm by iteratively:

pairing each vector with the index of the closest mean (its cluster);
computing the new means by averaging the values of each cluster.
To implement these iterative steps, use the provided functions 𝚏𝚒𝚗𝚍𝙲𝚕𝚘𝚜𝚎𝚜𝚝, 𝚊𝚟𝚎𝚛𝚊𝚐𝚎𝚅𝚎𝚌𝚝𝚘𝚛𝚜, and 𝚎𝚞𝚌𝚕𝚒𝚍𝚎𝚊𝚗𝙳𝚒𝚜𝚝𝚊𝚗𝚌𝚎.

Note 1:

In our tests, convergence is reached after 44 iterations (for langSpread=50000) and in 104 iterations (for langSpread=1), and for the first iterations the distance kept growing. Although it may look like something is wrong, this is the expected behavior. Having many remote points forces the kernels to shift quite a bit and with each shift the effects ripple to other kernels, which also move around, and so on. Be patient, in 44 iterations the distance will drop from over 100000 to 13, satisfying the convergence condition.

If you want to get the results faster, feel free to downsample the data (each iteration is faster, but it still takes around 40 steps to converge):


1
    val scored  = scoredPostings(grouped).sample(true, 0.1, 0)
However, keep in mind that we will test your assignment on the full data set. So that means you can downsample for experimentation, but make sure your algorithm works on the full data set when you submit for grading.

Note 2:

The variable 𝚕𝚊𝚗𝚐𝚂𝚙𝚛𝚎𝚊𝚍 corresponds to how far away are languages from the clustering algorithm's point of view. For a value of 50000, the languages are too far away to be clustered together at all, resulting in a clustering that only takes scores into account for each language (similarly to partitioning the data across languages and then clustering based on the score). A more interesting (but less scientific) clustering occurs when 𝚕𝚊𝚗𝚐𝚂𝚙𝚛𝚎𝚊𝚍 is set to 1 (we can't set it to 0, as it loses language information completely), where we cluster according to the score. See which language dominates the top questions now?

## Computing Cluster Details

After the call to kmeans, we have the following code in method 𝚖𝚊𝚒𝚗:


1
2
val results = clusterResults(means, vectors)
printResults(results)
Implement the 𝚌𝚕𝚞𝚜𝚝𝚎𝚛𝚁𝚎𝚜𝚞𝚕𝚝𝚜 method, which, for each cluster, computes:

(a) the dominant programming language in the cluster;
(b) the percent of answers that belong to the dominant language;
(c) the size of the cluster (the number of questions it contains);
(d) the median of the highest answer scores.
Once this value is returned, it is printed on the screen by the 𝚙𝚛𝚒𝚗𝚝𝚁𝚎𝚜𝚞𝚕𝚝𝚜 method.

## Questions

Do you think that partitioning your data would help?
Have you thought about persisting some of your data? Can you think of why persisting your data in memory may be helpful for this algorithm?
Of the non-empty clusters, how many clusters have "Java" as their label (based on the majority of questions, see above)? Why?
Only considering the "Java clusters", which clusters stand out and why?
How are the "C# clusters" different compared to the "Java clusters"?
