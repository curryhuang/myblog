---
layout: page
title: "A case study of Cost-based MapReduce workflow optimization"
description: ""
---
# A case study of Cost-based MapReduce workflow optimization

**Abstract:** With the growth of Internet, how to deal with big data challenges modern enterprises. MapReduce is an efficient tool to deal with big data. However, many algorithms such as K-Means, matrix multiplication cannot work out in a single MapReduce job efficiently. For such applications, researchers link inter dependent MapReduce jobs together, this is called MapReduce workflow. In this paper, we give a brief introduction to an open source MapReduce workflow engine Crunch and then present a cost-based algorithm to optimize it. We make experiments by a recommender algorithm. The experiments show our optimization is efficient. 

# Introduction

As an efficient tool for big data, MapReduce is more and more used in both industry and research. When the tasks become more complex, many algorithms such as K-Means, matrix multiplication and words co-occurrence cannot work out in a single MapReduce job. They typically consists of a series of inter dependent data operators e.g. loading, filtering, transformation and so on. This is called MapReduce workflow and can be represented as a directed acyclic graph (DAG). A workflow is showed in Figure 1-1, cycles represent a simple operator or MapReduce job, data is represented as rectangles. It is called MapReduce workflow when cycles represent MapReduce jobs. 
   
![1-1](/images/thesis/figure1-1.png)  
**Figure 1-1** A simple workflow  

It is easy to write a MapReduce program by customizing the map and reduce function, but running a MapReduce program on the cloud is not so easy. Take Hadoop as an example, besides writing map and reduce function, programmer need to specify input format(FileInputFormat), output format(FileOutputFormat), the type of input and output key-value pairs, the Mapper and Reducer class and so on. If the algorithm need to iterations, people also need to handle the intermediate input and output in each iteration. It leads programmers cannot focus on the MapReduce program itself. Fortunately, there are some systems named workflow system or workflow engine such as Pig, Hive, Oozie and Cascading to help programmer to deal with such a mass. The workflow engines run on the top of MapReduce, users define MapReduce workflow using traditional program language such as Java (Cascading) or workflow engines specific language such as Hive (HiveQL) and Pig (PigLatin).  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Figure 1-2 shows the state-of-the-art MapReduce workflow engines and their program language. For these workflow engines in Figure 1-2, the performance of the compiled workflow has a huge impact on the efficiency of data processing, it also affect the monetary cost since many cloud platforms use a pay-as-you-go price model. However, many data processing systems based on MapReduce are lack of optimization, these optimization exist in MapReduce itself and the application platform based on MapReduce. Tuning a MapReduce program is not a easy thing, it needs experienced programmers or administrators to modify the parameters of the systems manually, so it depends on the people’s knowledge of MapReduce framework, it is a difficult for the users who are lack of such related tuning experience. 
 
![1-2](/images/thesis/figure1-2.png)  
**Figure 1-2** MapReduce workflow engines  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;To increase the performance of MapReduce applications, more and more researchers are focus on MapReduce framework and its workflow. Nowadays, MapReduce optimization can be divided to these categories: (1)Optimize the MapReduce framework itself, e.g. Hadoop using store and forward model between map and reduce phase, it write the output of mapper to the disk, then reducer read these data from disk as input, this contains so many I/O operations. To solve this inefficiency, some systems like <a href=#ref1>[1]</a> handles intermediate data by pipeline, <a href=#ref2>[2]</a> changes the storage of files to avoid transferring data via network among nodes. (2)Optimize the configuration parameters of Hadoop. Hadoop has more than 190 parameters; it is very difficult to find a good composition of them for the performance. <a href=#ref3>[3]</a><a href=#ref4>[4]</a> both focus on tuning the parameters automatically, <a href=#ref4>[4]</a> presents a cost-based engine called What-if engine to estimate the cost of MapReduce, What-if engine can catch the change of the runtime cost, it uses a random search algorithm to find a parameters’ space with lowest cost. (3)Optimize the MapReduce workflow. If map or reduce functions consume the same data, <a href=#ref5>[5]</a> makes them to share the same data to avoid unnecessary read cost. Some system make a transformation on the workflow, the new workflow has the same input and output as the original one but works more efficient[6]. (4)Optimize the number of nodes in the cluster to save the monetary cost.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In this paper, we mainly focus on optimizing a MapReduce workflow engine Crunch which defines the workflow by Java API. As described above, a good way to optimize workflow is transformation, we will use cost-based transformation to optimize Crunch.  

