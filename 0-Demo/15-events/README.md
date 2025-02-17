**Install and Configure Argo Events**

Argo Events is normally installed into a namespace named `argo-events` , so let's create that:

`kubectl create ns argo-events`

Next, navigate to the [releases page](https://github.com/argoproj/argo-events/releases/tag/v1.9.5) and find the release you wish to use (the latest full release is preferred).

Scroll down to the `Installation` section and execute the kubectl commands.

Below is an example of the install command, ensure that you update the command to install the correct version number:

`kubectl apply -n argo-events -f https://github.com/argoproj/argo-events/releases/download/v1.9.5/install.yaml`

Argo Events also requires an eventbus to be created. This is a central point for all events to be sent to. The eventbus is created in the same namespace as the Argo Events controller.

There is an example available in the [argo events github repo](https://github.com/argoproj/argo-events/blob/stable/examples/eventbus/native.yaml). Execute the following command to create the eventbus:

`kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml`

We want Argo Events to trigger a workflow when a file is added to minio. In order to achieve this, we will add [a minio eventsource](https://argoproj.github.io/argo-events/eventsources/setup/minio/) which will listen for minio events.

The eventsource will require minio credentials. We will provide these in the form of a secret. View the secret with `cat /root/minio-secret.yaml` .

Deploy the secret with `kubectl apply -n argo-events -f /root/minio-secret.yaml` .

Now we can deploy our eventsource: View our eventsource with `cat /root/minio-eventsource.yaml` .

Deploy the eventsource with `kubectl apply -n argo-events -f /root/minio-eventsource.yaml` .

**What was installed?**

It may take about 1m for all deployments to become available. Let's look at what is installed while we wait.

The *Events Controller-Manager*: `kubectl -n argo-events get deploy controller-manager`

An *eventbus statefulset*: `kubectl -n argo-events get statefulsets eventbus-default-stan`

A *minio eventsource pod*: `kubectl -n argo-events get pod -l eventsource-name=minio`

**Add a Minio Sensor and Trigger**

We need to install a Sensor that is called by the EventSource. The Sensor is responsible for triggering the creation of a workflow when an event is received. Let's look at the Sensor configuration:

View our Sensor with `cat /root/minio-sensor.yaml` .

Deploy with `kubectl apply -n argo-events -f /root/minio-sensor.yaml` .

This ultimately creates a pod in our argo-events namespace that is responsible for triggering workflows when events are received.

Let's look again at our eventsource configuration: `cat /root/minio-eventsource.yaml` .

We can see that it is set to observe the minio bucket called `argoproj` . If we create a file, or delete a file in this bucket, we will trigger an event.

Let's create a file in the minio bucket: We need to port-forward the minio UI, log in and upload/delete a file from the argoproj bucket.

`kubectl -n argo port-forward --address 0.0.0.0 svc/minio 9001:9001 > /dev/null &`

Log in with the username `argoproj` and the password `sup3rs3cr3tp4ssw0rd1` , and navigate to the argoproj bucket.

Upload a file.

Let's look at the sensor pod logs:

`kubectl -n argo-events logs -l sensor-name=minio`

We can see that... it failed. You'll see logs similar to:

`{
   "level":"error",
   "ts":1685711961.5645943,
   "logger":"argo-events.sensor",
   "caller":"sensors/listener.go:356",
   "msg":"Failed to execute a trigger",
   "sensorName":"minio",
   "error":"failed to execute trigger, failed after retries: workflows.argoproj.io is forbidden: User \"system:serviceaccount:argo-events:default\" cannot create resource \"workflows\" in API group \"argoproj.io\" in the namespace \"argo\"",
   "triggerName":"minio-workflow-trigger",
   "triggeredBy":[
      "example-dep"
   ],
   "triggeredByEvents":[
      "37656535323431372d386530342d343639302d393434332d316336343039646138623631"
   ],
   "stacktrace":"github.com/argoproj/argo-events/sensors.(*SensorContext).triggerWithRateLimit\n\t/home/runner/work/argo-events/argo-events/sensors/listener.go:356"
}`


It's not all doom and gloom. We can see that something tried to happen when we uploaded our file to minio. The Sensor tried to trigger a workflow, but it failed because the default service account does not have permission to create workflows. We need to grant the default service account permission to create workflows.


**Resolve RBAC issues and re-trigger**
Kubernetes RBAC is a deep subject. Further reading can be found in the Kubernetes Documentation.

Our sensor is running using the default Service Account in the argo-events namespace. This service account does not have permission to create workflows in the argo namespace. We therefore need to give it permission to do so.

You may choose to create a completely new Service Account for this purpose. For brevity, we will just grant the default Service Account permission to create workflows. We do this using a ClusterRole, and a ClusterRoleBinding to bind the role to the Serviceaccount.

View this: cat /root/sa.yaml .

Deploy this with kubectl apply -n argo-events -f /root/sa.yaml .

Now we can attempt to re-trigger our workflow. We can do this by deleting the file we uploaded to minio. This will trigger a delete event, which will trigger our workflow.

In case you need to, port-forward the minio UI. Then log in and delete a file from the argoproj bucket.

Go back to Minio and delete the previously uploaded file.

Let's look at the sensor pod logs again.

kubectl -n argo-events logs -l sensor-name=minio

This time, the event successfully triggered our workflow:

{
   "level":"info",
   "ts":1685713173.4561043,
   "logger":"argo-events.sensor",
   "caller":"sensors/listener.go:417",
   "msg":"Successfully processed trigger 'minio-workflow-trigger'",
   "sensorName":"minio",
   "triggerName":"minio-workflow-trigger",
   "triggerType":"Kubernetes",
   "triggeredBy":[
      "example-dep"
   ],
   "triggeredByEvents":[
      "35313437336561662d633034302d346266632d613966302d613134383966313834383230"
   ]
}
We can see that the workflow was triggered, and that it completed successfully. It also contains the name of our file. In our case, we called it 'foo':

argo logs -n argo @latest