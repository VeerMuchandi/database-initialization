##Initializing a Database Pod

This blog explains how to initialize a database pod after it is created on OpenShift. I am using MYSQL database as an example. We can follow similar process with other databases.

Let us say we want to create a MYSQL pod and once the pod is created, we would like to add a table. We will use "Post" hook to achieve this.

**Step 1:** Spin up a MySQL database pod

You can add a MySQL database pod either from OpenShift Console or using command line as shown here.

```
$ oc new-app -e MYSQL_USER='user' \
> MYSQL_PASSWORD='password' \
> MYSQL_DATABASE=mydatabase \
> registry.access.redhat.com/rhscl/mysql-56-rhel7 --name='mysql'
--> Found Docker image 90db79d (3 weeks old) from registry.access.redhat.com for "registry.access.redhat.com/rhscl/mysql-56-rhel7"

    MySQL 5.6 
    --------- 
    MySQL 5.6 SQL database server

    Tags: database, mysql, mysql56, rh-mysql56

    * An image stream will be created as "mysql:latest" that will track this image
    * This image will be deployed in deployment config "mysql"
    * Port 3306/tcp will be load balanced by service "mysql"
      * Other containers can access this service through the hostname "mysql"
    * This image declares volumes and will default to use non-persistent, host-local storage.
      You can add persistent volumes later by running 'volume dc/mysql --add ...'

--> Creating resources ...
    imagestream "mysql" created
    deploymentconfig "mysql" created
    service "mysql" created
--> Success
    Run 'oc status' to view your app.
```

**Step 2:** Edit deployment configuration to add `Post` hook

Edit deploymentconfig created in the previous step by running `oc edit dc mysql`. Or you can do it from OpenShift Console as well. 

```
spec:
...
....
 strategy:
 ....
 .....
    rollingParams:
     post:
       execNewPod:
         command:
         - /bin/sh
         - -c
         - hostname && sleep 10 && /opt/rh/rh-mysql56/root/usr/bin/mysql -h $MYSQL_SERVICE_HOST
           -u $MYSQL_USER -D $MYSQL_DATABASE -p$MYSQL_PASSWORD -P 3306 -e 'CREATE
           TABLE IF NOT EXISTS emails (from_add varchar(40), to_add varchar(40),
           subject varchar(40), body varchar(200), created_at date);' && sleep 60
         containerName: mysql
       failurePolicy: ignore
....
....

```
The above is a partial snippet that shows where to add it. Note that `sleep 10` introduces a delay for the container to come up before connecting to the `mysql` client and execute the `CREATE TABLE` script. The `Post` hook will run as a separate pod that will die after the action is complete. `sleep 60` at the end is an intentional introduction to keep the pod alive for a minute, so that you get a chance to watch the logs. It can be removed, once you get hang of how this hook works.

Note that the indentation is very important. After the results, the end result should show up as follows.

