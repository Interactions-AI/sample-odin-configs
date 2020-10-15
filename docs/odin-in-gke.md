## Deploying Odin on GKE from Scratch

In this tutorial, we will bring a GKE cluster online and deploy Odin on it using Cloud SQL.

We assume that the reader has already perused the [guide to running Odin from scratch on MicroK8s](odin-from-scratch.md), and is therefore familiar with the components of Odin and the basics prerequisites.

We are going to do the absolute bare minimum in this tutorial to get set up.  For more information on how to
customize Google Cloud products, please see their documentation.

All of the template YAML for setting up the resources in this cluster can be found in this [repository](../k8s/gke).
Any items with curly braces (`{}`) should replaced with the actual values to be used for your cluster.  

If you followed through the last tutorial and aliased your `microk8s.kubectl` to `kubectl`, we will need to undo that.  Remove that from your `~/.bashrc` and [get the latest](https://kubernetes.io/docs/tasks/tools/install-kubectl/
) release of `kubectl`:

```
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
$ kubectl version --client

```

Now we are going to copy our kubeconfig over to our user directory.  If this directory doesnt exist, make it first

```
$ cd ~/.kube/
```

If you already have this directory and there is already a config, back that up first

```
$ cp config config.bak
```

### Setting up the cluster in GKE

To get started, you will need to either using either cloud shell or a local gcloud install (or the UI),
enable the Kubernetes API.  If you are not familiar with those steps, sees this overview:

https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl

```
$ gcloud services enable container.googleapis.com
```


### First lets init the cluster

Run this command, either from `cloud-shell` or from your local machine.


```
$ gcloud container clusters create odin-k8s --num-nodes=2 --zone us-central1-c
$ gcloud container clusters describe odin-k8s --zone us-central1-c
```

*Note*: If you run it locally, your local `kubeconfig` configuration will automatically be updated to include information on this cluster.  If you run it in cloud shell it will not.  To add after the fact:

```
$ gcloud container clusters get-credentials odin-k8s --zone us-central1-c
Fetching cluster endpoint and auth data.
kubeconfig entry generated for odin-k8s.
$ kubectl config get-contexts
CURRENT   NAME                                 CLUSTER                              AUTHINFO                             NAMESPACE
*         gke_xxxxxxx_us-central1-c_odin-k8s   gke_xxxxxxx_us-central1-c_odin-k8s   gke_xxxxxxx_us-central1-c_odin-k8s   
          microk8s                             microk8s-cluster                     admin                                

```
If your cluster isnt the current context, switch it like this:

```
$ kubectl config use-context gke_xxxxxxx_us-central1-c_odin-k8s
Switched to context "gke_xxxxxxx_us-central1-c_odin-k8s".
```

### Set up the Filestore instance, and the PV/PVC

First, enable Filestore API if you havent.  Then, in the cloud console, create an instance called `odin-data` and make it with a fileshare called `data` (you can set other values, but then you will need to update the PV listed here).  The default is to create 1TB file-system, which is sufficient for this tutorial

The overview page for `Filestore` will show the IP address, which we will need for our PV setup.
The YAML file should look like this:

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: data-odin-gke
    labels:
       type: local
spec:
    storageClassName: manual
    accessModes:
        - ReadWriteMany
    capacity:
        storage: 1T
    nfs:
        path: "/data"
        server: {FILE_SERVER_IP}
    persistentVolumeReclaimPolicy: Retain

```

And you will need a PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-rw-many
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  storageClassName: manual
  resources:
    requests:
      storage: 1T
  volumeName: odin-data

```


```
sample-odin-configs/k8s/gke$ kubectl apply -f pv/data.yml 
persistentvolume/data-odin-gke created
sample-odin-configs/k8s/gke$ kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
data-odin-gke   100Gi      RWX            Retain           Available           manual                  4s
sample-odin-configs/k8s/gke$ vi pvc/data-rw-many.yml 
sample-odin-configs/k8s/gke$ kubectl apply -f pvc/data-rw-many.yml 
persistentvolumeclaim/data-rw-many created
sample-odin-configs/k8s/gke$ kubectl get pvc
NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-rw-many   Bound    odin-data   1T         RWX            manual         32s
```

You can check that this PVC exists by navigating to `Kubernetes Engine` in your Google Cloud Platform console in the browser.

### Setting up a one-time Job to clone the repo

