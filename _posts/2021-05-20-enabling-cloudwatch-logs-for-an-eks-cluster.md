---
layout: post
title:  "Enabling CloudWatch logs for an EKS Cluster"
date:   2021-05-20 20:52:19
categories: [aws]
comments: true
excerpt: "This post describes how to configure an EKS cluster to send logs to CloudWatch Logs."
---
Introduction
------------

This post describes how to configure an EKS cluster to send logs to CloudWatch Logs.

Pushing logs to Cloudwatch from pods is a one-time task. Once set up, logs from every pod running on the cluster should automatically be sent to CloudWatch. Meaning the steps laid out here should ideally be only done once during the lifetime of a cluster.

How exactly are logs sent from a Kubernetes pod to CloudWatch?
--------------------------------------------------------------

[Fluent Bit](https://fluentbit.io/) and [Fluentd](https://www.fluentd.org/) are 2 options to select from when choosing what tool to use for sending logs to CloudWatch from an EKS Cluster. They work similarly but have subtle differences. AWS recommends Fluent Bit for 2 reasons:

*   Fluent Bit has a smaller resource footprint and is more resource-efficient with memory and CPU usage than FluentD

*   The Fluent Bit image is developed and maintained by AWS. This gives AWS the ability to adopt new Fluent Bit image features and respond to issues much quicker.


Let's follow AWS’s recommendation and make use of Fluent Bit. To set up FluentBit, we install it as a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/). A DaemonSet is just a way of specifying that all (or some) Nodes in a cluster run a copy of a particular pod. As new nodes are added to the cluster, Kubernetes will automatically run the pod on those new nodes.

Since we need a way to get logs from all applications running on every node in our cluster, it makes sense to achieve that with a DaemonSet. That way we can define/create a single container pod that collects and sends logs and Kubernetes will run that pod on all nodes that will be part of our cluster.

Setting up Fluent Bit
---------------------

### Create a namespace.

The first step is to create a [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). A Kubernetes namespace is similar to how we understand namespaces in software engineering. It is simply a way to organize our cluster into virtual sub-clusters. By default when you run any `kubectl` command, you are working in the `default` namespace. Resources in other namespaces are “hidden” from you.

We will create a namespace called `amazon-cloudwatch` by running the command below:

`kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml`

The link above points to a simple YAML that defines the namespace properties.

Now that the namespace has been created, you can either switch to the new namespace else you have to append the `-n amaozon-cloudwatch` flag to every command you run. Switch to the namespace by running:

`kubectl config set-context --current --namespace=amazon-cloudwatch`

Now every command you run will be in the `amazon-cloudwatch` namespace.

### Create a ConfigMap

The next step is to create a [configmap](https://kubernetes.io/docs/concepts/configuration/configmap/). It is really what the name says. It is a map of configuration data. Pods can reference a configmaps and access its data. We can use a configmap to configure how a pod should work, similar to how we can use environment variables to change how our application works.

A Configmap can be created by several methods, but here we will create one using [literal values](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values).

Run the following command to create a ConfigMap named `fluent-bit-cluster-info` with the cluster name and the region to send logs to. Replace `cluster-name` and `cluster-region` with the cluster's name and region.

```
ClusterName=cluster-name
RegionName=cluster-region
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
\[\[ ${FluentBitReadFromHead} = 'On' \]\] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
\[\[ -z ${FluentBitHttpPort} \]\] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
kubectl create configmap fluent-bit-cluster-info \\
--from-literal=cluster.name=${ClusterName} \\
--from-literal=http.server=${FluentBitHttpServer} \\
--from-literal=http.port=${FluentBitHttpPort} \\
--from-literal=read.head=${FluentBitReadFromHead} \\
--from-literal=read.tail=${FluentBitReadFromTail} \\
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch
```

### Create the DaemonSet

At this point, we can now create our DaemonSet. Run the following command:

`kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml`

The URL points to a Kubernetes YAML file that defines a few resources. Let’s briefly talk about each resource:

*   [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/): A ServiceAccount provides an identity for processes that run in a pod. We will use it here to give Fluent Bit pods access to all logs in the cluster.

*   [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole): Just as we have roles in classic RBAC, a ClusterRole is a container for permissions. There is also [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) that is almost the same as ClusterRole. The difference is that we use Role when we are setting permissions within a namespace, and we use ClusterRole when we set cluster-wide permissions. Since we need permission to get logs from the entire cluster ClusterRole is used. The Cluster role defined by the YAML gives permission to all logs in the cluster

*   [ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding): Think of role binding as a mapping of a role to a user(s) called subjects. We can have RoleBinding or ClusterRoleBinding based on the same reasons explained above. The cluster role binding defined by the YAML file binds the ClusterRole to the ServiceAccount. This automatically means any pod using the service account can access all logs in the cluster.

*   ConfigMap: We already discussed what a configmap is. The configmap defined here contains all the data/settings needed to configure how Fluent Bit works

*   DaemonSet: Finally the DaemonSet is defined. It states where to get the FluentBit container image (`amazon/aws-for-fluent-bit:2.10.0`) it also defines some environment variables amongst other things.


Run `kubectl get pods`. You should have `x` number of fluent bit pods running, depending on how many nodes the cluster has.

### **Verify the Fluent Bit setup**

1.  Open the CloudWatch console at [https://console.aws.amazon.com/cloudwatch/](https://console.aws.amazon.com/cloudwatch/).

2.  In the navigation pane, choose **Logs**.

3.  Make sure that you're in the Region where you deployed Fluent Bit.

4.  Check the list of log groups in the Region. You should see the following:

    *   `/aws/containerinsights/Cluster_Name/application`

    *   `/aws/containerinsights/Cluster_Name/host`

    *   `/aws/containerinsights/Cluster_Name/dataplane`

5.  Navigate to one of these log groups and check the **Last Event Time** for the log streams. If it is recent relative to when you deployed Fluent Bit, the setup is verified.

    There might be a slight delay in creating the `/dataplane` log group. This is normal as these log groups only get created when Fluent Bit starts sending logs for that log group.


### Conclusion

That summarizes the steps to take in order to have the logs from pods in a cluster show up in CloudWatch. There are a few gotchas along the way. Be sure to check the Troubleshooting section in case something does not work as it should. Also, [See here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-EKS-logs.html) for more in-depth information about sending Kubernetes cluster logs to CloudWatch.

Troubleshooting.
----------------

#### No logs sent to CloudWatch, logs show an error about credentials.

Now, this is one error I faced. Everything was set up correctly but still, no logs were sent to CloudWatch. On checking the logs from one of the Fluent Bit pods, I discovered there was an AWS credentials error.

Remember what we said about ServiceAccounts? The ServiceAccount is currently used to give all pods in our DaemonSet access to logs in our cluster. The problem is these pods don’t have a way of accessing AWS because they have no security Credentials. To fix this we need to give our ServiceAccount access to AWS. That way the pods can also have access to AWS

I will write about this in more detail in a future post. For now see [here](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) on how to fix this. Also, ensure to go by the **Least privilege** principle. Don’t give a ServiceAccount more permission than what it really needs. For example, for this specific situation, we need to only give the pods access to CloudWatch. Nothing more.
