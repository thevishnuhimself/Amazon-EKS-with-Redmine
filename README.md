

# What is Elastic Kubernetes Service by Amazon?
_**Elastic Kubernetes Service- which is an Amazon offering that helps in running the Kubernetes on AWS without requiring the user to maintain their own Kubernetes control plane. It is a fully managed service by Amazon.**_


# Let's begin....
**Hola People,**

   In this project, I have launched an issue tracking tool REDMINE Architecture on Amazon EKS. Redmine is a free and open-source, web-based project management, and issue tracking tool. It allows users to manage multiple projects and associated subprojects. It features per project wikis and forums, time tracking, and flexible role-based access control. It includes a calendar and Gantt charts to aid visual representation of projects and their deadlines. Redmine integrates with various version control systems and includes a repository browser and diff viewer. famous open-source blogging platform used widely by independent journalists & writers across the globe.
              
              
_**Let's start with the -**_

**DEPLOYMENT OF REDMINE AS THE FRONTEND WITH MYSQL AS THE DATABASE ON AWS EKS:**

There are certain prerequisites for this deployment- 

=> **Pre-installed KUBECTL.**

=> **Pre-installed AWS CLI.**

=> **Pre-installed EKSCTL.**




_**Step-1: Go to AWS and create a IAM user with all power access like ADMIN.**_



