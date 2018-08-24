# Installing Kubernetes in AWS using kops

I just wanted to have a straight forward single place knowledge of things to be done for installing kubernetes in AWS using kops.

## Prerequisites

### Step 1: User creation
I had an user with sufficient priviledes as mentioned in [this](https://github.com/kubernetes/kops/blob/master/docs/aws.md) document. So, I have skipped the first part. Otherwise, please follow this step to create your user.

### Step 2: Domain creation

Once again, I already had a domain so, I just followed the steps given in Scenario 1b as mentioned in the above document. However, just for simplicity, I added a subdomain under my parent domain.

So, if my parent domain = example.com and my sub-domain is subdomain.example.com then I followed the following steps:

1. Go to AWS Console > Route 53
2. Click 'Create Hosted Zone'
3. Create the hosted zone with domain name = subdomain.example.com
4. Click 'Create'

5. Click the radio button on the left of subdomain entry
6. Copy the Name Servers list from the right pane
7. Click the name of the Parent domain i.e. example.com
8. Click 'Create Record Set'
9. Create a record with the following:

```
name: subdomain.example.com
type: NS
ttl: 300
values: <NS records available in the subdomain.example.com> which you have just copied
```

1. Now, wait for some time to let the DNS entries get propagated.
2. To check if you are ready(Note: last dot is not a mistake):

```bash
$ dig ns example.subdomain.com.
```
This should give you all the name servers of the subdomain.example.com

### Step 3: Bucket creation

```
$ aws s3api create-bucket --bucket subdomain-kops --region us-east-1
```

### Step 4: Cluster Creation

Export the variables:

```bash
$ export NAME=subdomain.example.com
$ export KOPS_STATE_STORE=s3://subdomain-kops
```

Create the cluster:

```bash
$ kops create cluster $NAME --zones us-west-2a --yes # For applying
```

After that you need to wait for 5-10 mins for the cluster to be ready. To check if it is ready, run:

```bash
$ kops validate cluster
```
When it is ready, it should respond with something like:

```
Validating cluster subdomain.example.com

INSTANCE GROUPS
NAME            ROLE    MACHINETYPE    MIN    MAX    SUBNETS
master-us-west-2a    Master    m3.medium    1    1    us-west-2a
nodes            Node    t2.medium    2    2    us-west-2a

NODE STATUS
NAME                        ROLE    READY
ip-172-30-41-28.us-west-2.compute.internal    master    True
ip-172-30-45-245.us-west-2.compute.internal    node    True
ip-172-30-43-211.us-west-2.compute.internal    node    True

Your cluster subdomain.example.com is ready
```

### Step 4: Setting up Helm

Now, we need to setup helm. First, we need to install Tiller.

```bash
$ helm init
```

Then, we need to create the service account and configure it properly so that the helm runs properly.

```bash
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

### What next?

After this you should be ready with the cluster setup and you can run the standard kubectl commands.

You can check out [kubernetes-quickstart](https://github.com/onlydevelop/kubernetes-quickstart) for the commands as specified in Step 5.

Please note in order to make it run, you need to point the image to a image available in public repo like docker hub. The images as created in the `kubernetes-quickstart` are available in `onlydevelop/java-sample-demo`. You can use 0.2 tag because, it is a smaller size image.

And to make that run, you need to change the deployment.yaml's image from: `java-sample-demo:0.1` to `onlydevelop/java-sample-demo:0.2`, so that it would look like:

```
...
spec:
    containers:
    - name: sample-app
      image: onlydevelop/java-sample-demo:0.2
      image-pull-policy: Never
      ports:
      - containerPort: 8080
...
```
And you should be good to go.

### Disclaimer

Once again, all these information as written above are actually available in different places in the internet. However, I just collated them in one place so that, I can go that in one shot - without watsing time on troubleshooting.

