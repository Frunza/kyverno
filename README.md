# Kyverno

## Goal

Learn how `Kyverno` works in `k8s` and test it out for yourself.

## Prerequisites

A `k8s` cluster.

## What we have

Let's start with a testing deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testapp
  namespace: testapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
        - name: testapp
          image: hashicorp/http-echo:latest
          args:
            - "-text=Test App is Working!"
          ports:
          - containerPort: 5678
```
, a service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: testapp-service
  namespace: testapp
spec:
  type: LoadBalancer
  selector:
    app: testapp
  ports:
    - name: http
      port: 80
      targetPort: 5678
```
, and a namespace:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: testapp
```
If the yaml file is found in a directory named *testapp*, we can add it and other future yaml files to the cluster with the following command:
```sh
kubectl apply -f testapp
```

## Basic understanding

`Kyverno` is a policy engine designed specifically for `k8s`. Think of it as a customizable gatekeeper that sits inside your cluster and watches every resource creation and update request. Note that `k8s` native policies only work when a resource is created or updated, so if you alrady have stuff in your cluster, you will need something like `Kyverno` if you want to enforce policies.
Another advantage in using `Kyverno` is that it generates reports that you can find later, while `k8s` native policies just block stuff and generate an error at that time.
`Kyverno` is also a good choice if you have resources that pull in external data, or actively manage resources throughout their lifecycle, although this is not that usual.
Another great advantage in using `Kyverno` is especially noteworthy for developers. The reason for it is that `Kyverno` offers the same YAML syntax you already use for Kubernetes resources, which make the policies very easy to read, understand and maintain.
Let's take an example. If we want deploy a policy to block privileged containers, we can do that native in `k8s`, using a *ValidatingAdmissionPolicy*:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "restrict-privileged-pods"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: "!has(object.spec.containers) || object.spec.containers.all(c, !has(c.securityContext) || !has(c.securityContext.privileged) || c.securityContext.privileged != true)"
      message: "Privileged containers are not allowed"
```
, and a *ValidatingAdmissionPolicyBinding*:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "restrict-privileged-pods-binding"
spec:
  policyName: "restrict-privileged-pods"
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        security-level: "restricted"
```
Take a look at the *ValidatingAdmissionPolicy*. Is the validation expression easy to read? Do you know how to fix it if it does not work for some reason?
Let's see how to do the same in `Kyverno`:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-privileged
      match:
        any:
        - resources:
            kinds:
              - Pod
      validate:
        message: "Privileged containers are not allowed"
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```
If you take a look at the container configuration, it is self-explanatory just by reading it. `Kyverno` takes care of the implementation of it. This is the exact reason why most developers love `Kyverno` more.

## Setting up Kyverno

Let's first create a namespace:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kyverno
```
, and apply it:
```sh
kubectl apply -f kyverno/namespace.yaml
```

`Kyverno` can be found in a `Helm` repository. Let’s add it:
```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

Let's see what versions are available for `Kyverno` by running:
```sh
helm search repo kyverno/kyverno --versions
```
, which will give us something like:
```sh
NAME                    	CHART VERSION	APP VERSION	DESCRIPTION
kyverno/kyverno         	3.7.1        	v1.17.1    	Kubernetes Native Policy Management
kyverno/kyverno         	3.7.0        	v1.17.0    	Kubernetes Native Policy Management
kyverno/kyverno         	3.6.2        	v1.16.2    	Kubernetes Native Policy Management
kyverno/kyverno         	3.6.1        	v1.16.1    	Kubernetes Native Policy Management
```

Let's create a configuration yaml file for the `Helm` chart according to [official documentation](https://kyverno.io/docs/installation/installation/):
```yaml
admissionController:
  replicas: 3
backgroundController:
  replicas: 3
cleanupController:
  replicas: 3
reportsController:
  replicas: 3
```
Now we can install the `Kyverno` helm chart:
```sh
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --version 3.7.1 \
  -f kyverno/kyverno-values.yaml
```

## Trying things out

Let's create a policy to require all probes:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-all-probes-flexible
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-probes
      match:
        any:
        - resources:
            kinds:
              - Deployment
              - StatefulSet
              - DaemonSet
      validate:
        message: "All containers must have startup, readiness, and liveness probes defined."
        pattern:
          spec:
            template:
              spec:
                containers:
                  - name: "*"
                    livenessProbe: {}
                    readinessProbe: {}
                    startupProbe: {}