As in the [Microk8s tutorial](odin-from-scratch.md), we are going to use a github repository as a backend for our pipelines.

We could clone the repository if we wanted to create an NFS mount on the cluster, but its actually probably easier just to clone the repo using a Pod:

```
kind: Job
apiVersion: batch/v1
metadata:
   name: git-clone
spec:
   template:
      spec:
         volumes:
         - name: data-volume
           persistentVolumeClaim:
             claimName: data-rw-many
         - name: ssh-config
           configMap:
             name: ssh-config
         - name: ssh-key
           secret:
              secretName: ssh-key
              defaultMode: 0400
         restartPolicy: Never
         containers:
         - name: git-pull
           image: alpine/git
           imagePullPolicy: Always
           command:
           - /bin/sh
           - -c
           - "cd /data; git clone https://github.com/dpressel/sample-odin-pipelines.git pipelines"
           volumeMounts:
           - mountPath: /data
             name: data-volume
           - name: ssh-config
             mountPath: /etc/ssh/ssh_config
             subPath: ssh_config
           - name: ssh-config
             mountPath: /etc/ssh/ssh_known_hosts
             subPath: known_hosts
           - name: ssh-key
             mountPath: /etc/odind/

```

If you run this, you should see output like this:

```
$ kubectl get pods
NAME                         READY   STATUS      RESTARTS   AGE
git-clone-h9tff              0/1     Completed   0          45h
...
$ kubectl logs git-clone-h9tff
Cloning into 'pipelines'...
```

### Set up Cloud SQL with Postgres backend using a Service Account
In the SQL tab in GCP, create a database.  We will name it `pgdb` and for our this tutorial, it is going to be created in `us-central1`

Check out this tutorial for details:

https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine


Using the button labeled `Connect using Cloud Shell`, get cloud shell instance and type:

```
$ gcloud sql connect pgdb --user=postgres --quiet
```
From inside the `psql` shell, we will create 2 instances.

```
postgres=> CREATE DATABASE jobs_db;
CREATE DATABASE
postgres=> CREATE DATABASE odin_db;
CREATE DATABASE
postgres=> CREATE USER {USER} WITH PASSWORD {PASSWORD};
postgres=> grant all privileges on database odin_db to {USER};
GRANT
postgres=> grant all privileges on database jobs_db to {USER};
GRANT
```


From within the shell, create a user and grant that user access to `jobs_db` and `odin_db`


We are going set up our Cloud SQL using a Service Account key file.
Set up a secret containing the database username, password and job database.
Lets also set up a secret for odin to use to create a root user for its user database

```

$ kubectl create secret generic pgjobs   --from-literal=username={DB_USER}   --from-literal=password={DB_PASSWORD}   --from-literal=database=jobs_db
$ kubectl create secret generic pgodin   --from-literal=username={DB_USER}   --from-literal=password={DB_PASSWORD}   --from-literal=database=odin_db
$ gcloud iam service-accounts keys create ~/key.json   --iam-account {YOUR_IAM_ACCOUNT_ID}
$ kubectl create secret generic sak --from-file=service_account.json=~/key.json
secret/sak created
```

### Set up Odin deployment to talk to Cloud SQL Proxy

We will be setting odin up with a sidecar proxy to Cloud SQL for our Odin core deployment and our Odin HTTP one.

#### Odin Core

The core component of odin is a web socket tier, and it will not be accessible from the public internet.

The deployment for odin's core tier should look something like this:

```
kind: Deployment
apiVersion: apps/v1
metadata:
  generation: 1
  labels:
    app: odin
    version: "1"
  name: odin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: odin
  template:
    metadata:
      labels:
        app: odin
    spec:
      serviceAccountName: odin
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-rw-many
      imagePullSecrets:
      - name: registry
      containers:
      - name: odin
        env:
        - name: ODIN_LOG_LEVEL
          value: DEBUG
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: pgjobs 
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: pgjobs 
              key: password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: pgjobs 
              key: database
        image: interactions/odin-ml 
        imagePullPolicy: Always
        args:
        - --root_path
        - /data/pipelines
        - --data_path
        - /data/odin
        ports:
        - containerPort: 30000
        volumeMounts:
        - mountPath: /data
          name: data-volume
      - name: cloud-sql-proxy
        # It is recommended to use the latest version of the Cloud SQL proxy
        # Make sure to update on a regular schedule!
        image: gcr.io/cloudsql-docker/gce-proxy:1.17
        command:
          - "/cloud_sql_proxy"
          - "-instances={CLOUD_SQL_INSTANCE}:pgdb=tcp:5432"
          - "-credential_file=/secrets/service_account.json"
        securityContext:
          # The default Cloud SQL proxy image runs as the
          # "nonroot" user and group (uid: 65532) by default.
          runAsNonRoot: true
        volumeMounts:
        - name: sak-volume 
          mountPath: /secrets/
          readOnly: true
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-rw-many
      - name: sak-volume 
        secret:
          secretName: sak 

```
Notice that we are using our credentials that we set up previously, and we are using our `pgjobs` secret to connect to the `jobs_db` database.

