# Functions in Upbound

This demo shows off [Composition
Functions](https://docs.crossplane.io/latest/concepts/composition-functions/)
running in Upbound. By the end of the demo, we will have created a `UserAccess`
composition that uses
[`function-go-templating`](https://github.com/upbound/function-go-templating) to
create a variable number of IAM related resources in AWS.

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

## Build your new platform API

We want to build a composition that creates a variable number of IAM related
resources in AWS. We will use `function-go-templating` and its iteration
capabilities to loop and create the number of resources requested by the user.

### Develop the composition locally

1. First, let's define the `UserAccess` API that will allow the user to specify
the number of IAM resources in `.spec.count`. We create this new API by creating
a `CompositeResourceDefinition` (XRD). This XRD can be found in
[xrd.yaml](./xrd.yaml).
1. Next, we create the composition that contains the resource creation and
iteration logic. This can be found in [composition.yaml](./composition.yaml).
1. We specify the functions that our composition calls from its pipeline, which
   can be seen in [functions.yaml](./functions.yaml).
1. Finally, let's test our logic locally with an example XR instance and the
`crossplane beta render` command to see if it's working as we expect:

```
crossplane beta render xr.yaml composition.yaml functions.yaml
```

In the output, we see multiple `User` and `AccessKey` objects. The number of
them varies with the value in the `UserAccess`'s `spec.count`. Try changing some
of the values and logic in the XR and Composition to see how that affects the
output of `render`.

Everything looks like it's working correctly here, so let's move on to trying
this in our real control plane.

### Try your composition on a real control plane

Let's define our new `UserAccess` API by applying `xrd.yaml` to the control
plane:

```
kubectl apply -f xrd.yaml
```

Now, let's apply the composition logic by applying
[composition.yaml](./composition.yaml) to the control plane:

```
kubectl apply -f composition.yaml
```

We're ready to create claim of our `UserAccess` type, which should result in
live resources being created in AWS.

```
kubectl apply -f claim.yaml
```

Let's check in on the progress of our resources being created using the `trace`
command:

```
crossplane beta trace useraccessclaim/example
```

After a short time, we should see as many `User` and `AccessKey` resources as
specified by our claim's `spec.count`.

### Explore resources in the console

In addition to this nice `trace` tree view, we can examine the same resources in
the rich console list and graph views.

```
Navigate to your control plane in https://console.upbound.io/ and click on the `UserAccessClaim` resource `example` on the Resources tab, followed by the List and Graph buttons.
```

## Clean up cloud resources

When we are done with this demo, we can clean up the cloud resources with:

```
kubectl delete -f claim.yaml
```