```
, and apply it:
```yaml
kubectl apply -f kyverno/policy-require-all-probes.yaml
```
Since our testing application is already deployed, `Kyverno` will generate a report. Let's confirm this by calling:
```sh
kubectl get policyreport -n testapp -o yaml
```
This will output something like:
```yaml
...
- apiVersion: wgpolicyk8s.io/v1alpha2
  kind: PolicyReport
  metadata:
    creationTimestamp: "2026-02-19T15:47:47Z"
    generation: 4
    labels:
      app.kubernetes.io/managed-by: kyverno
    name: c9ba5f9a-4197-422a-9116-657d5c1357de
    namespace: testapp
    ownerReferences:
    - apiVersion: apps/v1
      kind: Deployment
      name: testapp
...
  - message: 'validation error: All containers must have startup, readiness, and liveness
      probes defined. rule require-probes failed at path /spec/template/spec/containers/0/livenessProbe/'
    policy: require-all-probes-flexible
    properties:
      process: background scan
    result: fail
    rule: require-probes
...
```
, which looks as expected. From this report we understand what is wrong and what we need to do.

Let's delete the deployment and add it again:
```sh
kubectl delete -f testapp/deployment.yaml
kubectl apply -f testapp/deployment.yaml
```
Now we get the following output in the console:
```sh
Error from server: error when creating "testapp/deployment.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Deployment/testapp/testapp was blocked due to the following policies

require-all-probes-flexible:
  require-probes: 'validation error: All containers must have startup, readiness,
    and liveness probes defined. rule require-probes failed at path /spec/template/spec/containers/0/livenessProbe/'
```
In this case, `Kyverno` blocked our deployment because it is violates some of its policies, which is our expectation also.

Let's fix our deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testapp
  namespace: testapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
        - name: testapp
          image: hashicorp/http-echo:latest
          args:
            - "-text=Test App is Working!"
          ports:
          - containerPort: 5678
          startupProbe:
            httpGet:
              path: /
              port: 5678
            failureThreshold: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 15
            periodSeconds: 20
```
If we apply it now:
```sh
kubectl apply -f testapp/deployment.yaml
```
, it is deployed successfully.

Let's create another policy to enforce limits and requests:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-resource-limits
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: validate-resources
      match:
        any:
        - resources:
            kinds:
              - Deployment
              - StatefulSet
              - DaemonSet
              - Job
              - CronJob
      validate:
        message: "All containers must have CPU and memory requests AND limits defined."
        pattern:
          spec:
            template:
              spec:
                containers:
                  - name: "*"
                    resources:
                      requests:
                        memory: "?*"
                        cpu: "?*"
                      limits:
                        memory: "?*"
                        cpu: "?*"
```
, and apply it:
```yaml
kubectl apply -f kyverno/policy-validate-resource-limits.yaml
```
Since our testing application is already deployed, `Kyverno` will generate a report. Let's confirm this by calling:
```sh
kubectl get policyreport -n testapp -o yaml
```
This will output something like:
```yaml
...
- apiVersion: wgpolicyk8s.io/v1alpha2
  kind: PolicyReport
  metadata:
    creationTimestamp: "2026-02-19T16:54:49Z"
    generation: 1
    labels:
      app.kubernetes.io/managed-by: kyverno
    name: 2751378d-5974-4c01-a0a8-21c2890abb8b
    namespace: testapp
    ownerReferences:
    - apiVersion: v1
      kind: Pod
...
  results:
  - message: 'validation error: All containers must have CPU and memory requests AND
      limits defined. rule validate-resources failed at path /spec/template/'
    policy: validate-resource-limits
    properties:
      process: background scan
    result: fail
...
- apiVersion: wgpolicyk8s.io/v1alpha2
  kind: PolicyReport
  metadata:
    creationTimestamp: "2026-02-19T16:54:48Z"
    generation: 1
    labels:
      app.kubernetes.io/managed-by: kyverno
    name: 3791ba7b-8e96-464c-be1b-1e40b7d3dd88
    namespace: testapp
    ownerReferences:
    - apiVersion: apps/v1
      kind: Deployment
      name: testapp
...
  results:
  - message: validation rule 'require-probes' passed.
    policy: require-all-probes-flexible
    properties:
      process: background scan
    result: pass
...
```
, which should looks as expected: we notice that the probes policy passed and the resource limits failed.
Just as before, let's delete the deployment and add it again:
```sh
kubectl delete -f testapp/deployment.yaml
kubectl apply -f testapp/deployment.yaml
```
Now we get the following output in the console:
```sh
Error from server: error when creating "testapp/deployment.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Deployment/testapp/testapp was blocked due to the following policies