# Introduce of Crunch

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Crunch is an open source implementation of FlumeJava, it is also a system using Java API to define workflow. The goal of Crunch is let programmer pay more attention to the algorithms rather than a lot of things related to the computation framework such as input/output format, data storage, job cascading and so on. After defined the workflow by Java API, Crunch will compile the user code to a MapReduce workflow called execution plan, then run the execution plan on the Hadoop.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Crunch will optimize the execution plan before submitting. First, Crunch will make a fusion of  parallelDo() operators which have a producer-consumer relationship. If a parallelDo() operator f’s result will be consumed by another parallelDo() operator g, the two parallelDo() operators will fused to a parallelDo() operator g(f()). This will avoid unnecessary input and output, besides, user can write the operators as they think without considering how to link them. Compare to the MapReduce program defined by Hadoop API directly, in Crunch, user do not need to write all the operations in a map or reduce function, user can define each operation in a more natural way, define each operation, Crunch will help user to link all the operations together. This is very useful when a job contains many complex operations, writing each operation separately simplify the functions’ logic and make less errors. Second, Crunch will fuse operations to eliminate intermediate data, make less I/O operations. As the workflow shows in Figure 2-1,  

![2-1](/images/thesis/figure2-1.png)  
**Figure 2-1** A simple workflow in Crunch  

S1-S6 represent different parallelDo() operators, GBK stands for groupByKey() operator, this workflow contains two GBKs and two outputs. Specially, S2 is used twice, if user defines workflow using Hadoop API, he must define a Map-Only job which contains S1 and S2 first, this job will write result to HDFS as intermediate data, then define two MapReduce jobs using this intermediate data as input. In Crunch’s philosophy, node S2 will be duplicated, consumed by S3 and S5 with no intermediate data I/O.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Crunch will split the execution plan into sub graphs, make MapReduce jobs by each sub graph respectively.Crunch provides a simple interface for complex computation tasks, makes writing algorithms on MapReduce much easier. However, as a new project in Apache’s incubator, Crunch has many limitations. One limitation is when split the execution plan, Crunch will write the previous job’s output to HDFS as intermediate data and then read by the next job. For the jobs with dependencies such as producer and consumer, Crunch will delegate all the consumer’s operations to the producer’s reduce stage, this will cause unnecessary I/O cost to decrease the system’s performance. As Figure 2-2 shows, A is the execution plan compiled by Crunch, there are   

![2-2](/images/thesis/figure2-2.png)  
**Figure 2-2** A division of a workflow with dependencies 
  
three operators between two GBKs. Crunch will using the way shown in D to split the workflow. Obviously, such workflow can be split like B or C shown. According to the analytics of MapReduce, not all the jobs are divided like D is optimal since put too many operations into one operator will increase the calculating cost, leading a higher running time. Furthermore, the size and the number of records of operations’ output are different, they will affect the I/O cost of the whole workflow. In Figure 2-2, if the output size of S3 is too large or too many records are output by S3, the I/O cost will be very high. So, the workflow should be divided with the lowest cost for the good performance.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Another limitation in Crunch is the reducer number allocation algorithm, Crunch provide APIs to customize the number of reducer, however, if the number of reducer is too large, some reduce operations cannot be executed at the same time. On the contrary, if the number is too small, the resources cannot be fully used, it will make a low performance. In Crunch APIs, if user does not provide the number of reducer, Crunch set the number of reducer by default. It will use the size of node which is the previous node of GBK e.g. S3 in Figure 2-2 dividing 1GB to determine how many reducers will be set. This approach has two shortcomings. Firstly, the size of S3 is estimated by the input and the parents’ nodes of S3, by default, the selectivity of each operator is 1.2, so the estimation cannot be precise. Secondly, when the output of the parent node of GBK is too large, the number of reducer will be set too large, but when the output size of S3 is smaller than 1GB, the number of reducer will be set to 1, if the operation in reducer is very complex, the input with size 1GB is too large to be handled efficiently. From above, we need to find a way to set the right number of reducer to make full use of resources.   