![0](https://user-images.githubusercontent.com/66829650/90311240-7d23e380-df16-11ea-868b-984f1e77ecbe.png)


These credentials help you to log in as a user (IAM) and launch the architecture of REDMINE.

![Untasitled](https://user-images.githubusercontent.com/66829650/90312136-9ed59880-df1f-11ea-81bd-ebf95acadda3.png)

_**STEP 2: Using Eksctl command create a EKS CLUSTER.**_

Amazon will manage the k8s cluster master internally but for the slaves, we need to mention the configurations like how many instances we want as a slave and what should be the hardware configuration of that system ie Instance type.

For Serverless Cluster we can go for the Fargate Profiles where we don't even require to tell the configuration about the salve node.

            apiVersion: eksctl.io/v1alpha5
            kind: ClusterConfig
            metadata:
              name: vpcluster
              region: ap-south-1
            nodeGroups:
            - name: ng1
              desiredCapacity: 3
              instanceType: t2.micro
              ssh:
            publicKeyName: newkey

 

         eksctl create cluster -f <file/name>
         
         
We can use Aws eks command for this but for some more customization about the different instance-type in one node group and different instances in the different node group is not possible.

![0 (1)](https://user-images.githubusercontent.com/66829650/90312150-d17f9100-df1f-11ea-915f-eefd3bdeea20.png)

Then after cluster created , it might take almost 30-35 minutes as this code first contact to cloud formation, and cloud formation code is created for launching these resources from code and then the cluster is clustered and then the node groups are created.

![part-4](https://user-images.githubusercontent.com/66829650/90312157-e1977080-df1f-11ea-807a-0d688161f63a.png)


Then we need to update the kubeconfig file for setting the current cluster details like the server IP, certificate authority, and the authentication of the kubectl so that client can launch their apps in that cluster using kubectl command.

aws eks update-kubeconfig --name <cluster name>
Then its good practice to create a namespace and create your resources in that namespace.

kubectl create ns <namespace name>
Then to set that namespace as the current context ( as by default namespace is the default):

kubectl config set-context --current --namespace=<namespace name>
Then we need to install amazon-efs-utils in the slave instances so that later on we can mount the EFS to those instances.

For this purpose we can either use terraform or the best automation tool for the configuration of the environment is ANSIBLE.

_**STEP 3: CREATE AN EFS PROVISIONER :**_

The efs-provisioner allows you to mount EFS storage as PersistentVolumes in kubernetes. It consists of a container that has access to an AWS EFS resource. The container reads a configmap which contains the EFS filesystem ID, the AWS region, and the name you want to use for your efs-provisioner.

But before creating the Efs- provisioner, create the EFS from where the pods will get the real storage.

![0 (2)](https://user-images.githubusercontent.com/66829650/90312182-fb38b800-df1f-11ea-857e-0169d4558b94.png)


               kind: Deployment
               apiVersion: apps/v1
               metadata:
                 name: efs-provisioner
               spec:
                 selector:
                   matchLabels:
                     app: efs-provisioner
                 replicas: 1
                 strategy:
                   type: Recreate
                 template:
                   metadata:
                     labels:
                       app: efs-provisioner
                   spec:
                     containers:
                       - name: efs-provisioner
                         image: quay.io/external_storage/efs-provisioner:v0.1.0
                         env:
                           - name: FILE_SYSTEM_ID
                             value: fs-ad169c7c
                           - name: AWS_REGION
                             value: ap-south-1
                           - name: PROVISIONER_NAME
                             value: eks-cluster/aws-efs
                         volumeMounts:
                           - name: pv-volume
                             mountPath: /persistentvolumes
                     volumes:
                       - name: pv-volume
                         nfs:
                           server: fs-ad169c7c.efs.ap-south-1.amazonaws.com
                           path: /


We also need to go inside our nodes & run this command once - yum install amazon-efs-utils. Go inside the nodes via ssh & runn this command. This will install the necessary softwares on those nodes so that they can be compatible with the EFS Storage.

_**STEP 4: Next, I have created a YML code to modify some permisssions using ROLE BASED ACCESS CONTROL (RBAC). The code is as follows -**_

                     apiVersion: rbac.authorization.k8s.io/v1beta1
                     kind: ClusterRoleBinding
                     metadata:
                       name: nfs-provisioner-role-binding
                     subjects:
                       - kind: ServiceAccount
                         name: default
                         namespace: default
                     roleRef:
                       kind: ClusterRole
                       name: cluster-admin
                       apiGroup: rbac.authorization.k8s.io
 _**STEP - 5: Next, I have created a secret using YML code, so that I don't have to reveal my passwords & other crucial info later in the Ghost & MYSQL launching codes. Here, I am giving an example of a secret creation but not putting in the actual values. Note that the passwords need to Base 64 Encoded before they are put in a secret.**_
 
                              apiVersion: v1
                              kind: Secret
                              metadata:
                                name: mysql-redmine
                              data:
                                 password: cmVkaGF0
                                 dcdatabase: cHJvamVjdF9kYg==
                                    sqlrootpassword: cm9vdHBhc3M=
                                    sqluser: c3BhcnNo
                                    sqlpassword: cmVkaGF0
                                    sqldb: cHJvamVjdF9kYg==
                              
                              
_**Step - 6: We need to launch a MYSQL which will be connected to the RED-MINE. For this purpose, I have used MYSQL version 5.7. I have picked up the Passwords from the pre-created secret file. The YML code is as follows-**_

                                 apiVersion: v1
                                 kind: Service
                                 metadata:
                                   name: redmine-mysql
                                   labels:
                                     app: redmine
                                 spec:
                                   ports:
                                     - port: 3306
                                   selector:
                                     app: redmine
                                     tier: mysql
                                   clusterIP: None
                                 ---
                                 apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
                                 kind: Deployment
                                 metadata:
                                   name: redmine-mysql
                                   labels:
                                     app: redmine
                                 spec:
                                   selector:
                                     matchLabels:
                                       app: redmine
                                       tier: mysql
                                   strategy:
                                     type: Recreate
                                   template:
                                     metadata:
                                       labels:
                                         app: redmine
                                         tier: mysql
                                     spec:
                                       containers:
                                       - image: mysql:5.6
                                         name: mysql
                                         env:
                                         - name: MYSQL_ROOT_PASSWORD
                                           valueFrom:
                                             secretKeyRef:
                                               name: mysql-redmine
                                               key: password
                                         - name: MYSQL_DATABASE
                                           value: redmine_db
                                         ports:
                                         - containerPort: 3306
                                           name: mysql
                                         volumeMounts:
                                         - name: mysql-persistent-storage
                                           mountPath: /var/lib/mysql
                                       volumes:
                                       - name: mysql-persistent-storage
                                         persistentVolumeClaim:
                                           claimName: efs-mysql
                                           
_**Step - 8: Now, it's time to launch our RED-MINE Architecture & connect it to our MYSQL Database. Here also, I have picked up the Passwords & other crucial data from the pre-created secret file that we had created in Step 6. The YML code is as follows-**_

                                          apiVersion: v1
                                          kind: Service
                                          metadata:
                                            name: redmine
                                            labels:
                                              app: redmine
                                          spec:
                                            ports:
                                              - port: 80
                                            selector:
                                              app: redmine
                                              tier: frontend
                                            type: LoadBalancer
                                          ---
                                          apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
                                          kind: Deployment
                                          metadata:
                                            name: redmine
                                            labels:
                                              app: redmine
                                          spec:
                                            selector:
                                              matchLabels:
                                                app: redmine
                                                tier: frontend
                                            strategy:
                                              type: Recreate
                                            template:
                                              metadata:
                                                labels:
                                                  app: redmine
                                                  tier: frontend
                                              spec:
                                                containers:
                                                - image: redmine
                                                  name: redmine
                                                  env:
                                                  - name: REDMINE_DB_HOST
                                                    value: redmine-mysql
                                                  - name: REDMINE_DB_PASSWORD
                                                    valueFrom:
                                                      secretKeyRef:
                                                        name: mysql-redmine
                                                        key: password
                                                  ports:
                                                  - containerPort: 80
                                                    name: redmine
                                                  volumeMounts:
                                                  - name: redmine-persistent-storage
                                                    mountPath: /var/www/html
                                                volumes:
                                                - name: redmine-persistent-storage
                                                  persistentVolumeClaim:
                                                    claimName: efs-redmine

_**Now that our RED-MINE Architecture is launched, we can access it using the DNS & use it -**_


**Here the login page of RED-MINE launched:-**

![maxresdefault](https://user-images.githubusercontent.com/66829650/90312274-b6615100-df20-11ea-845c-b7bd1934fdea.jpg)