The service looks identical to the [MicroK8s example](odin-from-scratch.md)

Unlike in the Microk8s example, we cannot easily access the cluster to use the `odin-ml` package to test that the core is up and running.

#### Sidenote: Sidecars in Pods

In many cases, there is a single docker container per pod.  Sidecars are named as such because they ride along with the primary container in a pod, meaning that pods with sidecars have (at least) two containers internally.  Normally, with a single pod, you only need to specify the pod for logging.  For multi-container pods, however, you need to specify which container to get logs from (notice the second argument):

```
$ kubectl logs odin-http-7575b8c8cc-7j2c9 cloud-sql-proxy
2020/10/15 17:54:42 current FDs rlimit set to 1048576, wanted limit is 8500. Nothing to do here.
...
$ kubectl logs odin-http-7575b8c8cc-7j2c9 odin-http
...
```

#### Odin HTTP

Odin HTTP defines a REST interface that is intended to be publicly consumed (exposed over some ingress).  It uses JWT for authentication and talks to a database called `odin_db` which we set up previosly.

As part of our deployment, we need to specify a few more parameters that will be passed to Our HTTP deployment.  These can be stored in a secret that we can reference later:

- The root user and password for odin.  This will be created when the service starts up and will be the initial entry in the user table, from which we can create more users using the `odin-user` command.  nNote that this is internal to Odin and thus is different from the postgres user that is used to access the endpoint.
- The secret and the salt/pepper that we will be using for JWT.  These will be string values

```
 kubectl create secret generic odinsecrets   --from-literal=username={USER}   --from-literal=password='{PASSWORD}' --from-literal=salt='{SALT}' --from-literal='{SECRET}'
secret/odinsecrets created
```

The deployment should look like this:

```
kind: Deployment
apiVersion: apps/v1
metadata:
  generation: 1
  labels:
    app: odin-http
  name: odin-http
spec:
  replicas: 1
  selector:
    matchLabels:
      app: odin-http
  template:
    metadata:
      labels:
        app: odin-http
    spec:
      serviceAccountName: odin
      containers:
      - name: odin-http
        env:
        - name: ODIN_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: odinsecrets
              key: username
        - name: ODIN_ROOT_PASS
          valueFrom:
            secretKeyRef:
              name: odinsecrets
              key: password
        - name: ODIN_GIT_NAME
          value: {GIT_USERNAME}
        - name: ODIN_GIT_EMAIL
          value: {GIT_USER_EMAIL}
        - name: ODIN_AUTH_ISSUER
          value: {AN_ISSUER_NAME} 
        - name: ODIN_SECRET
          valueFrom:
            secretKeyRef:
              name: odinsecrets
              key: secret
        - name: ODIN_SALT
          valueFrom:
            secretKeyRef:
              name: odinsecrets 
              key: salt
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: pgodin 
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: pgodin 
              key: password
        - name: ODIN_DB
          valueFrom:
            secretKeyRef:
              name: pgodin 
              key: database
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: pgodin 
              key: database
        image: interactions/odin-ml-http
        imagePullPolicy: Always
        args:
        - --host
        - odin
        - --root_path
        - /data/pipelines
        - --scheme
        - ws
        ports:
        - containerPort: 9003
        readinessProbe:
          httpGet:
            path: /v1/ping
            port: 9003
          initialDelaySeconds: 5
        livenessProbe:
          httpGet:
            path: /v1/ping
            port: 9003
          initialDelaySeconds: 5
        volumeMounts:
        - mountPath: /data
          name: data-volume
      - name: cloud-sql-proxy
        # It is recommended to use the latest version of the Cloud SQL proxy
        # Make sure to update on a regular schedule!
        image: gcr.io/cloudsql-docker/gce-proxy:1.17
        command:
          - "/cloud_sql_proxy"
          - "-instances={CLOUD_SQL_INSTANCE}:pgdb=tcp:5432"
          # This flag specifies where the service account key can be found
          - "-credential_file=/secrets/service_account.json"
        securityContext:
          # The default Cloud SQL proxy image runs as the
          # "nonroot" user and group (uid: 65532) by default.
          runAsNonRoot: true
        volumeMounts:
        - name: sak-volume 
          mountPath: /secrets/
          readOnly: true
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-rw-many
      - name: sak-volume 
        secret:
          secretName: sak 
      - name: ssh-key
        secret:
          secretName: ssh-key
          defaultMode: 0400
      - name: ssh-config
        configMap:
          name: ssh-config

```

