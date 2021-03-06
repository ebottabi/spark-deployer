# Cluster configuration file
Here we give two examples of `spark-deployer.conf` (settings commented out with `#` are optional).

## Amazon EC2 example
```
platform = "ec2"

cluster-name = "pishen-spark"

keypair = "pishen"
# pem = "/home/pishen/.ssh/pishen.pem"

region = "us-west-2"

# ami = "ami-acd63bcc"
# user = "ubuntu"
# root-device = "/dev/sda1"

# iam-role = "role_name"

master {
  instance-type = "c4.large"
  disk-size = 8
  driver-memory = "2G"
  # spot-price = "0.105"
}

worker {
  instance-type = "c4.xlarge"
  disk-size = 40
  executor-memory = "6G"
  # spot-price = "0.209"
}

# retry-attempts = 20

spark-tgz-url = "http://d3kbcqa49mib13.cloudfront.net/spark-1.6.0-bin-hadoop2.4.tgz"

# main-class = "mypackage.Main"

# app-name = "my-app-name"

# security-group-ids = ["sg-xxxxxxxx", "sg-yyyyyyyy"]

# subnet-id = "subnet-xxxxxxxx"
# use-private-ip = true

# spark-env = [
#   "SPARK_WORKER_CORES=3",
#   "SPARK_WORKER_MEMORY=6G"
# ]

# destroy-on-fail = true

# enable-s3a = true

# startup-script = [
#   "sudo apt-get -y install openjdk-8-jre &> logfile",
#   "sudo apt-get -y install python-pip &> logfile"
# ]
```
* Please see [here](#common-settings) for the common settings.
* `ami` should be HVM EBS-Backed with Java 7+ installed.
* `root-device` will be your root volume's name that can be enlarged by `disk-size` in `master` and `worker` settings.
* Currently tested `instance-type`s are `t2.medium`, `m3.medium`, and `c4.xlarge`. All the M3, M4, C3, and C4 types should work, please report an issue if you encountered a problem.
* `disk-size` is in GB, which should be at least 8. It resets the size of root partition, which is used by both OS and Spark.
* If `spot-price` is provided, will try to create spot instance with this price as the bid.

## OpenStack example
```
platform = "openstack"

cluster-name = "pishen-spark"

keypair = "pishen"
# pem = "/home/pishen/.ssh/pishen.pem"

auth-url = "http://127.0.0.1:5000/v2.0"
tenant-id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
tenant-name = "pishen"

image-id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
user = "ubuntu"

# add-host-ip = true

master {
  flavor-name = "m1.medium"
  driver-memory = "2G"
}

worker {
  flavor-name = "m1.medium"
  executor-memory = "2G"
}

# retry-attempts = 20

spark-tgz-url = "http://d3kbcqa49mib13.cloudfront.net/spark-1.6.0-bin-hadoop2.4.tgz"

# main-class = "mypackage.Main"

# app-name = "my-app-name"

# security-group-ids = ["xxxxxxxx", "yyyyyyyy"]

network-id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# spark-env = [
#   "SPARK_WORKER_CORES=3",
#   "SPARK_WORKER_MEMORY=6G"
# ]

# destroy-on-fail = true

# enable-s3a = true

# startup-script = [
#   "sudo apt-get -y install openjdk-8-jre &> logfile",
#   "sudo apt-get -y install python-pip &> logfile"
# ]
```
* Please see [here](#common-settings) for the common settings.

## Common settings
* `user` will be the username used to login the machines.
* Make `add-host-ip` to `true` if you want to read the input by `sc.textFile()`.
* `driver-memory` and `executor-memory` are the memory available for Spark, you may subtract 2G from the physically available memory on that machine.
* Some steps of the deployment (e.g. SSH login) may fail at the first time. In default, spark-deployer will retry 20 times before it throw an exception. You can change the number of retries at `retry-attempts`.
* `spark-tgz-url` specifies the location of Spark tarball for each machine to download.
  * You may find one tarball at [Spark Downloads](http://spark.apache.org/downloads.html).
  * To install a different version of Spark, just replace the tarball with the corresponding version.
  * The URL can also be a S3 path starting with `s3://` (Please make sure you have `awscli` installed on your created machines).
  * The URL must ends with `/<spark-folder-name>.tgz` for the auto deployment to work.
* `security-group-ids` specify a list of security groups to apply on the machines.
  * Since Akka use random port to connect with master, the security groups should allow all the traffic between machines in the cluster.
  * Allow port 22 for SSH login.
  * Allow port 8080, 8081, 4040 for web console (optional).
  * Please check [Spark security page](http://spark.apache.org/docs/latest/security.html#configuring-ports-for-network-security) for more information about port settings.
* `spark-env` adds the additional Spark settings to `conf/spark-env.sh` on each node. Note that `SPARK_MASTER_IP`, `SPARK_MASTER_PORT`, `SPARK_PUBLIC_DNS`, and `SPARK_LOCAL_IP` are hard-coded for now.
* `destroy-on-fail`: if set to `true`, destroy the cluster when spark-deployer met an error in `sparkCreateCluster` or `sparkSubmitJob`. Note that you still need to destroy the cluster by yourself if no error happens.
* `enable-s3a`: if set to `true`, add the support for s3a (require hadoop 2.0+). We use the workaround as described [here](http://deploymentzone.com/2015/12/20/s3a-on-spark-on-aws-ec2/).
* `startup-script`: an array of commands that will be executed on each machine before starting Spark master/slave.

## Multiple configs
If you have multiple configurations, you can wrap them in different keys, and specify them by `sparkChangeConfig`, for example
```
child-config-1 = ${default} {
  keypair = "jenkins"
}

child-config-2 = ${default} {
  cluster-name = "pishen-child-spark"
  master.instance-type = "c4.xlarge"
}

default {
  cluster-name = "pishen-spark"
  
  keypair = "pishen"
  
  ...
}
```
Then, after entering sbt, you can type `sparkChangeConfig child-config-1`, it will first load the settings in `default {...}` then overwrite `keypair` with `"jenkins"`.

## Run a batch of commands on master
Provide your own key, which is an array of commands:
```
my-cmd = [
  "sudo pip install awscli",
  "aws s3 cp s3://my-bucket/my-script.sh ./",
  "chmod u+x my-script.sh"
]
```
Then in sbt, type `sparkRunCommands my-cmd` to execute these commands on master machine.
