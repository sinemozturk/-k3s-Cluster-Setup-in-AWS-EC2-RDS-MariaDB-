# k3s Cluster Setup in AWS EC2 + RDS (MariaDB)

## High-Availability K3s Server with an External DB

- K3s Architecture with a High-availability Server

![ha](k3s-HA.JPG)

- In our project we used 2 instances for master and worker node and 1 RDS (MariaDB) as a database

![ec2-rds](ec2-rds.jpg)

### EC2 Instances Requirements

- EC2 installation; Select `Ubuntu 20.04` SSD Volume Type and Instance type `t3a.nano` select the key pair. And make sure instance has access to SSH connection (Port 22).
- We need 2 instances. One of the instances will be `master`
- Other instance will be `agent` or known as `worker node`

- Requirements for project-X

- Virtual Machine Requirements:

|EC2 AMI Type|Architecture|Instance Type| VCPUS | RAM|
|---|---|---|---|---|
|Ubuntu| 64-bit (x86)| t3a.nano or t4g.nano| 2 core | 512 Mb|

- Inbound Rules for EC2

|Port range|Protocol|Source|
|---|---|---|
|22|TCP|0.0.0.0/0|
|80|TCP|0.0.0.0/0|
|443|TCP|0.0.0.0/0|
|6443|TCP|0.0.0.0/0|
|30000-32767|TCP|0.0.0.0/0|
|30000-32767|UDP|0.0.0.0/0|

- Outbount Rules

|Port range|Protocol|Destination|
|---|---|---|
|All|All|0.0.0.0/0|

### RDS-MariaDB Requirements

- We will use RDS `MariaDB`
- we can use `free tier`
- Note: It should be created in `default VPC` same with EC2 instances
- Creation Method: standard create
- Engine option : MariaDB
- Version : MariaDB 10.6.8
- Templates : Free Tier
- DB instance identifier: k3s-cluster0db (named with related project)
- master username: k3s `<expamle>`
- master password: k3s12345 `<example>`
- Instance configuration : leave it as is
- Storage: unenabled auto-scaling
- Additional configuration: unenable automated backup, unenable encryption , unenable enhanced monitoring, unenable auto minor upgrade
- create database

- Edit security group of the RDS

|IP version|Type|Protocol|Port range|Source|
|---|---|---|---|---|
|Ipv4|MySQL/Aurora|TCP|3306|(`master's private IP address/range`)172.31.86.155/32|
|-| All traffic|All|All|default sec group|

## Part 1 - Launch Amazon Ubuntu 20.04 EC2 Instance and Connect with SSH

- Launch an EC2 instance using the Amazon Ubuntu 20.04 AMI with security group allowing SSH connections.

- Connect to your instance with SSH.

```bash
ssh -i .ssh/call-training.pem ec2-user@ec2-3-133-106-98.us-east-2.compute.amazonaws.com
```

- Hostname change of the nodes, so we can discern the roles of each nodes. For example, you can name the nodes (instances) like `k3s-master, k3s-worker`

```bash
hostnamectl set-hostname <node-name-master-or-worker>
hostnamectl set-hostname k3s-worker `example of worker node name `
```

```bash
bash
```

- Always is better to update the machine before we use. Go to root account in both ec2 instances and apply following codes to both seperately

- Switching ubuntu account to root account in both instances

```bash
sudo su - 
```

- Update the instances with following bash commands apply seperately

```bash
apt-get install -y
apt-get update -y
```

- We need to bootstrap the `master node` and then we have to attach the `worker node` to the master node. First thing is we need to connect to the `RDS` and create a database to store our cluster information in `master node`. To do this we need to install MysQL client.

- Install MysQL client on `Master Node`

```bash
apt install mysql-client -y
```

- We need to reach out to RDS from EC2 instance so doing this we need `endpoint of RDS` Go to the AWS Console -> RDS -> MariaDB database on the `connectivity & security` section and copy `End point` of the database. example : k3s.cspat0oooh9v.us-east-1.rds.amazonaws.com

- Connect to the RDS on Master Node;

- example of the following bash -> mysql -h k3s.cspat0oooh9v.us-east-1.rds.amazonaws.com -u k3s -p and enter the password

```bash
mysql -h <the endpoint>-u <masteruser name of the rds> -p 
```

- It will ask you the password when you created at the beginning of the RDS installation enter it.

```bash

...
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
Server version: 5.5.5-10.6.8-MariaDB managed by 
https://aws.amazon.com/rds/

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

```

- Now we connected the RDS let's see what are the available databases in there.

```bash
show databases;
```

- all default databases;

```bash
...
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |  
| innodb             |  
| mysql              |  
| performance_schema |  
| sys                |  
+--------------------+  
5 rows in set (0.00 sec)
```

- Create a database with the name k3s to store our cluster information

```bash
CREATE DATABASE k3s;
```

