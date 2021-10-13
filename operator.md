
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