# How to optimize Crunch

## Profiling

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Some class in Crunch will use some runtime statistics of MapReduce, for instance, in the function getSize() of PCollection, it will call the scaleFactor() of class DoFn’s instance to get the operator’s selectivity. To estimate the size of PCollection more precisely, user need to override the scaleFactor() in DoFn.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In the subclasses of PCollection, the size of class is estimated by calling the parent’s getSize() times scaleFactor() except for InputCollection class, InputCollection’s size is the HDFS files’ size, it can be got by calling Hadoop API. The configureShuffle() of class PGroupedTable will use getSize() to determine the number of reducer. So it is important to estimate the selectivity of PCollection correctly. Besides the selectivity by size, the cost-based algorithm also need selectivity by records, we add a function getSizeByRecord() to get the size of PCollection by records. MapReduce is designed using a lazy mode, say, all the input/output information can be got when the job is running, so for optimizing the workflow, we must find some ways to estimate the selectivity both by size and records before jobs run. <a href="#ref7">[7]</a> runs the MapReduce workflow using default configuration parameters, profiling the jobs and then get the statistic information. <a href="#ref7">[7]</a> is focus on optimizing the parameters of Hadoop workflow, the structure of workflow doesn’t change when the jobs are running, however, in Crunch, the optimization for the execution plan will change the structure of workflow, optimization in <a href="#ref7">[7]</a> doesn’t work in this case.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In this paper, we design a profiler to get the statistic of MapReduce, user can override the scaleFactor() and scaleFactorByRecord() using the result of profiling. scaleFactor() and scaleFactorByRecord() are related to the size and the number of records of each operator, the input size of map dMapInBytes can be got via Hadoop API, the number of records for the map  can be got via Hadoop API as well. The output size of map dMapOutBytes and the number of records output by map dMapOutRecs can be got via the Hadoop’s counter. The output of map is the input of reduce, so dReduceInBytes = dMapOutBytes, dReduceInRecs = dMapOutRecs, the output of reduce can be got by using the same way as getting map’s input. The selectivity calculate as Formula 3.1-3.4,  


	dsMapSizeSel    =  dMapOutBytes / dMapInBytes       (3.1)
	dsMapRecsSel    =  dMapOutRecs / dMapinRecs         (3.2)
	dsReduceSizeSel =  dReduceOutBytes / dReduceInBytes (3.3)
	dsReduceRecsSel =  dReduceOutrecs / dReduceInRecs   (3.4)

	
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;As a plugin of Crunch, the profiler can be turn on or off using a command line parameter, user need to add some code to the workflow for profiling. The most import function in profiler is profile(). The signature of profile() is  

```java
public <K, V> PTable<K, V> profile(
	String stageName,
	Pipeline pipeline,
	PTable<K, V> table,
	DoFn<String, Pair<K, V>> fn,
	PTableType<K, V> outputType
)
```

**Code 3-1** Signature of profile() 

stageName is the name of profiling, for MapReduce job, the profiler need the corresponding PCollection’s name of map and reduce operation, for Map-Only job, the profiler need the PCollection’s name of map. All crunch workflow need a Pipeline instance, table is the last node before GBK, the profiler using fn to serialize and deserialize the data so that profiler can be controlled by command line parameter. outputType is the parameter for the I/O of Crunch.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Code 3-2 is an example of profiling PTable instance userVector, S0 and S1 are two operators in workflow, static function Long_vector() is the converter of PCollection and strings stored in the intermediate file.

```java
userVector = profiler.profile(
	"S0-S1",
	pipeline,
	userVector,
	ProfileConverter.Long_vector(),
	Writables.tableOf(Writables.longs(),
	Writables.vectors()
)
```

**Code 3-2** Profiling a PTable variable named userVector 
 
Figure 3-1 use a part of workflow used in Chapter 5 to show how the profiler works, the left most column is how user defines the workflow, user customizes each operator and links them by Crunch API.  