The `ODIN_GIT_NAME` and `ODIN_GIT_EMAIL` above can be anything you want.  For example `Odin User` could be the git name.  These are used by git, the same as you would normally set in your git configuration.  For the JWT issuer, an example might be `com.interactions`.

The service YAML will be of type `NodePort`.  This is required since we will be making use of GKE's ingress functionality in the UI:

```
kind: Service
apiVersion: v1
metadata:
  labels:
    app: odin-http
  name: odin-http
spec:
  selector:
    app: odin-http
  ports:
    - name: http-server
      port: 9003
      targetPort: 9003
  type: NodePort
status:
  loadBalancer: {}

```

If everything is looking good at this point, you should see something like this:

```
$ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
odin        1/1     1            1           47h
odin-http   1/1     1            1           21h
$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.119.240.1     <none>        443/TCP          2d3h
odin         ClusterIP   10.119.252.106   <none>        30000/TCP        47h
odin-http    NodePort    10.119.242.225   <none>        9003:31551/TCP   27h
```

We still cannot hit our cluster yet.  To do that, we need to set up Ingress.
We are going to do that with the UI.  In the cloud console in your browser, navigate to `Kubernetes Engine`.  The side tab will have a menu item called `Service & Ingress`.  Click on that.

You should see a list including our service `odin-http` of type `NodePort`.  Click on that and then at the top of the page click on `CREATE INGRESS`.  You can name this Ingress whatever you want. Set up an `External HTTP/S load balancer` and set up thye host and path rules to default to `odin-http`.  Then click `create`.  You can also see the exact YAML used in case you want to do this step from the command line instead.

Once that has been created, you can click on the `Ingress Details` and it will tell you all of the information, including the `Load balancer IP`.  That is the public URL that we are going to use to connect to odin.

If you have not already, from an environment with Python 3.x, do a `pip install odin-ml`.  Then, for convenience, we will create an environment variable for the IP and the PORT.

```
$ export ODIN_URL={LOAD_BALANCER_IP}
$ export ODIN_PORT=80
```

### Creating User accounts

To set up a user account, lets log in as the root user first:

```
$ odin-auth --username={ODIN_ROOT_USER} --scheme http
odin password: ********                                                                                                                                          
{JWT_TOKEN}
$ odin-user --scheme http
create username: {WHOAMI-OUTPUT}                                                                                                                                    
new password: ***************                                                                                                                                    
...

```