```
$ oc get dc mysql -o yaml
apiVersion: v1
kind: DeploymentConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  creationTimestamp: 2017-03-17T02:58:18Z
  generation: 3
  labels:
    app: mysql
  name: mysql
  namespace: dbinit
  resourceVersion: "3752083"
  selfLink: /oapi/v1/namespaces/dbinit/deploymentconfigs/mysql
  uid: 92f95a70-0abd-11e7-aa23-000d3af7a1bb
spec:
  replicas: 1
  selector:
    app: mysql
    deploymentconfig: mysql
  strategy:
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      post:
        execNewPod:
          command:
          - /bin/sh
          - -c
          - hostname && sleep 10 && /opt/rh/rh-mysql56/root/usr/bin/mysql -h $MYSQL_SERVICE_HOST
            -u $MYSQL_USER -D $MYSQL_DATABASE -p$MYSQL_PASSWORD -P 3306 -e 'CREATE
            TABLE IF NOT EXISTS emails (from_add varchar(40), to_add varchar(40),
            subject varchar(40), body varchar(200), created_at date);' && sleep 60
          containerName: mysql
        failurePolicy: ignore
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      annotations:
        openshift.io/generated-by: OpenShiftNewApp
      creationTimestamp: null
      labels:
        app: mysql
        deploymentconfig: mysql
    spec:
      containers:
      - env:
        - name: MYSQL_DATABASE
          value: mydatabase
        - name: MYSQL_PASSWORD
          value: password
        - name: MYSQL_USER
          value: user
        image: registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:1c767450a7b1ef7151bf54f18d96ae6fef0e71a52ee4a58b9b28e65614fe5962
        imagePullPolicy: Always
        name: mysql
        ports:
        - containerPort: 3306
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /var/lib/mysql/data
          name: mysql-volume-1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: mysql-volume-1
  test: false
  triggers:
  - type: ConfigChange
  - imageChangeParams:
      automatic: true
      containerNames:
      - mysql
      from:
        kind: ImageStreamTag
        name: mysql:latest
        namespace: dbinit
      lastTriggeredImage: registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:1c767450a7b1ef7151bf54f18d96ae6fef0e71a52ee4a58b9b28e65614fe5962
    type: ImageChange
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: 2017-03-17T02:58:25Z
    message: Deployment config has minimum availability.
    status: "True"
    type: Available
  - lastTransitionTime: 2017-03-17T02:58:22Z
    message: Replication controller "mysql-1" has completed progressing
    reason: NewReplicationControllerAvailable
    status: "True"
    type: Progressing
  details:
    causes:
    - imageTrigger:
        from:
          kind: ImageStreamTag
          name: mysql:latest
          namespace: dbinit
      type: ImageChange
    message: image change
  latestVersion: 1
  observedGeneration: 3
  replicas: 1
  updatedReplicas: 1
```

Now deploy the changes by running `oc rollout` command.

```
$ oc rollout latest mysql
deploymentconfig "mysql" rolled out
```

You will notice that a pod for post hook comes up after the mysql pod starts running. It will die after the hook is executed. 

```
$ oc get pods
NAME                READY     STATUS    RESTARTS   AGE
mysql-2-deploy      1/1       Running   0          16s
mysql-2-hook-post   1/1       Running   0          6s
mysql-2-sgr9p       1/1       Running   0          12s
```
You may want to look at the logs of the post hook pod by running `oc logs -f mysql-2-hook-post`.

Once done, you can verify that the table is created as shown below.

```
$ oc rsh mysql-2-sgr9p
sh-4.2$ mysql -u $MYSQL_USER -p$MYSQL_PASSWORD -h $MYSQL_SERVICE_HOST -P 3306 -D $MYSQL_DATABASE
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.34 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show tables;
+----------------------+
| Tables_in_mydatabase |
+----------------------+
| emails               |
+----------------------+
1 row in set (0.00 sec)

mysql> exit 
Bye
sh-4.2$ exit
exit
```

### Simplifying the patching process

Editing yaml files can be tedious. So, I have included a bash script that patching a little easier. This is not a supported solution, but just a tiny workaound script.

You can either clone this repository to get the scripts locally or download/copy the files individually.

```
git clone 
```

**Step 1:** Create a file named `initsql.txt` and add your SQL in there. In this example, I am creating a table named `customer` and adding some data.

```
$ cat initsql.txt 
CREATE TABLE IF NOT EXISTS customer (CUST_ID int(10) unsigned NOT NULL AUTO_INCREMENT, NAME varchar(100) NOT NULL, AGE int(10) unsigned NOT NULL, PRIMARY KEY (CUST_ID) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;insert into customer values (null, "Joe", 88);insert into customer values (null, "Jack", 54);insert into customer values (null, "Ann", 32);
```

**Step 2:** Create a MySQL database just like above. You can use CLI as shown below or do it from OpenShift Console.