![3-1](/images/thesis/figure3-1.png)  
**Figure 3-1** A workflow with profiler 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The execution plan is shown in the second column. If turn off the profiler, Crunch will optimize the execution plan, the workflow runs as the third column shows, otherwise, if the profiler is enabled, the workflow runs without optimization as shown in the fourth column.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The result of profiling is a XML file, Figure 3-2 is the result of profiling for the algorithm in Chapter 5 on book-crossing dataset. As Figure 3-2 shows, user can get the selectivity of all map and reduce operations. If the operator is in map, the selectivity is dsMapSizeSel and dsMapRecsSel, when the operator is in reduce, the selectivity represents dsReduceSizeSel and dsReduceRecsSel.  

![3-2](/images/thesis/figure3-2.png)  
**Figure 3-2** Result of profiling

## Cost-based division algorithm

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In this section, we present an algorithm to optimize job division in Crunch. This algorithm calculates the cost of each division and finds the division with lowest cost. Take workflow in Figure 2-2 as an example, for different division B, C and D, each division creates two MapReduce jobs, furthermore, the division affects the output phase of reduce of first job and all the phases of second job. For example, for B in Figure 2-2, the output of S1 will write to file system as intermediate data, the next job reads these data and processes these data by S2 and S3, for D, the intermediate data contains the result of S1, S2 and S3, Crunch will add a Map-Only job to process the intermediate data in this case. The total cost of division can be calculated as following,  

	TotalCost = lastReduceOutCost + nextMapCost   (3.5)

Algorithm 3-1 is the job division algorithm,  
	
	Input:
	Path: The path which the first and the last node are GBK
	MRStat: MapReduce statistic

	Output:
	SplitIndex: The optimized split point

	1: optIndex <- 1
	2: optCost <- getCost(optIndex, Path, MRStat)
	3: foreach index in Path:
	4:     cost <- getCost(index, Path, MRStat)
	5:     if cost < optCost:
	6:         then optCost <- cost
	7:             optIndex <- index
	8: return optIndex
**Algorithm 3-1** Cost-base division algorithm

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Given an execution plan of Crunch, find all the Paths which the head and tail are both GBK, function getCost() calculates the split cost at index using MapReduce statistic MRStat. After the calculating, algorithm 3-1 finds the SplitIndex with lowest cost. The execution plan will be spited into many sub plans, they are submit to cluster by class MRExcutor.  

## Reduce optimization

This section describes how to optimize the number of reducer in Crunch. Figure 3-3 shows the process.  

![3-3](/images/thesis/figure3-3.png)  
**Figure 3-3** Flow chart of reduce optimization  

An optimized system must have a reasonable number of reducer, not beyond the limitation of resources and making full use of resources either. In Hadoop, the number of reduce task can be executed in one node parallel depends on the number of task slots, task slots can be specified by parameter mapred.tasktracker.reduce.task.maximum, the default value is 2. The best number of reducer can be calculated as Formula 3-6 shows,  

	reducers = slots on each node × 0.95 × nodes in cluster   (3.6)
	
Here the coefficient 0.95 is to avoid using up all the resources, otherwise, if a TaskTracker doesn’t report to JobTracker for a long time, the TaskTracker will be killed after some attempts. Hadoop also adds this TaskTracker into a blacklist and will not allocate task to the node. This decreased the performance of cluster.   

## Sampling

When the size of input data is too large, it is unreal to profiling using all the data. In the profiler described in section 3.1, if the size of input data is larger than 10MB, it uses the sampled data to profiling, here, the profiler makes a random sampling on the data, chooses 10% as the input of profiling. The process of sampling shows as Figure 3-4,

![3-4](/images/thesis/figure3-4.png)  
**Figure 3-4** Flow chart of sampling  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;It should compromise between the size of data and the accuracy of estimation, less data has less profiling time while makes more inaccurate, vice versa. 

# Experiment

In this section we use a collaborative recommender algorithm to build a workflow and then verify the correctness and effectiveness for algorithm 3-1. We run this workflow on both Crunch and Cost-based Crunch with algorithm 3-1 (CBCrunch). There are two parts in the experiment: (1) Verifying algorithm 3-1 can find the best split point for the workflow. (2)Verifying algorithm 3-1 can speed up Crunch.

