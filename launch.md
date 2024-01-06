# Deploy Hadoop on AWS EC2

> This tutorial is based on [How to create a Hadoop Cluster for free in AWS Cloud?](https://medium.com/analytics-vidhya/how-to-create-a-hadoop-cluster-for-free-in-aws-cloud-a95154980b11).

## Prerequisites
 - A free AWS account, [to register](http://aws.amazon.com/free).

*Note: we assume you use the 12-Month free tier resources (750 hours/month + 30GB storage/year) provided by AWS, see [this page](https://aws.amazon.com/free/free-tier/) for more information.*

## Step 1 Create a key pair
 - Go to https://aws.amazon.com/ -> `Sign In to the Console`.
 - `Sevices` -> `Compute` -> `EC2`.
 - Left menu -> `Network & Security` -> `Key Pairs`.
 - `Create key pair`
   - Input `Name`, e.g., `cs230`.
   - Select `Private key file format`, use `.pem` for Linux/Mac or `.ppk` for Windows.
   - Confirm `Create key pair`.
   - Save the file to your local computer for future use.

## Step 2 Create 3 Amazon EC2 Ubuntu instances
 - Left menu -> `Instances` -> `Instances`.
 - `Launch instances`
   - Under tab `Application and OS Images (Amazon Machine Image)`, select `Ubuntu`. (By default, the AMI is `Ubuntu Server 22.04 LTS (HVM), SSD Volume Type`)
   - Under tab `Instance type`, select `t2.micro` (by default).
   - Under tab `Key pair`, select the one created in Step 1.
   - On the right `Summary`, select `3` in `Number of instances`, then `Launch instance`.
   - Click `View all instances` and wait for the 3 instances' statuses become `running`.
 - Use terminal to connect to the instances.
   - On Linux/Mac, 
```bash
ssh –i <pem_filepath> ubuntu@<public_ip>

# `<pem_filepath>` is the path to the `.pem` file downloaded in Step 1, 
# `<public_ip>` is the `Public IPv4 address` in the details of each instance.
# NOTE: If see `Permission denied (publickey).` error, 
#       use `chmod 400 <pem_filepath>` to update the permissions to the `.pem` file.
```

## Step 3 Java Installation
(On all 3 instances)
 - Install Java and verify installation using the following commands.
```bash
sudo apt update
sudo apt-get install default-jdk
java -version
```
 - Set up the `JAVA_HOME` environment variable.
Modify the environment file
```bash
sudo vi /etc/environment
## Add the following line to the file and save
JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
```
Load the environment file in the current session
```bash
source /etc/environment
```
Verify the `JAVA_HOME` variable has value.
```bash
echo $JAVA_HOME
```

## Step 4 Setting up password-less SSH login between instances
 - Assign names to the 3 instances: `Master`, `Worker1`, `Worker2`.
 - Generate the public key and private key on all 3 instances.
```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
```
 - Show the public key on all 3 instances.
```bash
cat .ssh/id_rsa.pub
```
 - Copy and append all 3 public keys to all 3 instances' `.ssh/authorized_keys` file.
 - Verify that you can ssh login in all the following directions without password.

On `Master` run the following,
```bash
# From Master to Master
ssh <public-ip-of-master>
ssh <private-ip-of-master>

# From Master to Worker1
ssh <public-ip-of-worker1>
ssh <private-ip-of-worker1>

# From Master to Worker2
ssh <public-ip-of-worker2>
ssh <private-ip-of-worker2>
```
On `Worker1` run the following,
```bash
# From Worker1 to Worker1
ssh <public-ip-of-worker1>
ssh <private-ip-of-worker1>

# From Worker1 to Master
ssh <public-ip-of-master>
ssh <private-ip-of-master>
```
On `Worker2` run the following,
```bash
# From Worker2 to Worker2
ssh <public-ip-of-worker2>
ssh <private-ip-of-worker2>

# From Worker2 to Master
ssh <public-ip-of-master>
ssh <private-ip-of-master>
```

## Step 5 Hadoop Installation & Configuration
 - On all 3 nodes: Download Hadoop and gunzip it to the same folder location.
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz

tar -zxvf hadoop-3.3.6.tar.gz 
```
 - On all 3 nodes: Append the following lines in `<HadoopInstallationFolder>/etc/hadoop/core-site.xml` file inside the `<configuration>`. Replace the ip with your `<private-ip-of-master>`.
    - HINT: Refer here for instructions on how to add an IP property to a configuration: https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/conf/Configuration.html 
    - **Note:** For the default network security setting of AWS instance, port `54310` on `Master` node is not accessible from worker nodes. 
    - To allow access, go to the `AWS Console` -> `EC2 instances` -> Click `Master` instance -> `Security` -> Click the security group link -> `Edit inbound rules` -> `Add rule` -> `Type = Custom TCP`, `Port range = 54310`, `Source = 172.31.0.0/16` -> `Save rules`.
    - **Note:** When you upload data to HDFS, the `Master` node will use the port `9866` to connect with `Worker` nodes, so you need to allow access of port `9866` to `172.31.0.0/16` ip ranges on both `Worker` nodes too.
```bash
<property>
<name>fs.default.name</name>
<value>hdfs://172.31.28.18:54310</value>
</property>
```
 - On `Master` node only: Clear contents of the file `<HadoopInstallationFolder>/etc/hadoop/workers` and add the private ips of all 3 nodes (including master). This step gives the list of all the machines who can be workers for Hadoop cluster.
 - On all 3 nodes: Set `JAVA_HOME` in `<HadoopInstallationFolder>/etc/hadoop/hadoop-env.sh`
```bash
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
```
 - On `Master` node only: Run the following command in `<HadoopInstallationFolder>/bin/` folder to format the HDFS system.
```bash
./hadoop namenode -format
```
 - On `Master` node only: Run the following command in `<HadoopInstallationFolder>/sbin/` folder to start your cluster. It will take a while.
```bash
./start-dfs.sh
```
 - Check the status of Hadoop cluster by entering `<public-ip-of-master>:9870` in your browser. If you reach a web page with “Namenode Information” you have started Hadoop successfully. You should have 3 live nodes listed.
   - **Note:** For the default network security setting of AWS instance, this port is not accessible. 
   - To allow access, go to the `AWS Console` -> `EC2 instances` -> Click `Master` instance -> `Security` -> Click the security group link -> `Edit inbound rules` -> `Add rule` -> `Type = Custom TCP`, `Port range = 9870`, `Source = My IP` -> `Save rules`.
 - To use HDFS: On `Master` node, run the following command in `<HadoopInstallationFolder>/bin/` folder
```bash
./hadoop fs -help

# Basic usage
./hadoop fs -ls /  # list files under root directory
./hadoop fs -put localFile.txt /hdfsFile.txt  # copy a local file to hdfs under /
./hadoop fs -cat /hdfsFile.txt  # cat an hdfs file
```

## Congratulations! You have deployed Hadoop on an AWS cluster of 3 machines!

### Important !!
* ***Please remember to stop the AWS instances when you don't use them! You only have 750 hours per month for free.***
* ***Whenever you restart the AWS instances, the IP addresses may be changed. You need to modify the settings of AWS and Hadoop accordingly!***

# Appendix
 - Official Doc: [Hadoop Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)
