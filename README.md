# Introduction
​
Kubernetes manifest defines the workload objects and available resources it can use. There are several object types, including Deployments, Services, Jobs, DaemonSet, and so on. In this tutorial, we will show how to create Kubernetes templates using Jsonnet and our in house libraries.
​
​
# Prerequisites 
​
1. Install Jsonnet using ```brew install jsonnet```
2. Download the Zendesk's Jsonnet libraries using this [tutorial](https://github.com/zendesk/config-docs/blob/master/jsonnet/how-to-use-jsonnet-libs.md).
3. Install [Kubeval](https://kubeval.instrumenta.dev/installation/) to evaluate whether the manifest is valid
​
​
# Create Kubernetes Manifests
​
All manifests require 4 fields: apiVersion, kind, metadata, and spec. Here at Zendesk, we enforce additional rules to ensure smooth deployment with our system. In particular, inside the metadata field, a labels field is required which is used to describe your project. These rules are defined by the Zendesk OPA Gatekeeper and a full list of the rules are [here](https://github.com/zendesk/opa-gatekeeper/blob/master/rules.md).
​
```
apiVersion: apps/v1   # desired version of API group
kind: Job             # desired resource type
metadata:             # specify the identity of the resource
   name:
   labels:
      project:
      role:
      team:
spec:              
  containers:
```
​
We will use Jsonnet to generate our Kubernetes manifest file. Take a look at the ```K8s.libsonnet``` file to see all the predefined workloads. We will go over two examples, the **Job** resource which is defined in the ```K8s.libsonnet``` library and the **DaemonSet** resource which is not defined. 
​
## Job
A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created [\[1\]](https://kubernetes.io/docs/concepts/workloads/controllers/job/).
​
​
The following is the code for our ```.jsonnet``` file. Inside this file we defined the parameters for the ``K.Job`` function. These parameters are defined from other functions within the ``K8s`` library.  
```
local K = import 'lib/K8s.libsonnet';
​
local labels = {
  project: 'new-app',
  role: 'worker',
  team: 'awesome-team',
};
​
local containerResources = {
  limits: { cpu: '0.2', memory: '128Mi' },
  requests: { cpu: '0.2', memory: '128Mi' },
};
​
local container = K.Container.
  withName(labels.project).
  withImage('gcr.io/docker-images-180022/apps/new-app@sha256:abc123').
  withResources(containerResources);
​
local podTemplate = K.PodTemplate.
  withContainers([container]).
  withEnvFromMap({ CLUSTER: 'pod998' }).
  withRestartPolicy('OnFailure');  // specifies restarts on failure
​
local job = K.Job.
  withName('new').
  withLabels(labels).
  withPodTemplate(podTemplate).
  withBackoffLimit(3);  // defines number of retries on failure
​
std.manifestYamlDoc(job)
```
​
We defined all the required fields: apiVersion, kind, metadata, and spec, same as all the other manifests. In this example, K.Container passes in a pre-existing docker image with a defined name and resources using the Jsonnet libraries. The containerResources contains `Request` and `Limits` and more information about how to manage resources can be found [here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). K.PodTemplate has the pre-defined functions that allow passing the Container object as parameters. It generates a template with metadata and spec which includes the container information. K.Job sets the ‘apiVersion’ to the default value ‘batch/v1’ and the ‘kind’ to ‘Job’.
​
### Notice
​
A Job resource requires the definition of restart policy. The withRestartPolicy function used to set the policy cannot be set to ‘always’. Since Job will not restart a pod if it terminates successfully, we can only set it to either “Never” or “OnFailure”.
​
A Job resource does not require a Selector since the resource automatically gives a label to a pod when it is created. However, we can also define and overwrite the label selector ourselves but make sure it is unique to the pod of that job.
​
### Run
Run ```jsonnet -J vendor -S job.jsonnet -o worker.yml```  to generate the Kubernetes manifest using for deployment purpose. The manifest is shown below. 
​
```
"apiVersion": "batch/v1"
"kind": "Job"
"metadata":
  "labels":
    "project": "new-app"
    "role": "worker"
    "team": "awesome-team"
  "name": "new"
"spec":
  "backoffLimit": 3
  "template":
    "metadata": {}
    "spec":
      "containers":
      - "env":
        - "name": "CLUSTER"
          "value": "pod998"
        "image": "gcr.io/docker-images-180022/apps/new-app@sha256:abc123"
        "name": "new-app"
        "resources":
          "limits":
            "cpu": "0.2"
            "memory": "128Mi"
          "requests":
            "cpu": "0.2"
            "memory": "128Mi"
      "initContainers": []
      "restartPolicy": "OnFailure"
      "terminationGracePeriodSeconds": 30
      "volumes": []
​
```
Verify your output using kubeval
```
kubeval worker.yml 
```
​
## DaemonSet
​
​
If you take a look within the ``K8s`` library, you can see that the **DaemonSet** is not defined. This means that we need to implement it ourselves. An example of a  **DaemonSet** manifest can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/). 
​
Similar to what we with the **Jobs** Jsonnet file, we first import the ``K8s`` library, as we will be using some helper functions provided.
​
```
local K = import 'github.com/zendesk/jsonnet-kubernetes/lib/K8s.libsonnet';
```
​
We define the labels, container, and podTemplate as follows
```
local labels = {
  project: 'test-project',
  role: 'daemon',
  team: 'pc-team',
};
​
local containerResources = {
  limits: { cpu: '0.2', memory: '128Mi' },
  requests: { cpu: '0.2', memory: '128Mi' },
};
​
local container = K.Container.
  withName(labels.project).
  withImage('gcr.io/docker-images-180022/apps/new-app@sha256:abc123').
  withResources(containerResources);
​
local podTemplate = K.PodTemplate.
  withContainers([container]).
  withEnvFromMap({ CLUSTER: 'pod998' }).
  withMetadata({ labels: { name: labels.project } }); 
```
Finally, we create our DaemonSet function. In your function, change the value of `kind` to the workload object you are implementing and you can change value of the `apiVersion` if needed. Refer to the workload documentation for the fields requires for `spec`. DaemonSet manifest requires `selector` and `template` as fields for `spec`. 
```
local DaemonSet(labels, container) = {
  apiVersion: 'apps/v1',
  kind: 'DaemonSet',
  metadata: {
    lables: labels,
  },
  spec: {
    selector: { matchLabels: { name: labels.project } },
    template: podTemplate,
  },
};
​
local d = DaemonSet(labels, container);
​
std.manifestYamlDoc(d)
```
​
### Generate and verify your output
```
jsonnet -J vendor -S DaemonSet.jsonnet -o daemonset.yml
kubeval daemonset.yml 
```
Here is the expected output generated by Jsonnet. 
```
"apiVersion": "apps/v1"
"kind": "DaemonSet"
"metadata":
  "lables":
    "project": "test-project"
    "role": "daemon"
    "team": "pc-team"
"spec":
  "selector":
    "matchLabels":
      "name": "test-project"
  "template":
    "metadata":
      "labels":
        "name": "test-project"
    "spec":
      "containers":
      - "env":
        - "name": "CLUSTER"
          "value": "pod998"
        "image": "gcr.io/docker-images-180022/apps/new-app@sha256:abc123"
        "name": "test-project"
        "resources":
          "limits":
            "cpu": "0.2"
            "memory": "128Mi"
          "requests":
            "cpu": "0.2"
            "memory": "128Mi"
      "initContainers": []
      "terminationGracePeriodSeconds": 30
      "volumes": []
```
​
### Note
​
DaemonSet is a [restricted object](https://github.com/zendesk/opa-gatekeeper/blob/master/rules.md#restricted-kinds-daemonset) at Zendesk. If you try to deploy this, OPA Gatekeeper will throw an error at you. Message `#ask-compute` on Slack if you run into errors with OPA Gatekeeper.