## Setup the environment

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We use a Hadoop cluster running on 4 nodes, all the nodes have a 4 core Intel Xeon X3470 CPU, two of them have 16GB memory, 1TB local storage, this two nodes are namenode and secondary namenode respectively. The other two nodes have 4GB memory, 500GB local storage. The cluster can run 8 maps and 8 reduces task parallel, 2 maps and 2 reduces for each. Hadoop version is 1.0.4, JDK version is 1.7.0, Crunch version is 0.4.0.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The Hadoop parameters we modify in the experiment shows in table 4-1.  

**Table 4-1** Customized Hadoop parameters  

Parameter | Value | Description  
--------- | ----- | -----------  
io.sort.mb | 512 | The size of map in-memory buffer (MB)
dfs.block.size | 268435456 | HDFS block size (Byte)
mapred.task.timeout | 0 | Task timeout，0 represents no timeout
mapred.child.java.opts | -Xmx1024m | The size of JVM heap on each node

Note: For the sake of the limitation of cluster scale, we must set mapred.task.timeout = 0 to turn off the timeout or Crunch cannot complete the job which runs on dataset book-crossing. The TaskTracker will be killed if it doesn’t send the heart beat to JobTracker for a time specified by mapred.task.timeout.

## Dataset and test cases

### Test Case

We rewrite an item-based recommender algorithm in Mahout using Crunch API to build a workflow. The input is a user-item matrix, each intersection of row and column represents user’s rating for item. The algorithm predicts the users’ preference for each item; Choose the top N preferences as result. Here’s a toy example of recommender algorithm. We suppose the input matrix is table 4-2, the steps are as following:  
**Table 4-2** user-item matrix  

&nbsp; | item101 | item102 | item103 | item104
------ | ------- | ------- | ------- | -------
user1 | 3.0 | 5.0 | 2.0 | N/A	
user2 | N/A | 2.0 | 3.0 | 4.0
user3 | 2.0 | N/A | 1.0 | N/A	
user4 | N/A | 3.0 | 4.0 | N/A	
	
(1) Calculating item co-occurrence matrix, the formula is given in Formula 4.1 and Formula 4.2
	r(user,item1,item2) = 1 if item1 and item2 are both rate by user (4.1)
	r(user,item1,item2) = 0 otherwise

	CoOccurrence(i,j)=sum(r(user, i, j)) user from k to m   		 (4.2)
In Formula 4.2, m and n is the number of user and item respectively, here m = n = 4. The co-occurrence matrix of table 3-2 is shown in table 4-3,  

**Table 4-3** item co-occurrence matrix  

&nbsp;| 101 | 102 | 103 | 104
----- | --- | --- | --- | ---
101 | 3 | 1 | 2 | 0
102 | 1 | 3 | 2 | 1
103 | 2 | 2 | 2 | 1
104 | 0 | 1 | 1 | 2

(2) Calculating the preference of user for item
For the symmetry of co-occurrence matrix, we only need to focus on the upper of lower part of it. The value on the diagonal are meaningless and the algorithm will ignore this value. The user’s preference can be got by co-occurrence matrix times user vector. Figure 4-1 shows how to calculate the user3’s preference,  

![4-1](/images/thesis/figure4-1.png)  
**Figure 4-1** Calculation of preference prediction  

In Figure 4-1, we don’t care the prediction of item 101 and 103 marked as blue since they have rated by the user already. The prediction for item 102 is the highest score in the rest predictions, so we use it as recommender. Here we select top 1 because the number of item is small. Take item 102 as an example to describe the result.  

	1×2.0 + 3×0.0 + 2×1.0 + 1×0.0 = 4.0 	(4.3)

Formula 4.3 shows how to predict user3's preference for item102, the more times item102 and one item both rated by user3, the more probability user3 like this item.

### Datasets

We add a filer operator to the item-based algorithm. For the recommender algorithm, inactive user decreases the performance of the algorithm. If the algorithm doesn’t solving the cold start problem, the inactive users will be filtered in most recommender system. Many datasets have filtered the inactive user already such as MovieLens, but some datasets such as Book-Crossing are the original data without filtering.
The datasets used in our experiment are shown in table 4-4,  

