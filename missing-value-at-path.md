# MissingValueAtPath

Deploying a workload shows `MissingValueAtPath` while it is still being built.

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

## Result

### kubectl

```sh
kubectl get workload tanzu-java-web-app
```
```text
NAME                 SOURCE                                           SUPPLYCHAIN          READY     REASON               AGE
tanzu-java-web-app   https://github.com/trisberg/tanzu-java-web-app   source-test-to-url   Unknown   MissingValueAtPath   9s
```

## Analysis

Running `kubectl get workload tanzu-java-web-app -oyaml` produces large amount of confusing status message while app is being built.


Some examples:

```
  - lastTransitionTime: "2023-05-18T19:01:45Z"
    message: waiting to read value [.status.outputs.url] from resource [runnables.carto.run/tanzu-java-web-app]
      in namespace [default]
    reason: MissingValueAtPath
    status: Unknown
    type: Ready
```

It's not really clear what this indicates. Is it an error?

---

```
    - lastTransitionTime: "2023-05-18T19:01:37Z"
      message: "unable to stamp object for resource [image-provider] for template
        [ClusterImageTemplate/kpack-template] in supply chain [source-test-to-url]:
        unable to apply ytt template: ytt: Error: \n- struct has no .source field
        or method (did you mean .sources?)\n    in <toplevel>\n      stdin.yml:43
        |       url: #@ data.values.source.url\n"
      reason: TemplateStampFailure
      status: "False"
      type: Ready
```

Did I forget to include a `.source` field?

---

```
    - lastTransitionTime: "2023-05-18T19:01:37Z"
      message: "unable to stamp object for resource [service-bindings] for template
        [ClusterConfigTemplate/service-bindings] in supply chain [source-test-to-url]:
        unable to apply ytt template: ytt: Error: \n- struct has no .app_def field
        or method\n    in add_claims\n      stdin.yml:147 | #@    return struct.decode(data.values.configs.app_def.config)\n
        \   in <toplevel>\n      stdin.yml:157 | data: #@ add_claims()\n"
      reason: TemplateStampFailure
      status: "False"
      type: Ready
```

Again, did I forget something? Should I provide an `.app_def` field somewhere?


## Ask

Could we show more developer focused messages?
