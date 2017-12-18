# AWS-Kafka
Tutorial how to setup one-node Kafka server on AWS's EC2. 

# Preliminaries
You need:
- AWS Account
- PuTTY (if you are on Windows)

# How to
### Step 1
Launch EC2 Ubuntu Instance. If you have the 12-month free account, use the free micro instance (t2.micro). You can use the default settings. You don't have to use the Ubuntu distribution, you can use whatever Linux/Windows distribution you like, but note that in this tutorial we will be using Ubuntu.

**Important**: AWS will generate new *key pairs* for the instance. SAVE THEM somewhere on you computer (e.g. as `key.pem`)! It is important for the SSH authentication. More details on how to connect to your AWS instance can be found here: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html

**Important**: The EC2 machine will be assigned a public IP adress (IPv4 Public IP). We will use this to connect to the Kafka server from anywhere on the planet. In this tutorial, we will call it: `MACHINE_PUBLIC_IP_ADRESS`.

### Step 2 
To be able to connect to the EC2 machine as Kafka consumer, we must allow the public TCP access to our machine on port 9092.

Go to:

```Security Groups > Inbound rules > set permissions (edit) Custom TCP for all IPs```

### Step 3 (only for Windows)
If you are on Windows, you can use PuTTY for the SSH connection to your EC2 machine. 

Convert `key.pem` to `key.ppk` using PuTTYGen. 

### Step 4
Login to the EC2 instance (on Windows: using PuTTY). 

``` login as: ubuntu ```

### Step 5
Now, update the apt-get: 

```ubuntu@ip-162-31-40-165:~$ sudo apt-get update ```

**Note**: the ip adress in `ubuntu@ip-162-31-40-165` will be different for your machine!
### Step 6
Check if you have Java. Type in:

```ubuntu@ip-172-31-40-165> java -version ```

If the response contains a version number, e.g.:

```
openjdk version "1.8.0_151"
```

then you have Java installed. However, AWS Ubuntu comes usually without Java, so you have to install it first. 

Install Java:

```ubuntu@ip-162-31-40-165:~$ sudo apt-get install default-jre ```

### Step 7
Download Kafka from the official Kafka website, https://kafka.apache.org/quickstart
As of 2017/12/18, one can use following mirror: https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.0/kafka_2.11-1.0.0.tgz. 

You download the package with `wget`, e.g.:

```
ubuntu@ip-162-31-40-165:~$ wget http://apache.miloslavbrada.cz/kafka/1.0.0/kafka_2.11-1.0.0.tgz
```

and unzip 

```
ubuntu@ip-162-31-40-165:~$ tar -xzf kafka_2.11-1.0.0.tgz
```

### Step 8
Go to the Kafka directory

```
ubuntu@ip-172-31-40-165:~$ cd kafka_2.11-1.0.0
```

Now it is time to setup the properties (for public acess). Go to config

```
ubuntu@ip-162-31-40-165:~/kafka_2.11-1.0.0$ cd config
```

and edit `producer.properties` (e.g. using `nano` editor in Ubuntu), by setting 

```bootstrap.servers=MACHINE_PUBLIC_IP_ADRESS:9092```

Next, edit `server.properties` by setting

```advertised.listeners=PLAINTEXT://MACHINE_PUBLIC_IP_ADRESS:9092```
### Step 9
Go back to the Kafka directory

```
ubuntu@ip-162-31-40-165:~/kafka_2.11-1.0.0/config$ cd ..
```

and *start the Zookeeper*: 
```

ubuntu@ip-162-31-40-165:~/kafka_2.11-1.0.0$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

### Step 10
Open another terminal (Windows: new PuTTY connection) to your EC2 machine, go to the Kafka directory and *start the Kafka server*

```
ubuntu@ip-162-31-40-165:~/kafka_2.11-1.0.0$ KAFKA_HEAP_OPTS="-Xmx512M -Xms512M" bin/kafka-server-start.sh config/server.properties 
```

**Note**: `KAFKA_HEAP_OPTS="-Xmx512M -Xms512M"` is here because AWS micro instance (free one) is very small and does not have enough memory for the default Kafka server settings. Explicitly setting the memory boundaries allows us to start the Kafka server. 

### Step 11:
Open another terminal (Windows: new PuTTY connection) to your EC2 machine, go to the Kafka directory and *create a topic*

```
ubuntu@ip-162-31-40-165:~/kafka_2.11-1.0.0$ bin/kafka-topics.sh --create --zookeeper MACHINE_PUBLIC_IP_ADRESS:2181 --replication-factor 1 --partitions 1 --topic test```

You can type in something, e.g.

```
>First message
>Second message
>Third message
```

### Step 12: 
Connect to the server and consume the data by whatever system you want! E.g. using Spark (on Databricks): 

```
import org.apache.spark.sql.functions.{explode, split}

// Setup connection to Kafka
val kafka = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "MACHINE_PUBLIC_IP_ADRESS:9092")   // comma separated list of broker:host
  .option("subscribe", "test")    // comma separated list of topics
  .option("startingOffsets", "latest") // read data from the end of the stream
  .load()
  
  // split lines by whitespace and explode the array as rows of `word`
val df = kafka.select(explode(split($"value".cast("string"), "\\s+")).as("word"))
  .groupBy($"word")
  .count
  
  display(df.select($"word", $"count").orderBy($"count"))