**Table 4-4** Dataset used in experiment  

Datasets | Filtered | Description
-------- | -------- | -----------
MovieLens ml-100k | Yes | 100,000 ratings, including 943 users and 1682 movies
MovieLens ml-1m | Yes | 1,000,000 ratings, including 6040 users and 3952 movies
Book-Crossing | No | 1,140,000 ratings, including 278,858 users and 217,379 books

For general purpose, we use both two kinds of datasets, the size of datasets is different as well. We define the inactive user with ratings less than 20 times, for MovieLens dataset, the user who rates less than 20 times are filtered, it will not affect by the operator in our algorithm while for Book-Crossing dataset, the inactive users will be filtered.

## Implementation

This workflow need 8 MapReduce or Map-Only jobs, compiled and optimized by Crunch, there are 4 jobs left, the workflow is shown in Figure 4-2.  

![4-2](/images/thesis/figure4-2.png)  
**Figure 4-2** Workflow of parallel item-based recommendation algorithm  

The key point in optimizing workflow is edge division. The split point is decided by algorithm 3-1. We give a short describe here using table 3-2 to show operations in each operator. The input format is userID::itemID::Rating, e.g. 1::101::3.0, S0 convert the input to user-item pairs such as <1, 101>, <1, 102>, <2, 102>, the next GBK after S0 groups pairs by userID, S1 make the grouped result to user vector, e.g. the user vector of user1 is [101:1.0, 102:1.0, 103:1.0, 104:0.0], S2 filters out the user vectors of which  length are less than a specific value, S3 calculates co-occurrence pairs using user vectors, e.g. <101, 102>, <101, 103>, <102, 103>, the GBK next to S3 groups the pairs and S4 calculates the co-occurrence matrix, e.g. the row in matrix for item 101 is [101:4, 102:1, 103:2, 104:0], the number after the colon is the times of co-occurrence. MapReduce needs all the inputs have the same type, so S5 and S6 wrapper the result of S2 and S4 so that they can be handled by S7, the input record of S7 like <101, [101:4, 102:1, 103:2, 104:0]> or <101, 1:1.0>, S7 merge the two types of inputs into one, S8 calculate the partial product for each output of S7, e.g. the result of S8 is <1, [101:4, 102:1, 103:2, 104:0]>, the output of S8 is merged by GBK and S9, the output of S9 is the predictions for each item. We don’t want to talk more about the recommender algorithm here, we only use it to build a workflow.

## Verifying the correctness

We use a sub plan in Figure 4-2 as the workflow, shows in Figure 4-3,  

![4-3](/images/thesis/figure4-3.png)  
**Figure 4-3** Sub workflow of recommendation algorithm  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The reason why we choose this part of workflow is there are three operators between two GBKs, this make three different divisions. We can also choose the path from input, S6 to S7 as our sub workflow, the number of operators in this path is the same as the previous one, so choose one of them is OK.

### Design

In this part of the experiment, we divide the Figure 4-3 manually using different split point respectively, here the sub workflow will be divided at between S1 and S2 (SplitAt1), S2 and S3 (SplitAt2), S3 and GBK (SplitAt3). There will be two MapReduce jobs after Crunch complied, run the jobs and get the running time. Compare the result of manual division and division via algorithm 3-1. We use MovieLens 100k and 1m datasets as input and define the threshold of active user to 30 ratings.

### Result

Table 4-5 and Figure 4-4 is the result of three different division,  

**Table 4-5** Execution time of different division (sec)  

&nbsp; | SplitAt1 | SplitAt2 | SplitAt3 
------ |--------- | -------- | --------
ml-100k| 96 | 95 | 131 |
ml-1m  | 591 | 541 | 1118 |

![4-4](/images/thesis/figure4-4.png)  
**Figure 4-4** Histogram of execution time  

From table 4-5 and Figure 4-4, SplitAt2’s running time is shortest on both two datasets, it is the optimized split point for workflow in Figure 4-3. Table 4-6 is the result of running algorithm 3-1,  

**Table 4-6** Cost estimation on ml-1m by algorithm 3-1  

