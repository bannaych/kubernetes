
In this blog I will show you how to use the MySQL Shell to manage and create a clustered database on an InnoDB MySQL cluster which has been provisioned using the Oracle MySQL Kubernetes Operator.

This blog will not go into the details on how to setup Kubernetes or Operator, my Colleague Ron Ekins has written an excellent blog on the installation process. 

https://ronekins.com/2021/08/31/getting-started-with-the-oracle-mysql-kubernetes-operator-and-portworx/

# Environment
* Kubernetes 1.21
* Portworx 2.8
* MySQL operator installed https://github.com/mysql/mysql-operator
* MySQL Shell

Lets check to make sure our operator is up and running

![Screen Shot 2021-10-13 at 12 39 57 pm](https://user-images.githubusercontent.com/13579776/137116994-59ff5307-d4f2-43c7-b2a7-d62f7b17085c.png)

Lets check to make sure our InnoDB Cluster  is up and running

![Screen Shot 2021-10-13 at 12 40 42 pm](https://user-images.githubusercontent.com/13579776/137117161-fb651a5c-3a7d-4254-a65f-9af55df33bf5.png)

Lets check to make sure our InnoDB Cluster  PODS are up and running

![Screen Shot 2021-10-13 at 11 50 59 am](https://user-images.githubusercontent.com/13579776/137117286-44fc4925-1ca1-4aea-b54e-c007d91f54b3.png)

# Install MySQL Shell
Installation of the MySQL Shell is very straight forward

* yum install mysql-shell

![Screen Shot 2021-10-13 at 12 41 28 pm](https://user-images.githubusercontent.com/13579776/137117466-33fca0bc-8d0c-4b97-b97a-64a6736d69d7.png)

Note: because I don't have a load balancer on my kubernetes environment, I will be using kubernetes port forwarding to connect to the MySQL PODS from the K8 Master node we will connect to port 6446

![Screen Shot 2021-10-13 at 12 45 17 pm](https://user-images.githubusercontent.com/13579776/137117575-affab605-792a-4b1a-9b77-c0d43858deca.png)

Lets get the IP addresses of the cluster PODS we want to connect to.
root@px-centk8s01 ~]# kubectl get pods -o wide
10.244.1.44,45,46

![Screen Shot 2021-10-13 at 12 53 05 pm](https://user-images.githubusercontent.com/13579776/137117760-0979a90f-e8ff-459c-8559-42295995ae95.png)

Connecting to the localhost using mysqlsh "mysqlsh chrisb@localhost:6446 --sql"
I have already created a user called "chris" as an admin user on the cluster nodes
In this example I am connecting to the mysql cluster pod running on port 6446 as the user chris, notice the SQL prompt - mysqlsh allows you to connect in SQL mode or JS ( java script ) mode

![Screen Shot 2021-10-13 at 12 47 30 pm](https://user-images.githubusercontent.com/13579776/137117839-f10f2782-2c9e-4389-aea5-f8dd6d351d3d.png)

Lets change to JS mode ( SQL> \js ) and check the cluster status, we can see the three cluster nodes  with mycluster-0 as the primary node

![Screen Shot 2021-10-13 at 2 00 42 pm](https://user-images.githubusercontent.com/13579776/137117911-93da839d-f493-400e-9e79-208d87de0baf.png)

Now let's change back to SQL mode in mysqlsh.

![Screen Shot 2021-10-13 at 4 13 18 pm](https://user-images.githubusercontent.com/13579776/137118016-da83f70c-08ba-40c9-a9b3-e6a2ea6ee448.png)

From the SQL prompt we can run any SQL command which you can run from a standard mysql database connection, let's have a look at what databases are configured.

![Screen Shot 2021-10-13 at 3 54 18 pm](https://user-images.githubusercontent.com/13579776/137118065-077653b0-0d72-4aac-8753-3db105166f82.png)

Let me check which which cluster node we are on, then will create a new database called blog and confirm all the cluster nodes can see and read the database. The new blog database is now visible. We are on the mycluster-0

![Screen Shot 2021-10-13 at 4 19 14 pm](https://user-images.githubusercontent.com/13579776/137118124-8f0c644d-3cbe-4897-ad69-d08ad619a623.png)

![Screen Shot 2021-10-13 at 4 20 21 pm](https://user-images.githubusercontent.com/13579776/137118177-d9dad467-f289-4d49-bd80-c6fccb0c3eee.png)

Lets now connect to cluster node mycluster-1 "10.244.1.45" and confirm they all see the blog database, I will switch back to JS mode and connect to each cluster node

![Screen Shot 2021-10-13 at 4 22 04 pm](https://user-images.githubusercontent.com/13579776/137118248-40be2100-9fc0-4a9c-b361-e95ebcb92d6e.png)

We will do the same thing with cluster node mycluster-2

![Screen Shot 2021-10-13 at 4 23 12 pm](https://user-images.githubusercontent.com/13579776/137118314-ddf12b90-2fd4-440f-94f2-1d4de658b29f.png)

Now because cluster nodes 1 and 2 are read-only nodes we cannot create a database or add data to a table as we can see from attempting to create a new database called blog2 on mycluster-2

![Screen Shot 2021-10-13 at 4 24 04 pm](https://user-images.githubusercontent.com/13579776/137118413-a938bab6-f181-47cb-be9b-b664958e9bac.png)






