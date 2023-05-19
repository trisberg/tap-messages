# Noisy downstream neighbors

When deploying a workload and it is going through all the steps in the supply chain, there are messages for steps not yet run that look like there might be something wrong. This can be unsettling, especially for new developers.

## Command to try

```sh
tanzu apps workload create tanzu-java-web-app \
  --git-repo https://github.com/trisberg/tanzu-java-web-app \
  --git-branch main \
  --type web \
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --label apps.tanzu.vmware.com/auto-configure-actuators=true \
  --label apps.tanzu.vmware.com/has-tests=true \
  --annotation autoscaling.knative.dev/min-scale=1 \
  --build-env "BP_JVM_VERSION=17"
```

Then run this command

```sh
tanzu apps workload get tanzu-java-web-app
```

which should show something like this:

```text
📡 Overview
   name:        tanzu-java-web-app
   type:        web
   namespace:   default

💾 Source
   type:     git
   url:      https://github.com/trisberg/tanzu-java-web-app
   branch:   main

📦 Supply Chain
   name:   source-test-to-url

   NAME               READY     HEALTHY   UPDATED   RESOURCE
   source-provider    True      True      44s       imagerepositories.source.apps.tanzu.vmware.com/tanzu-java-web-app
   source-tester      Unknown   Unknown   44s       runnables.carto.run/tanzu-java-web-app
   image-provider     False     Unknown   48s       not found
   config-provider    False     Unknown   48s       not found
   app-config         False     Unknown   48s       not found
   service-bindings   False     Unknown   48s       not found
   api-descriptors    False     Unknown   48s       not found
   config-writer      False     Unknown   48s       not found

🚚 Delivery
   name:   delivery-basic

   NAME              READY   HEALTHY   UPDATED   RESOURCE
   source-provider   False   False     27s       imagerepositories.source.apps.tanzu.vmware.com/tanzu-java-web-app-delivery
   deployer          False   Unknown   31s       not found

💬 Messages
   Workload [MissingValueAtPath]:   waiting to read value [.status.latestImage] from resource [images.kpack.io/tanzu-java-web-app] in namespace [default]
   Workload [HealthyConditionRule]:   Tasks Completed: 0 (Failed: 0, Cancelled 0), Incomplete: 1, Skipped: 0
   Deliverable [HealthyConditionRule]:   Unable to resolve image with tag "accwest.azurecr.io/tap/tanzu-java-web-app-default-bundle:edd8a493-6b41-4f3e-a33c-6d1865e7b125" to a digest: HEAD https://accwest.azurecr.io/v2/tap/tanzu-java-web-app-default-bundle/manifests/edd8a493-6b41-4f3e-a33c-6d1865e7b125: unexpected status code 404 Not Found (HEAD responses have no body, use GET for details)

🛶 Pods
   NAME                                READY   STATUS    RESTARTS   AGE
   tanzu-java-web-app-fmkhx-test-pod   1/1     Running   0          24s

To see logs: "tanzu apps workload tail tanzu-java-web-app --timestamp --since 1h"
```

## Analysis

The `Delivery` step has not started yet but the `source-provider` is not healthy and the `Deliverable` shows a message of 

```
Unable to resolve image with tag "accwest.azurecr.io/tap/tanzu-java-web-app-default-bundle:edd8a493-6b41-4f3e-a33c-6d1865e7b125"
to a digest: HEAD https://accwest.azurecr.io/v2/tap/tanzu-java-web-app-default-bundle/manifests/edd8a493-6b41-4f3e-a33c-6d1865e7b125:
unexpected status code 404 Not Found (HEAD responses have no body, use GET for details)
```
which isn't really showing that it's a bit early to look for the delivery bundle. The build is just getting started.

The message for the  `Workload [MissingValueAtPath]` says 
```
waiting to read value [.status.latestImage] from resource [images.kpack.io/tanzu-java-web-app] in namespace [default]
```

It's not obvious what `MissingValueAtPath` refers to. This is again due to the build just getting started.

There is also another message `Workload [HealthyConditionRule]` that says
```
Tasks Completed: 0 (Failed: 0, Cancelled 0), Incomplete: 1, Skipped: 0
```

It is not clear what this refers to, I assume the test step.

## Ask

### Start with READY as Unknown

Could we change the READY for `source-provider` of the `Delivery` to be "Unknown" like we do for all other steps that have not started yet.

```
   NAME              READY   HEALTHY   UPDATED   RESOURCE
   source-provider   False   Unknown   26s       imagerepositories.source.apps.tanzu.vmware.com/tanzu-java-web-app-delivery
   deployer          False   Unknown   29s       not found
```

### Don't show messages for steps with READY as Unknown

There doesn't seem to be much point in showing messages for the Delivery resources until there is something to report and the step is either ready or has an error.
