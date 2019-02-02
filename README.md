# Lab: Hadoop Streaming

1. Start an Amazon Elastic MapReduce (EMR) Cluster using Quickstart with the following setup:
	* In *General Configuration*
		*  Give the cluster a name that is meaningful to you
		*  Un-check *Logging*
		*  Make sure that *Cluster* is selected, NOT *Step execution*
	*  In *Software configuration*
		*  Select `emr-5.20.0` Release from the drop-down
		*  Select the first option under Applications
	*  In *Hardware configuration*
		*  Select `m4.large` as instance types 
		*  Enter `3` for number of instances (1 master and 2 core nodes)
	* In *Security and access*
	* 	Select your correct EC2 keypair or you will not be able to connect to the cluster
	*  Leave everything else the same
	*  Click **Create Cluster**, and wait...

2. Once the cluster is up and running and in "waiting" state, ssh into the master node: `ssh hadoop@[[master-node-dns-name]]`

3. Install git on the master node: `sudo yum install -y git`

3. Clone this repository to the master node. Note: since this is a public repository you do can use the `http` GitHub URL: `https://github.com/bigdatateaching/lab-hadoop-streaming.git`

4. Change directory into the lab: `cd lab-hadoop-streaming` 

5. In this example, you will run a **simulated MapReduce job** on a text file. We say simulated because you will not be using Hadoop to do this but rather a combination of command line functions and pipes that resemble what happens when you run a Hadoop Streaming job on a cluster on a large file. Page 50 of the book shows an example of how to test your mapper and reducer. 

	There is a file in this repository called `shakespeare.txt` which contains all of William Shakespeare's works in a single file. There are also two Python files: a mapper called `basic-mapper.py` and a reducer called `basic-reducer.py`. Open the files and look at the code so you get familiar with what is going on.

	Using Linux pipes, you will pipe the file into the mapper which produces a (key, value) pair in the form of `word\t1`. The `\t` is a tab character. 
	
	The output from the mapper is then piped into the `sort` command, which takes all of the mapper output and sorts it by key.
	
	The output of the `sort` is piped into the reducer, which totals up the sum, by, key.
	
	- Try just the mapper: `cat shakespeare.txt | ./basic-mapper.py` and see the output
	- Now add the `sort` command: `cat shakespeare.txt | ./basic-mapper.py | sort`
	- Now add the reducer: `cat shakespeare.txt | ./basic-mapper.py | sort | ./basic-reducer.py`
	- If you want to save the output to a file, just add a redirect and filename to the end of the string of commands: `cat shakespeare.txt | ./basic-mapper.py | sort | ./basic-reducer.py > shakespeare-word-count.txt`

	This process simulates what happens at scale when running a Hadoop Streaming job on a a larger file. This was a linear process. When run at scale, the following is happening:
	
	- Every block of input data is piped through the mapper program, so you will have a mapper task per block
	- After all the map processes are done, the *Shuffle and Sort* part of the Hadoop framework is performing one large `sort` operation. 
	- Many keys are sent together to a single reduce process. There is a guarantee that all the records with the same key are sent to the same reducer. **A single reducer program will process multiple keys.**

6. Now let's run an actual Hadoop Streaming job, using this mapper and reducer but on a relatively larger dataset. We have compiled the entire [Project Gutenberg](https://www.gutenberg.org/) collection, a set of over 50,000 ebooks in plain text files, into a single file on S3 - `s3://bigdatateaching/gutenberg-single-file.txt`. This is single, large, text file with approximately 700 million (yes, million) lines of text, and about ~30GB in size. 

	For the purposes of this lab, you will work with a subset of the file, the first 25 million lines in a file that is about ~1.3GB, for the sake of speed. You will run a Hadoop Streaming job to do run the word count mapper and reducer to gain experience using Hadoop Streaming.
	
	Run the following command (you can cut/paste this into the command line):
	
	```
	hadoop jar /usr/lib/hadoop/hadoop-streaming.jar \
	-files basic-mapper.py,basic-reducer.py \
	-input s3://bigdatateaching/gutenberg-single-file-25m.txt \
	-output gutenberg-word-count \
	-mapper basic-mapper.py \
	-reducer basic-reducer.py
	```
	* The first line `hadoop jar /usr/lib/hadoop/hadoop-streaming.jar` is launching Hadoop with the Hadoop Streaming jar. A jar is a Java Archive file, and Hadoop Streaming is a special kind of jar that allows you to run non Java programs.
	* The line `-files /home/hadoop/hadoop-streaming/basic-mapper.py,/home/hadoop/hadoop-streaming/basic-reducer.py` tells Hadoop that it needs to "ship" the executable mapper and reducer scripts to every node on the cluster. If you are working on the master node, these files need to be on your **master (remote) filesystem.** Remember, these files do not exist before the job is run, so you need to package those files with the job so they run
	* The line `-input [[input-file]]` tells the job where your source file(s) are. These files need to be either in HDFS or S3. If you specify a directory, all files in the directory will be used as inputs
	* The line line `-output [[output-location]]` tells the job where to store the output of the job, either in HDFS or S3. **This parameter is just a name of a location, and it must not exist before running the job otherwise the job will fail.** In this case, the output of the Hadoop Streamin job will be placed in a directory called `gutenberg-word-count` inside your cluster's **HDFS**. 
	* The line `-mapper basic-mapper.py` specifies the name of the executable for the mapper. Note that you need to ship the programs if they are custom programs. **This file must be an executable shell script or native Linux command.**
	* The line `-reducer basic-reducer.py` specifies the name of the executable for the mapper. Note that you need to ship the programs if they are custom programs. **This file must be an executable shell script or native Linux command.**

	For more information about the Hadoop Streaming parameters, look at the documentation: [https://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html](https://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html)

	* Once the job finishes, look at the results:

	```
	[hadoop@ip-172-31-56-229 lab-hadoop-streaming]$ hadoop fs -ls
Found 1 items
drwxr-xr-x   - hadoop hadoop          0 2019-02-01 23:33 gutenberg-word-count
[hadoop@ip-172-31-56-229 lab-hadoop-streaming]$ hadoop fs -ls gutenberg-word-count/
Found 4 items
-rw-r--r--   1 hadoop hadoop          0 2019-02-01 23:33 gutenberg-word-count/_SUCCESS
-rw-r--r--   1 hadoop hadoop  267630729 2019-02-01 23:32 gutenberg-word-count/part-00000
-rw-r--r--   1 hadoop hadoop  267477980 2019-02-01 23:33 gutenberg-word-count/part-00001
-rw-r--r--   1 hadoop hadoop  267519115 2019-02-01 23:31 gutenberg-word-count/part-00002
[hadoop@ip-172-31-56-229 lab-hadoop-streaming]$ hadoop fs -cat gutenberg-word-count/part-00000 | head
on,	2
**This	2
!"--is	2
!)	8
!..._	2
!_	20
!_...	2
!fit	2
"!--la	1
"!Viel	2
cat: Unable to write to output stream.
[hadoop@ip-172-31-56-229 lab-hadoop-streaming]$
```	
	
	
2. Let's explore scalability and paralleism by increasing the size of the cluster by adding four more *Task* nodes and re-running the streaming job.

	* Go to the EMR Console and click on your current cluster
	* Click on the Hardware tab
	* Click the blue *Add task instance group* button
	* Enter `4` for the number
	* Click *Add*
	* Wait until the TASK group changes status from *Provisioning* to *Running*
	* Run the job again, but now instead of having the output go into HDFS, make the output go into your S3 bucket. You will see that the job runs much faster and there will probably be more reducer processes and therefore more output files.

	