validate-resource-limits:
  validate-resources: 'validation error: All containers must have CPU and memory requests
    AND limits defined. rule validate-resources failed at path /spec/template/spec/containers/0/resources/limits/'
```
In this case, `Kyverno` blocked our deployment because it violates one of its policies, which is our expectation also.

Let's fix our deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testapp
  namespace: testapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
        - name: testapp
          image: hashicorp/http-echo:latest
          args:
            - "-text=Test App is Working!"
          ports:
          - containerPort: 5678
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
          startupProbe:
            httpGet:
              path: /
              port: 5678
            failureThreshold: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 15
            periodSeconds: 20
```
If we apply it now:
```sh
kubectl apply -f testapp/deployment.yaml
```
, it is deployed successfully.


Let's create another policy to enforce *app* label for services:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-app-label
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-app-label
      match:
        any:
        - resources:
            kinds:
              - Service
      validate:
        message: "Services must have 'app' label."
        pattern:
          metadata:
            labels:
              app: "?*"
```
, and apply it:
```yaml
kubectl apply -f kyverno/policy-require-app-label.yaml
```
Since our testing application is already deployed, `Kyverno` will generate a report. Let's confirm this by calling:
```sh
kubectl get policyreport -n testapp -o yaml
```
This will output something like:
```yaml
...
- apiVersion: wgpolicyk8s.io/v1alpha2
  kind: PolicyReport
  metadata:
    creationTimestamp: "2026-02-19T17:00:24Z"
    generation: 3
    labels:
      app.kubernetes.io/managed-by: kyverno
    name: c459fa29-1a83-49bd-b306-ecf201031011
    namespace: testapp
    ownerReferences:
    - apiVersion: apps/v1
      kind: Deployment
      name: testapp
...
  results:
  - message: validation rule 'require-probes' passed.
    policy: require-all-probes-flexible
    properties:
      process: background scan
    result: pass
...
  - message: validation rule 'validate-resources' passed.
    policy: validate-resource-limits
    properties:
      process: background scan
    result: pass
...
- apiVersion: wgpolicyk8s.io/v1alpha2
  kind: PolicyReport
  metadata:
    creationTimestamp: "2026-02-19T17:06:50Z"
    generation: 1
    labels:
      app.kubernetes.io/managed-by: kyverno
    name: f8568394-e000-461b-b570-a2ef06064648
    namespace: testapp
    ownerReferences:
    - apiVersion: v1
      kind: Service
      name: testapp-service
...
  - message: 'validation error: Services must have ''app'' label. rule check-app-label
      failed at path /metadata/labels/'
    policy: require-app-label
    properties:
      process: background scan
    result: fail
...
```
, which should looks as expected: we notice that the probes and resource limits policies passed and the require *app* label failed.
Just as before, let's delete the service and add it again:
```sh
kubectl delete -f testapp/service.yaml
kubectl apply -f testapp/service.yaml
```
Now we get the following output in the console:
```sh
Error from server: error when creating "testapp/service.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Service/testapp/testapp-service was blocked due to the following policies

require-app-label:
  check-app-label: 'validation error: Services must have ''app'' label. rule check-app-label
    failed at path /metadata/labels/'
```
In this case, `Kyverno` blocked our service because it is violates some of its policies, which is our expectation also.

Let's fix our service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: testapp-service
  namespace: testapp
  labels:
    app: testapp
spec:
  type: LoadBalancer
  selector:
    app: testapp
  ports:
    - name: http
      port: 80
      targetPort: 5678
```
If we apply it now:
```sh
kubectl apply -f testapp/service.yaml
```
, it is deployed successfully.

## What We Learned

`Kyverno` lets you write `k8s` policies using familiar YAML, so no need to learn complex languages. Unlike native policies that only check resources at creation time, `Kyverno` continuously scans your entire cluster and generates persistent *PolicyReport* resources showing compliance status.

We saw this in action: existing resources appeared as fail in reports but kept running (safety first), while new non-compliant deployments were actively blocked. `Kyverno` also works alongside tools like *ResourceQuota* policies ensure individual resources are well-formed, quotas control namespace totals.

By the end, we built three practical policies, watched them report violations, block bad deployments, and learned how to fix non-compliant resources. You're now ready to start implementing policy-as-code in your own clusters.

## Cleanup

If you followed along, run the command to clean everything up:
```sh
kubectl delete -f testapp
helm uninstall kyverno --namespace kyverno
kubectl delete -f kyverno
```