Split point | SplitAt1 | SplitAt2 | SplitAt3
----------- | -------- | -------- | --------
Estimated cost(Byte) | 58198754 | 57249538 | 102921120

In table 4-6, SplitAt2 has the lowest estimated cost, it is the same as Figure 4-4. Besides, in table 4-6, the cost of SplitAt2 is a little smaller than SplitAt1 and much smaller SplitAt3, this is also consistent with Figure 4-4. This consistency shows algorithm 3-1 can not only find the split point with lowest cost and also estimate the cost of each division precisely. 

## Verifying the effectiveness

### Design

In this section, we run the workflow in Figure 4-2 on both Crunch and CBCrunch using datasets in table 4-4. Compare the running time of them.

### Result

The result shows in table 4-7 and Figure 4-5,  

**Table 4-7** Execution time on different dataset (sec)  

&nbsp; | ml-100k | ml-1m | book-crossing
-------|---------| ----- | -------------
Crunch | 390 | 5849 | 31227
CBCrunch | 204 | 1223 | 9291

![4-5](/images/thesis/figure4-5.png)  
**Figure 4-5** Histogram of execution time  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Table 4-7 and Figure 4-5 shows algorithm 3-1 improves the performance of Crunch significantly. The speedup of these three datasets is 1.63, 4.78 and 3.36. Although the number of records in ml-1m and book-crossing are almost the same, the number of users and item in book-crossing are both nearly 5 times larger than them are in ml-1m, so the execution time on book-crossing is much longer than it on ml-1m and the speedup on book-crossing is less than it on ml-1m. However, 3 is also an ideal speedup when the execution time is very long.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;When run the workflow using Crunch on book-crossing dataset, for the sake of too many I/O operations, two nodes with less memory and storage in the cluster stop sending heart beat information and all tasks allocated to them are killed. This don’t occur when using CBCrunch.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;This experiment shows optimizing the I/O cost can speed up the workflow as well as make full use of resources. Algorithm 3-1 improve the performance of Crunch effectively.  

# Conclusion and future work

In this paper, we describe how to use MapReduce to deal with big data, introduce some state-of-the-art MapReduce workflow engines, and present a cost-based division algorithm to improve an open source workflow engine Crunch. We do some experiments on different dataset, the result shows we improve the performance of Crunch.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The division algorithm can only handle the workflow which there are only one path between two GBKs. Actually, in some complex algorithm, there may be many paths between two GBKs, the algorithm should consider all the paths’ cost when optimizing the execution plan. We can add time calculations to the profiler so that we can calculate the cost of MapReduce by time, make the cost model more precisely.

# References

<a name="ref1"></a>
[1]. Ekanayake J, Li H, Zhang B, et al. Twister: a runtime for iterative mapreduce[A]. Proceedings of the 19th ACM International Symposium on High Performance Distributed Computing[C]. ACM, 2010. 810-818.  
<a name="ref2"></a>
[2]. Peregrine[EB/OL]. [2012-11-20]. http://peregrine_mapreduce.bitbucket.org/  
<a name="ref3"></a>
[3]. Babu S. Towards automatic optimization of MapReduce programs[A]. Proceedings of the 1st ACM symposium on Cloud computing[C]. NY: ACM, 2010. 137-142.  
<a name="ref4"></a>
[4]. Herodotou H, Dong F, Babu S. MapReduce programming and cost-based optimization? Crossing this chasm with Starfish[J]. Proceedings of the VLDB Endowment, 2011,4(12).  
<a name="ref5"></a>
[5]. Nykiel T, Potamias M, Mishra C, et al. Mrshare: Sharing across multiple queries in mapreduce[J]. Proceedings of the VLDB Endowment, 2010,3(1-2):494-505.  
<a name="ref6"></a>
[6]. Lim H, Herodotou H, Babu S. Stubby: a transformation-based optimizer for MapReduce workflows[J]. Proceedings of the VLDB Endowment, 2012,5(11):1196-1207.  
<a name="ref7"></a>
[7]. Herodotou H, Babu S. Profiling, what-if analysis, and cost-based optimization of MapReduce programs[J]. Proceedings of the VLDB Endowment, 2011, 4(11): 1111-1122.  


