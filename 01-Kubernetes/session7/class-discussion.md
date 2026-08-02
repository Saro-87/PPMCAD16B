Q.) How to make sure that the data inside your container is not lost even when the container is deleted?

Ans: Docker volume

Map some data from the container to the local system
Volume mounts and bind mounts

Q.) How to make sure that the data inside your pod in kubernetes is not lost when the pod is deleted?


Containers are ephemeral in nature and they do not natively persist data outside of their lifecycle..

Then my option is to use Kubernetes volumes...

PV & PVC supported by SC

PV: Persistent Volumes - these are actual volume in k8s, just like docker volume, this can be a simple storage on the same worker node or this can be a remote volume on AWS like EBS (Elastic Block Storage) or EFS (Elastic File Storage)..

PVC: Persistent Volume Claims - these are the request object where in a pod / deployment request certain storage from K8S cluster 

SC: Storage Class - this is a object that supports dynamic creation of volumes in K8S


Dynamic PV:

Deployment asks for 10 GB of storage via PVC -> PVC Claims for this 10 GB of storage to SC -> SC (based which type it is) will talk to AWS EBS and get one new volume of size 10 GB created in AWS -> PV which directly points to AWS EBS volume gets created in K8S -> PVC <-> PV 


Static PV:

Pre-Req: You as an admin will login to AWS, go to EC2, EBS section..
Create one EBS volume by yourself of 10 GB in size

you will create a PV and link that PV to the EBS volume created by yourself

Deployment asks for 10 GB of storage via PVC -> PVC Claims for this 10 GB of storage directly to the PVs available -> and as soon as it finds the right PV available, it will link itself with that -> PVC <-> PV 

----

1 important, out of context question?
Can you attach one EBS volume with multiple EC2 servers? No, for this req..

Multi-attach feature of EBS with details...


---

If I have a PV in my cluster of size 100 GB statically created...

I went to AWS EBS volumes and I created a volume of size 100 GB and I created a PV and linked this 100 GB to that PV...

Now one of the PVC, with a claim of  10 GB is created in the cluster...

PVC will start scanning for available PVs in the cluster... it will find one with 100 GB storage... it will ignore that the total storage that the PV has... and simple binds itself to that PV..

100 GB PV <-- bound to --> 10 GB PVC

and in K8S, there is only 1-1 binding between PV and PVC

---

Pods, Deployments, DaemonSets, StatefulSets...

Stateful Application vs Stateless application

stateless apps: those apps, which do not store any state on their filesystem, but every such things are stored in the database

stateful apps: they store their state on the filesystem where they run, e.g. database, kafka, rabbitmq, jenkins



