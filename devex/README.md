# Developer Experience Improvments

Improving the developer (platform builder) experience in Crossplane was a major
investment in [v1.14](https://github.com/crossplane/crossplane/issues/4648).
This demo shows off some of the new capabilities that make building your
platform easier than ever.

## Create a control plane

Log into https://console.upbound.io and navigate to Control Planes to create a
new control plane.  This new control plane should install the Configuration
https://github.com/jbw976/configuration-eks to get started.

### Connect to the control plane

Once the control plane is created and updated to the latest version of the
Configuration, we can connect to it using the [`up`
CLI](https://github.com/upbound/up) while specifying an API token. Don't forget
to first `up ctp disconnect` if you've previously connected to a different
control plane.

```
up ctp connect jared-01 -a jaredorg --token=<redacted>
```

### Add AWS authentication to the control plane

For our control plane to be able to create resources in AWS, we'll need to add
basic authentication.

```
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > creds.conf

kubectl create secret generic aws-creds -n upbound-system --from-file=credentials=./creds.conf

kubectl apply -f aws-default-provider.yaml
```

## Request a new workload cluster from our platform

Now that the platform is configured and running, we can use it to request new infrastructure from its platform API.  Let's create a new workload cluster (EKS) using the `KubernetesCluster` abstraction available in our platform.

```
kubectl apply -f cluster.yaml
```

### Watch resource creation progress with `trace`

This will initiate the creation of the many resources (and nested composites)
that are defined by our `KubernetesCluster` type.  We can check in on this
progress using the helpful new `trace` command in the `crossplane` CLI:

```
crossplane beta trace kubernetescluster/my-cluster
```

Over time (slowly), we'll see resources being created and becoming ready. Now
that we can see all of the objects involved in this complex composite resource,
we can choose individual objects to drill down into them further and get more
details about their progress or any issues they are having:

```
kubectl describe VPC/my-cluster-fghjk-w2crz
```

### Other `trace` output formats

The default view is not the only visualization option we have, we can also look
at all of the resources in a rich `json` format:

```
crossplane beta trace kubernetescluster/my-cluster -o json > trace.json
cat trace.json | jq .
```

With the full `json` output of all the objects in our XR, we can perform queries
with `jq` to get specific fields are are interested in, such as all the kinds of
child managed resources:
```
cat trace.json | jq '.children[].children[].children[].object.kind'
```

As well as all of their status conditions:
```
cat trace.json | jq '.children[].children[].children[].object | .kind, .status.conditions'
```

Finally, we can also generate a graph visualization using the `dot` format:

```
crossplane beta trace kubernetescluster/my-cluster -o dot | dot -Tpng -o trace.png
open trace.png
```