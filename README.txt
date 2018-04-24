
These are the instructions for running using the MapR Hadoop Sandbox with Virtualbox
Install and fire up the Sandbox using the instructions here: http://maprdocs.mapr.com/home/SandboxHadoop/c_sandbox_overview.html. 

This is an example using Spark Machine Learning decision trees , written in Scala, 
to demonstrate How to get started with Spark ML on a MapR sandbox 

Use an SSH client such as Putty (Windows) or Terminal (Mac) to login. See below for an example:
use userid: user01 and password: mapr.

For VMWare use:  $ ssh user01@ipaddress 

For Virtualbox use:  $ ssh user01@127.0.0.1 -p 2222 
______________________________________________________

Step 0: 

First compile the project: Select project  -> Run As -> Maven Install
 

You can build this project with Maven using IDEs like Eclipse or NetBeans, and then copy the JAR file to your MapR Sandbox, or you can install Maven on your sandbox and build from the Linux command line, 
for more information on maven, eclipse or netbeans use google search. 

After building the project on your laptop, you can use scp to copy your JAR file from the project target folder to the MapR Sandbox:

use userid: user01 and password: mapr.
For VMWare use:  $ scp  *.jar  user01@ipaddress:/user/user01/. 

For Virtualbox use:  $ scp -P 2222 *.jar  user01@127.0.0.1:/user/user01/.  

Copy the data files from the project data folder to the sandbox using scp to this directory /user/user01/data/ on the sandbox:
flights20170102.json	
flights20170304.json

from the project data folder: 
For VMWare use:  $ scp  *.json user01@ipaddress:/user/user01/.
For Virtualbox use:  $ scp -P 2222 *.json  user01@127.0.0.1:/user/user01/data/. 

You can run also the code in the spark shell by copy pasting the code from the scala file in the spark shell after launching:
 
$spark-shell --master local[2]


____________________________________________________________________

Step 1:  Run the Spark program which will create and save the machine learning model: 

spark-submit --class ml.Flight --master local[2] mapr-es-db-60-spark-flight-1.0.jar

______________________________________________________



- Create the topics to read from and write to in MapR streams:

maprcli stream create -path /apps/stream -produceperm p -consumeperm p -topicperm p
maprcli stream topic create -path /apps/stream -topic flights -partitions 3
maprcli stream topic create -path /apps/stream -topic flightp -partitions 3

to get info on the flights topic :
maprcli stream topic info -path /apps/stream -topic flights

to delete topics:
maprcli stream topic delete -path /apps/stream -topic flights
maprcli stream topic delete -path /apps/stream -topic flightp

- Create the MapR-DB table:
maprcli table create -path /mapr/demo.mapr.com/apps/flights -tabletype json -defaultreadperm p -defaultwriteperm p
____________________________________________________________________


Step 2: Publish flight data to the flights topic

Run the MapR Event Streams Java producer to produce messages with the topic and data file arguments:

java -cp mapr-es-db-60-spark-flight-1.0.jar:`mapr classpath` streams.MsgProducer /apps/stream:flights /apps/data/flights20170102.json

Step 3: Consume from flights topic, enrich with Model predictions, publish to flightp topic 

Run the Spark Consumer Producer (in separate consoles if you want to run at the same time) run the spark consumer with the topic to read from and write to:

spark-submit --class stream.SparkKafkaConsumerProducer --master local[2] spark-ml-flightdelay-1.0-jar-with-dependencies.jar  /apps/stream:flights /apps/stream:flightp

____________________________________________________________________

Step 4: Consume from the flightp topic and write to mapr-db

Run the Spark Consumer with the topic to read from

spark-submit --class stream.SparkKafkaConsumeWriteMapRDB --master local[2] spark-ml-flightdelay-1.0-jar-with-dependencies.jar /user/user01/stream:flightp


