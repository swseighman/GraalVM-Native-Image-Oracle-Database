# GraalVM Native Image with Oracle Database XE

### Overview

### Credits
The contents of this tutorial is based on a video originally recorded by **Oleg Šelajev**, see [this link](https://www.youtube.com/watch?v=QP83j_Q0CjE&list=PLirn7Sv6CJgGEBn0dVaaNojhi_4T3l2PF&index=2).


### Build an Oracle Database 18c XE Container

Create the following directories on your local filesystem:

- `/home/<username>/oradata`
  - contains database and required database configuration files and preserves the database’s data and configuration files on the host file system in the event the container is deleted
- `/home/<username>/scripts/setup`
  - contains either shell or SQL scripts that are executed once after the database setup (creation) has been completed
- `/home/<username>/scripts/startup`
  - contains either shell or SQL scripts that are executed every time the container starts

The directories must be writable by user `UID 54321`, which is the Oracle user within the container. With root access, change the ownership of the directory:

```
$ chown 54321:54321 /home/<username>/oradata
$ chown 54321:54321 /home/<username>/scripts/setup
$ chown 54321:54321 /home/<username>/scripts/startup
```
Clone the Oracle container image repository:
```
$ git clone https://github.com/oracle/docker-images.git
```
Change to the directory containing the Oracle Database container files:
```
$ cd docker-images/OracleDatabase/SingleInstance/dockerfiles
```
Build the database image:
```
$ ./buildDockerImage.sh -v 18.4.0 -x
```
Check that the image was created:
```
$ docker images
REPOSITORY         TAG          IMAGE ID       CREATED         SIZE
oracle/database    18.4.0-xe    827aba9c0902   4 days ago      6.03GB
```
Run the container:
```
$ docker run --rm --name myexdb-1 \
-p 1521:1521 -p 5500:5500 \
-e ORACLE_PWD=mysecurepassword \
-v /home/<username>/oradata:/opt/oracle/oradata \
-v /home/<username>/scripts/setup:/opt/oracle/scripts/setup \
-v /home/<username>/scripts/startup:/opt/oracle/scripts/startup \
oracle/database:18.4.0-xe

The Oracle base remains unchanged with value /opt/oracle
#########################
DATABASE IS READY TO USE!
#########################
The following output is now a tail of the alert.log:
2021-05-03T16:34:55.661012+00:00
QPI: opatch file present, opatch
QPI: qopiprep.bat file present
2021-05-03T16:34:55.849888+00:00
XEPDB1(3):Opening pdb with Resource Manager plan: DEFAULT_PLAN
Pluggable database XEPDB1 opened read write
Starting background process CJQ0
2021-05-03T16:34:55.974901+00:00
CJQ0 started with pid=41, OS id=319
Completed: ALTER DATABASE OPEN
2021-05-03T16:34:56.785318+00:00
Shared IO Pool defaulting to 48MB. Trying to get it from Buffer Cache for process 94.
===========================================================
Dumping current patch information
===========================================================
No patches have been applied
===========================================================

```

Check to confirm it's running:
```
$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED           STATUS     PORTS                                                            NAMES
cd58d5a0f3a1   oracle/database:18.4.0-xe    "/bin/sh -c 'exec $O…"   48 minutes ago   Up 48 minutes (healthy)   0.0.0.0:1521->1521/tcp, 0.0.0.0:5500->5500/tcp     myexdb-1

```
Install SQL*PLUS client (see [link](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html)) and connect to the running database:
```
$ sqlplus system/mysecurepassword@localhost:1521/XE

SQL*Plus: Release 21.0.0.0.0 - Production on Mon May 3 14:10:03 2021
Version 21.1.0.0.0

Copyright (c) 1982, 2020, Oracle.  All rights reserved.

Last Successful login time: Mon May 03 2021 13:33:06 -04:00

Connected to:
Oracle Database 18c Express Edition Release 18.0.0.0.0 - Production
Version 18.4.0.0.0

SQL> SELECT TO_CHAR (SYSDATE, 'MM-DD-YYYY HH24:MI:SS') "NOW" FROM DUAL;

NOW
-------------------
05-03-2021 18:10:35

SQL> exit
Disconnected from Oracle Database 18c Express Edition Release 18.0.0.0.0 - Production
Version 18.4.0.0.0
```

Congratulations, you now have an Oracle Database 18c XE container running!


### Build and Compile the Application

First, clone the following repository:
```
$ git clone https://github.com/swseighman/GraalVM-Native-Image-Oracle-Database.git
```

```
$ cd GraalVM-Native-Image-Oracle-Database
```

Compile the application:

```
$ javac -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample.java
```
***NOTE**: If you need a different version of the Oracle JDBC drivers, you can download it [here](https://www.oracle.com/database/technologies/maven-central-guide.html).*

Run the application:
```
$ java -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample
Driver Name: Oracle JDBC driver
Driver Version: 21.1.0.0.0
Default Row Prefetch Value is: 20
Database Username is: SYSTEM

'SELECT * FROM DUAL' returned: X
```
Let's perform a query:
```
$ java -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample "SELECT TO_CHAR (SYSDATE, 'MM-DD-YYYY HH24:MI:SS') \"NOW\" FROM DUAL"
```
For a simple performance test, we'll time how long it takes to connect to the database and perform the same query:
```
$ time java -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample "SELECT TO_CHAR (SYSDATE, 'MM-DD-YYYY HH24:MI:SS') \"NOW\" FROM DUAL"
Driver Name: Oracle JDBC driver
Driver Version: 21.1.0.0.0
Default Row Prefetch Value is: 20
Database Username is: SYSTEM

'SELECT TO_CHAR (SYSDATE, 'MM-DD-YYYY HH24:MI:SS') "NOW" FROM DUAL' returned: 05-03-2021 17:19:16

real    0m0.947s
user    0m1.324s
sys     0m0.118s

```
As you can see, it takes **947ms** to complete the query *(your actual time may vary)*.

Now let's create a native image executable of the DataSourceSample application:
```
$ native-image -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample
[datasourcesample:46780]    classlist:   1,119.72 ms,  0.96 GB
[datasourcesample:46780]        (cap):     698.05 ms,  0.96 GB
WARNING: Method java.sql.SQLXML.<init>() not found.
[datasourcesample:46780]        setup:   2,703.47 ms,  0.96 GB
[datasourcesample:46780]     (clinit):     650.37 ms,  3.22 GB
[datasourcesample:46780]   (typeflow):  11,076.31 ms,  3.22 GB
[datasourcesample:46780]    (objects):  21,435.26 ms,  3.22 GB
[datasourcesample:46780]   (features):   1,643.29 ms,  3.22 GB
[datasourcesample:46780]     analysis:  35,755.73 ms,  3.22 GB
[datasourcesample:46780]     universe:   1,143.49 ms,  3.22 GB
[datasourcesample:46780]      (parse):   4,728.68 ms,  5.34 GB
[datasourcesample:46780]     (inline):   2,556.02 ms,  5.54 GB
[datasourcesample:46780]    (compile):  39,862.82 ms,  6.43 GB
[datasourcesample:46780]      compile:  50,517.38 ms,  6.43 GB
[datasourcesample:46780]        image:   3,942.94 ms,  6.45 GB
[datasourcesample:46780]        write:     593.37 ms,  6.45 GB
[datasourcesample:46780]      [total]:  95,996.08 ms,  6.45 GB
```

```
$ ./datasourcesample -cp .:/ojdbc11-21.1.0.0.jar DataSourceSample
Driver Name: Oracle JDBC driver
Driver Version: 21.1.0.0.0
Default Row Prefetch Value is: 20
Database Username is: SYSTEM

'SELECT * FROM DUAL' returned: X
```
And we'll run the same performance test using the native image executable:
```
 $ time ./datasourcesample "SELECT TO_CHAR (SYSDATE, 'MM-DD-YYYY HH24:MI:SS') \"NOW\" FROM DUAL"
Driver Name: Oracle JDBC driver
Driver Version: 21.1.0.0.0
Default Row Prefetch Value is: 20
Database Username is: SYSTEM

'SELECT TO_CHAR (SYSDATE, 'MM-DD-YYY HH24:MI:SS') "NOW" FROM DUAL' returned: 05-03-021 15:48:04

real    0m0.040s
user    0m0.016s
sys     0m0.000s

```
Impressive, it takes only **40ms** using the native image executable!

### Using Oracle Database 18c XE with Kubernetes

#### Kubernetes Configuration and Setup

First, we'll need to install `minikube`, follow the instructions here: 
https://minikube.sigs.k8s.io/docs/start/

Next, install `kubectl`, see the instructions here:
https://kubernetes.io/docs/tasks/tools/install-kubectl/


```
$ minikube version
minikube version: v1.19.0
commit: 15cede53bdc5fe242228853e737333b09d4336b5
```

```
$ minikube config set memory 8192
$ minikube config set cpus 4
$ minikube config set driver docker
$ minikube config set kubernetes-version v1.20.2
```
 
```
 $ minikube config view
- driver: docker
- kubernetes-version: v1.20.2
- memory: 8192
- cpus: 4
```

```
$ minikube addons enable metrics-server
▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.4.2
🌟  The 'metrics-server' addon is enabled
```

```
$ minikube start
😄  minikube v1.19.0
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.20.2 on Docker 20.10.5 ...
🔎  Verifying Kubernetes components...
    ▪ Using image kubernetesui/dashboard:v2.1.0
    ▪ Using image kubernetesui/metrics-scraper:v1.0.4
    ▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.4.2
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass, metrics-server, dashboard
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

#### Build an Oracle Database XE Container in Kubernetes
```
$ git clone https://github.com/oracle/docker-images.git
$ cd docker-images/OracleDatabase/SingleInstance/dockerfiles
$ eval $(minikube docker-env)
$ ./buildDockerImage.sh -v 18.4.0 -x
```

```
$ minikube stop
```