# Kubernetes .NetCore services with Tye, Dapr and Istio
This solution use Project Tye for the services deployment on a Kubernetes environment, Dapr as sidecar and Istio as service discover.

## Init Project Tye 
From the rooth path, run the follow command to create the tye.yaml file with all the services integrated

```sh
tye init
```

After that, you will see a tye.yaml file created.

```yaml
# tye application configuration file
# read all about it at https://github.com/dotnet/tye
#
# when you've given us a try, we'd love to know what you think:
#    https://aka.ms/AA7q20u
#
name: darp_istio
services:
- name: servicea
  project: ServiceA/ServiceA.csproj
- name: serviceb
  project: ServiceB/ServiceB.csproj
```

## Init Dapr in your cluster
We are going to launch Dapr in our cluster, but also we need to disable the mTLS because we are going to let istio do that.

If Dapr is running yet, uninstall it after execute the init command:
```sh
dapr uninstall -k
```

Output:
```sh
ℹ️  Removing Dapr from your cluster...
✅  Dapr has been removed successfully
```

Init Dapr:
```sh
dapr init --enable-mtls=false -k
```

Output:
```sh
⌛  Making the jump to hyperspace...
ℹ️  Note: To install Dapr using Helm, see here: https://docs.dapr.io/getting-started/install-dapr-kubernetes/#install-with-helm-advanced

✅  Deploying the Dapr control plane to your cluster...
✅  Success! Dapr has been installed to namespace dapr-system. To verify, run `dapr status -k' in your terminal. To get started, go here: https://aka.ms/dapr-getting-started
```

## Add Dapr as extension
Project tye allow us to specified the Dapr extension
