# Demo: Crossplane Devex Improvements

This demo walks through a typical platform builder experience using previous
versions of Crossplane. It demonstrates that building your platform was somewhat
challenging and it could be difficult to understand where errors may be lurking.

Then the demo uses the new `convert` command to migrate to a functions based
composition and uses new local development tooling with a great experience to
rapidly find issues in your platform and get to a functional successful state.

## Pre-reqs

### Create a new control plane
```
kind create cluster
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace --set args='{"--enable-composition-webhook-schema-validation=false"}'
```

### Install Providers and config
```
kubectl apply -f provider.yaml

# first generate GCP creds file with https://docs.crossplane.io/latest/getting-started/provider-gcp/#generate-a-gcp-service-account-json-file
kubectl create secret generic gcp-secret -n crossplane-system --from-file=creds=./gcp-credentials.json
kubectl apply -f providerconfig.yaml
```

## Deploy platform (the hard way)

Using an older (and more challenging) experience of Crossplane, you would need
to test the platform you're building by deploying to a live control plane and
provisioning real world resources. That's slow, expensive, and who really has
the patience for that?

```
kubectl apply -f pre/definition.yaml
kubectl apply -f pre/composition.yaml
kubectl get xrd
```

Now we can test the platform we've built by provisioning an `AcmeDatabase`
resource:
```
kubectl apply -f pre/claim.yaml
```

Let's examine the database we just requested:
```
kubectl get claim
kubectl get composite
kubectl get managed
```

Let's check the database settings in GCP:
```
kubectl get DatabaseInstance -l crossplane.io/claim-name=acme-db-prod -o json | jq '.items[0].spec.forProvider.settings'
```

Hmm, wait a minute - we asked for a 100GB database, right? But we're getting one
that is only 10GB. What could possibly be going wrong here?

```
kubectl get acmedatabase -o json | jq '.items[0].spec.storageGB'
```

Creating real resources and waiting for them is a long tough cycle to debug.
Isn't there a better way to iterate on my platform instead of provisioning live
resources and seeing what happens?


## Platform iteration made easier with Functions & DevEx

Let's migrate to the new way of building platforms in Crossplane that has a much
improved platform builder experience!

The `convert` command will migrate our classic patch & transform `Composition`
to one that has a Functions pipeline. Note that most of the content doesn't
actually change, because while we're now using functions, we are using
[`function-patch-and-transform`](https://github.com/crossplane-contrib/function-patch-and-transform/)
that has an almost identical API as classic P&T.

```
crossplane beta convert pipeline-composition pre/composition.yaml -o post/composition-func.yaml
```

Now we can build, test, iterate, and verify our platform all on our local
laptop! No need to provision live resources in the cloud on a live control
plane, we can get instant feedback as we make changes!

```
crossplane beta render post/xr.yaml post/composition-func.yaml post/functions.yaml
```

Aha! The issue is we're patching to the `discSize` field, which is a typo from
the correct `diskSize` field. We see this immediately when we run the local
`render` of our Composition:
```yaml
   settings:
    - discSize: 100
      diskSize: 10
```

Let's fix this minor (but majorly pesky) bug and try again:

```
crossplane beta render post/xr.yaml post/composition-func.yaml post/functions.yaml
```

To be even more confident that we're building a correct composition, let's also
run `validate` to verify all the generated resources are compliant with their
schemas:
```
crossplane beta render post/xr.yaml post/composition-func.yaml post/functions.yaml -x | crossplane beta validate post/extensions.yaml --cache-dir="${HOME}/.crossplane/cache" -
```

### Clean up

When we're done, make sure to clean up any cloud resources we created:
```
kubectl delete -f pre/claim.yaml
kubectl get managed
kind delete cluster
```