```bash
...
mysql> CREATE DATABASE k3s;
Query OK, 1 row affected (0.00 sec)
```

- flush privileges

```bash
FLUSH PRIVILEGES;
```

- Exit from mysql

```bash
exit
```

- We need to export endpoint of database on `master node`

```bash
export K3S_DATASTORE_ENDPOINT='mysql://username:password@endpoint(hostname:3306)/database-name'
```

- example: export K3S_DATASTORE_ENDPOINT='mysql://k3s:12345678@tcp(k3s.cspt0ooh9v.us-east-1.rds.amazonaws.com:3306)/k3s'

- Now we need to install k3s on `master node` to RDS

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig=/home/ubuntu/.kube/config --write-kubeconfig-mode=644" sh -
```

- Check the status of k3s

```bash
systemctl status k3s
```

- output of the status of k3s to quit press `q`

```bash
...
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor pres>     
     Active: active (running) since Wed 2022-08-10 17:43:43 UTC; 2h 26min >      
      Docs: https://k3s.io
    Process: 18875 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enable>    
    Process: 18877 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, >   
     Process: 18878 ExecStartPre=/sbin/modprobe overlay (code=exited, statu>  
      Main PID: 18879 (k3s-server)
      Tasks: 98
     Memory: 537.6M
        CPU: 9min 16.253s
     CGroup: /system.slice/k3s.service
             ├─ 1252 /var/lib/rancher/k3s/data/1d787a9b6122e3e3b24afe621da>            
             ├─ 1462 /var/lib/rancher/k3s/data/1d787a9b6122e3e3b24afe621da>             
             ├─ 1687 /var/lib/rancher/k3s/data/1d787a9b6122e3e3b24afe621da>             
             ├─ 1854 /var/lib/rancher/k3s/data/1d787a9b6122e3e3b24afe621da>             
             ─ 2024 /var/lib/rancher/k3s/data/1d787a9b6122e3e3b24afe621da>             
             ├─18879 "/usr/local/bin/k3s server" "" "" "" "" "" "" "" "" ">             
             └─18911 containerd -c /var/lib/rancher/k3s/agent/etc/containe>

Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.413930   18879 shared_in>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.416203   18879 controlle>
ug 10 17:44:15 master k3s[18879]: I0810 17:44:15.426609   18879 controlle>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.427244   18879 shared_in>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.427401   18879 disruptio>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.466195   18879 shared_in>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.492454   18879 shared_in>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.847915   18879 shared_in>
Aug 10 17:44:15 master k3s[18879]: I0810 17:44:15.853220   18879 shared_in>
lines 1-29
```

- Check the nodes of k3s

```bash
kubectl get node
```

- Output of nodes

```bash
...
NAME               STATUS   ROLES                  AGE    VERSION
master             Ready    control-plane,master   18h    v1.24.3+k3s1 
```

- Exit the root

```bash
exit
```

- List the files that we created

```bash
ll
```

- Output -> we can see k3s is successfully installed and created .kube file

```bash
...
total 32
drwxr-xr-x 5 ubuntu ubuntu 4096 Aug  9 02:41 ./
drwxr-xr-x 3 root   root   4096 Aug  9 01:38 ../
-rw-r--r-- 1 ubuntu ubuntu  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu 3771 Feb 25  2020 .bashrc
drwx------ 2 ubuntu ubuntu 4096 Aug  9 01:56 .cache/
drwxr-xr-x 2 root   root   4096 Aug  9 02:41 .kube/
-rw-r--r-- 1 ubuntu ubuntu  807 Feb 25  2020 .profile
drwx------ 2 ubuntu ubuntu 4096 Aug  9 01:38 .ssh/
-rw-r--r-- 1 ubuntu ubuntu    0 Aug  9 01:58 .sudo_as_admin_successful
```

- Check the node

```bash
kubectl get node
```

- Output

```bash
...
NAME     STATUS   ROLES                  AGE     VERSION
master   Ready    control-plane,master   4m48s   v1.24.3+k3s1
```

- Swith to the root user again

```bash
sudo su -
```

- Check the namespaces that are created by default

```bash
kubectl get ns
```

- Output

```bash
...
NAME              STATUS   AGE  
default           Active   7m24s                                                      
kube-system       Active   7m24s                                                          
kube-public       Active   7m24s
kube-node-lease   Active   7m24s
```

- Check the namespaces in the kube-system

```bash
kubectl get pods -n kube-system
```

- Output

```bash
...
NAME                                      READY   STATUS      RESTARTS   AGE                 Reference{Kind:\"HelmC>
local-path-provisioner-7b7dc8d6f5-6v4mj   1/1     Running     0          10m                 Reference{Kind:\"HelmC>
helm-install-traefik-2hlht                0/1     Completed   4          10m                 048431408s due to clie>
helm-install-traefik-crd-zqt8c            0/1     Completed   1          10m                 to 127.0.0.1:6443"     
coredns-b96499967-fkg4c                   1/1     Running     0          10m                 roxy" url="wss://172.3>
traefik-7cd4fcff68-fzzdr                  1/1     Running     0          7m52s               ="context canceled" ur>
svclb-traefik-12df9903-99kd4              2/2     Running     0          7m52s               dialer server [400]: w>
metrics-server-668d979685-h2b6n           1/1     Running     0          10m
```

- To understand files and folder structure which are created by the k3s. It is the home directory for this particular K3s.

```bash
cd /var/lib/rancher/
```

- Check the files inside the home directory of k3s

```bash
ls
```

- Output

```bash
...
k3s  
```

- Go to inside of `k3s`

```bash
cd k3s/
```

- Check the files inside the k3s

```bash
ls
```

- Output

```bash
...
agent  data  server
```

- Go to `server` file

```bash
cd server/
```

- Check the files inside the server file

```bash
ls
```

- Output

```bash
... 
cred  etc  kine.sock  manifests  node-token  static  tls  token
```

- We need TOKEN to create a cluster and connect the `master` and `worker` node. Run following command. Save the TOKEN we will use.

```bash
cat token
```

- Output of token `example`

```bash
...
K107fcfbe92a673665be43ff81d87c6619dd621dc4a27d2ee4e08106351a339f::server:d1210b27ea0eedfd225b7a642330c558
```

- `master node` and watch the nodes see how many nodes are available on the cluster. We will connect the worker node and we will be able to see while attaching TOKEN on worker node.Note: you can quit watching with `q`

```bash
watch kubectl get nodes
```

- We need to export some variables on `worker node` so go worker node (worker ec2) we need to provide the master node's `private IP` address. Note; API server is listening of 6443 port

```bash
export K3S_URL=https://private-Ip-of-master-ec2:6443
```

- example of usage above code `export K3S_URL=https://172.21.76.185:6443`

- We need to export TOKEN variable on `worker node` or `nodes`

```bash
export K3S_TOKEN=< paste here the token that we take from master node>
```

- Execute the following command on  `worker`

```bash
curl -sfL https://get.k3s.io | sh 
```

- Also you can use the last 3 bash command in just one command see the following bash command;

```bash
curl -sfL <https://get.k3s.io> | K3S_URL=https://private-Ip-of-master-ec2:6443 K3S_TOKEN="paste here the token" sh -
```

- check the k3s agent status on `worker node`

```bash
systemctl status k3s-agent 
```

- Output -> to quit the page press `q`

```bash
...
 k3s-agent.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s-agent.se>     
     Active: active (running) since Wed 2022-08-10 17>       
     Docs: https://k3s.io
   Main PID: 511 (k3s-agent)
      Tasks: 39
     Memory: 184.9M
        CPU: 7.901s
     CGroup: /system.slice/k3s-agent.service
             ├─  511 "/usr/local/bin/k3s agent"       
             ├─14697 containerd -c /var/lib/rancher/k>             
             └─15317 /var/lib/rancher/k3s/data/1d787a>

Aug 10 17:12:59 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:12:59 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:12:59 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:12:59 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:12:59 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:17:09 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:17:09 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:17:09 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:17:09 ip-172-31-94-244 k3s[511]: I0810 17:1>
Aug 10 17:17:09 ip-172-31-94-244 k3s[511]: I0810 17:1>
```

- Now we have k3s on worker node as well so check the cluster nodes on `master node`

```bash
kubectl get node
```

- Output : now we can see one master and one worker node are in cluster

```bash
...
NAME               STATUS   ROLES                  AGE     VERSION
ip-172-31-82-108   Ready    <none>                 3h21m   v1.24.3+k3s1
master             Ready    control-plane,master   19h     v1.24.3+k3s1
```

- We need to check RDS database and then we will try to observe whether this particular setup is using the RDS databases or not

- To connect the database

```bash
mysql -h < paste here the endpoint > -u <masteruser of the rds> -p 
```

- Show databases

```bash
show databases;
```

- Output now we can see k3s database which means we had successful install

```bash
...
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |  
| innodb             |  
| k3s                |  
| mysql              |  
| performance_schema |  
+--------------------+  
5 rows in set (0.00 sec)

```

- Check the contents inside of the k3s database

```bash
use k3s;
```

- Output -> it connected to the this particular k3s database

```bash
...
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

- Show the tables of k3s

```bash
show tables;
```

- Output -> in this `kine` table has all the cluster information. Note; if you want to see all the data inside of this table we need to install any web clients such as MySQL or Workbench  and we can connect to the database and see all data, but in this project so far we do not need it.

```bash
...
+---------------+      
| Tables_in_k3s |      
+---------------+      
| kine          |      
+---------------+      
1 row in set (0.00 sec)
```

- Exit the mysql

```bash
exit
```