```
$ oc new-app -e MYSQL_USER='user' \
> MYSQL_PASSWORD='password' \
> MYSQL_DATABASE=mydatabase \
> registry.access.redhat.com/rhscl/mysql-56-rhel7 --name='mysql'
--> Found Docker image 90db79d (3 weeks old) from registry.access.redhat.com for "registry.access.redhat.com/rhscl/mysql-56-rhel7"

    MySQL 5.6 
    --------- 
    MySQL 5.6 SQL database server

    Tags: database, mysql, mysql56, rh-mysql56

    * An image stream will be created as "mysql:latest" that will track this image
    * This image will be deployed in deployment config "mysql"
    * Port 3306/tcp will be load balanced by service "mysql"
      * Other containers can access this service through the hostname "mysql"
    * This image declares volumes and will default to use non-persistent, host-local storage.
      You can add persistent volumes later by running 'volume dc/mysql --add ...'

--> Creating resources ...
    imagestream "mysql" created
    deploymentconfig "mysql" created
    service "mysql" created
--> Success
    Run 'oc status' to view your app.
```

**Step 3:** Apply Patch

This time we will use a script that parses the `initsql.txt` file that has your database init script and applies the patch using the `oc patch` command.

Here is what the script looks like:

```
$ cat run.sh
export escapedQuery=$(sed -e 's:":\\\\":g' initsql.txt)
eval $(sed -e "s/\$MYQUERY/$escapedQuery/" patch.txt)

$ cat patch.txt
oc patch dc/mysql --patch '{"spec":{"strategy":{"rollingParams":{"post":{"failurePolicy": "ignore","execNewPod":{"containerName":"mysql","command":["/bin/sh","-c","hostname&&sleep 10&&echo $QUERY | /opt/rh/rh-mysql56/root/usr/bin/mysql -h $MYSQL_SERVICE_HOST -u $MYSQL_USER -D $MYSQL_DATABASE -p$MYSQL_PASSWORD -P 3306"], "env": [{"name": "QUERY", "value":"$MYQUERY"}]}}}}}}'
```

Apply patch by running the script as shown below:

```
$ source run.sh
"mysql" patched
```

**Step 4:** Verify the patch and deploy the changes

Check the deployment configuration on how `oc patch` added the Post hook. 

```
$ oc get dc mysql -o yaml
apiVersion: v1
kind: DeploymentConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  creationTimestamp: 2017-03-17T02:35:43Z
  generation: 3
  labels:
    app: mysql
  name: mysql
  namespace: dbinit
  resourceVersion: "3749204"
  selfLink: /oapi/v1/namespaces/dbinit/deploymentconfigs/mysql
  uid: 6b35faef-0aba-11e7-aa23-000d3af7a1bb
spec:
  replicas: 1
  selector:
    app: mysql
    deploymentconfig: mysql
  strategy:
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      post:
        execNewPod:
          command:
          - /bin/sh
          - -c
          - hostname&&sleep 10&&echo $QUERY | /opt/rh/rh-mysql56/root/usr/bin/mysql
            -h $MYSQL_SERVICE_HOST -u $MYSQL_USER -D $MYSQL_DATABASE -p$MYSQL_PASSWORD
            -P 3306
          containerName: mysql
          env:
          - name: QUERY
            value: CREATE TABLE IF NOT EXISTS customer (CUST_ID int(10) unsigned NOT
              NULL AUTO_INCREMENT, NAME varchar(100) NOT NULL, AGE int(10) unsigned
              NOT NULL, PRIMARY KEY (CUST_ID) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT
              CHARSET=utf8;insert into customer values (null, "Joe", 88);insert into
              customer values (null, "Jack", 54);insert into customer values (null,
              "Ann", 32);
        failurePolicy: ignore
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      annotations:
        openshift.io/generated-by: OpenShiftNewApp
      creationTimestamp: null
      labels:
        app: mysql
        deploymentconfig: mysql
    spec:
      containers:
      - env:
        - name: MYSQL_DATABASE
          value: mydatabase
        - name: MYSQL_PASSWORD
          value: password
        - name: MYSQL_USER
          value: user
        image: registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:1c767450a7b1ef7151bf54f18d96ae6fef0e71a52ee4a58b9b28e65614fe5962
        imagePullPolicy: Always
        name: mysql
        ports:
        - containerPort: 3306
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /var/lib/mysql/data
          name: mysql-volume-1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: mysql-volume-1
  test: false
  triggers:
  - type: ConfigChange
  - imageChangeParams:
      automatic: true
      containerNames:
      - mysql
      from:
        kind: ImageStreamTag
        name: mysql:latest
        namespace: dbinit
      lastTriggeredImage: registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:1c767450a7b1ef7151bf54f18d96ae6fef0e71a52ee4a58b9b28e65614fe5962
    type: ImageChange
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: 2017-03-17T02:35:50Z
    message: Deployment config has minimum availability.
    status: "True"
    type: Available
  - lastTransitionTime: 2017-03-17T02:35:51Z
    message: Replication controller "mysql-1" has completed progressing
    reason: NewReplicationControllerAvailable
    status: "True"
    type: Progressing
  details:
    causes:
    - imageTrigger:
        from:
          kind: ImageStreamTag
          name: mysql:latest
          namespace: dbinit
      type: ImageChange
    message: image change
  latestVersion: 1
  observedGeneration: 3
  replicas: 1
  updatedReplicas: 1
```