The next time you run a job after the current JWT token expires (or the next time you run `odin-auth`, you will be able to give the new user/password.


### Adding GPUs

So far, we have no GPUs set up.  One great thing about GKE over a normal cluster is that you can add node pools of GPUs to the cluster as needed.  Lets add a 2-node pool to our cluster:

```
$ gcloud container node-pools create gpu-pool --num-nodes=2 --accelerator type=nvidia-tesla-v100,count=1 --zone us-central1-c --cluster odin-k8s --machine-type=n1-standard-2
WARNING: Newly created clusters and node-pools will have node auto-upgrade enabled by default. This can be disabled using the `--no-enable-autoupgrade` flag.
WARNING: Starting in 1.12, new node pools will be created with their legacy Compute Engine instance metadata APIs disabled by default. To create a node pool with legacy instance metadata endpoints disabled, run `node-pools create` with the flag `--metadata disable-legacy-endpoints=true`.
Machines with GPUs have certain limitations which may affect your workflow. Learn more at https://cloud.google.com/kubernetes-engine/docs/how-to/gpus
Creating node pool gpu-pool...done.                                             
```

We cannot actually use this cluster yet, first we need to install the GPU drivers:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

This will install of the needed drivers onto the cluster.
Our cluster now should look something like this:

```
$ kubectl get nodes
NAME                                      STATUS   ROLES    AGE     VERSION
gke-odin-k8s-default-pool-2f22d1bb-9gjb   Ready    <none>   2d4h    v1.16.13-gke.401
gke-odin-k8s-default-pool-2f22d1bb-q6fw   Ready    <none>   2d4h    v1.16.13-gke.401
gke-odin-k8s-gpu-pool-65d16d95-2vsm       Ready    <none>   8m52s   v1.16.13-gke.401
gke-odin-k8s-gpu-pool-65d16d95-6wmr       Ready    <none>   8m54s   v1.16.13-gke.401
```

### Optional: Setting up Midgard

By default, GCP prevents pods from scheduling to GPU nodes that do not explicitly require GPUs using a Kubernetes `taint`
To use midgard to monitor, we need to remove this:



```
$ kubectl taint nodes gke-odin-k8s-gpu-pool-65d16d95-2vsm nvidia.com/gpu=present:NoSchedule-
node/gke-odin-k8s-gpu-pool-65d16d95-2vsm untainted
$ kubectl taint nodes gke-odin-k8s-gpu-pool-65d16d95-6wmr nvidia.com/gpu=present:NoSchedule-
node/gke-odin-k8s-gpu-pool-65d16d95-6wmr untainted
```

Now we will set up our midgard daemonset.  Note that GKE clusters run CUDA 10.0, so we need to explicitly request
the `cuda100` tag for midgard:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: midgard
  namespace: default
  labels:
    app: midgard
spec:
  selector:
    matchLabels:
      name: midgard
  template:
    metadata:
      labels:
        name: midgard
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - image: interactions/midgard:cuda100
        name: midgard
        ports:
        - name: midgard
          containerPort: 29999
          hostPort: 29999

      hostNetwork: true
      hostPID: true

```

To run it:

```
$ kubectl apply -f daemonsets/midgard.yml 
```
###  Deleting GPU pools

To delete the GPU pool from the cluster, we can run:

```
gcloud container node-pools delete gpu-pool --zone us-central1-c --cluster odin-k8s
```

###  Rewriting Limits in the default namespace

The default GKE namespace comes with some weird limits.  These limits will cause your jobs to not run properly due to the default CPU limits.  To remove this, we will get the limits and write them to a YAML file and update them

```
$ kubectl get limitrange limits -o yaml > limitrange/limits.yml
```

Comment out the information under `limits:` and replace it with an empty list

```
apiVersion: v1
kind: LimitRange
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"LimitRange","metadata":{"annotations":{},"creationTimestamp":"2020-10-13T15:21:27Z","labels":{"addonmanager.kubernetes.io/mode":"EnsureExists"},"name":"limits","namespace":"default","resourceVersion":"447010","selfLink":"/api/v1/namespaces/default/limitranges/limits","uid":"d8d11d26-4bf5-44c1-96b6-56713c9e213a"},"spec":{"limits":[{"defaultRequest":{"cpu":"1"},"type":"Container"}]}}
  creationTimestamp: "2020-10-13T15:21:27Z"
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
  name: limits
  namespace: default
  resourceVersion: "709077"
  selfLink: /api/v1/namespaces/default/limitranges/limits
  uid: d8d11d26-4bf5-44c1-96b6-56713c9e213a
spec:
  limits: []
```
Then do a `kubectl apply -f limitrange/limits.yml`.
You should now be able to run jobs inside of Odin on GPUs!



```
$ odin-run --scheme http bert-ner
{"pipeline": {"id": "bert-ner-0yog5kfdj", "job": "bert-ner", "name": "bert-ner-0yog5kfdj"}}
$ odin-cleanup bert-ner-0yog5kfdj --scheme http
Results of this request:
task_id                     | cleaned_from_k8s | purged_from_db | removed_from_fs
----------------------------|------------------|----------------|----------------
bert-ner-0yog5kfdj          | No               | No             | No             
bert-ner-0yog5kfdj--conll-0 | No               | No             | No             

```

### Deleting the Cluster

If you want to burn down the entire GKE instance you can run the following command:

```
gcloud container clusters delete odin-k8s --region=us-central1-c
```