Deploy the changes applied to the deploymentconfig by running `oc rollout`.

```
$ oc rollout latest mysql
deploymentconfig "mysql" rolled out
```

Observe that the new MySQL pod comes up and then the post hook pod comes up to run the database initialization.

```
$ oc get pods -w
NAME             READY     STATUS              RESTARTS   AGE
mysql-1-gmtqr    1/1       Running             0          6m
mysql-2-2k9q4    0/1       ContainerCreating   0          1s
mysql-2-deploy   1/1       Running             0          5s
NAME            READY     STATUS    RESTARTS   AGE
mysql-2-2k9q4   1/1       Running   0          3s
mysql-1-gmtqr   1/1       Terminating   0         6m
mysql-2-hook-post   0/1       Pending   0         0s
mysql-2-hook-post   0/1       Pending   0         0s
mysql-2-hook-post   0/1       ContainerCreating   0         0s
mysql-1-gmtqr   0/1       Terminating   0         6m
mysql-1-gmtqr   0/1       Terminating   0         6m
mysql-1-gmtqr   0/1       Terminating   0         6m
mysql-2-hook-post   1/1       Running   0         4s
mysql-2-hook-post   0/1       Completed   0         14s
mysql-2-deploy   0/1       Completed   0         24s
mysql-2-deploy   0/1       Terminating   0         24s
mysql-2-deploy   0/1       Terminating   0         24s
mysql-2-hook-post   0/1       Terminating   0         15s
mysql-2-hook-post   0/1       Terminating   0         15s
^C
```

Now verify that the table is created and data initialized in the pod as shown below:

```
$ oc rsh mysql-2-2k9q4
sh-4.2$ mysql -u $MYSQL_USER -p$MYSQL_PASSWORD -h $MYSQL_SERVICE_HOST -P 3306 -D $MYSQL_DATABASE
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.6.34 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show tables;
+----------------------+
| Tables_in_mydatabase |
+----------------------+
| customer             |
+----------------------+
1 row in set (0.00 sec)

mysql> select * from customer;
+---------+------+-----+
| CUST_ID | NAME | AGE |
+---------+------+-----+
|       2 | Joe  |  88 |
|       3 | Jack |  54 |
|       4 | Ann  |  32 |
+---------+------+-----+
3 rows in set (0.00 sec)

mysql> exit
Bye
sh-4.2$ exit
exit
```

**Summary:** You have learnt how to use post hook to initialize a MySQL pod. Same approach can be followed for other databases as well.

