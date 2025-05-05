---
title: "How I deployed a Three-Tier Application using HELM in Kubernetes Cluster"
datePublished: Mon May 05 2025 05:29:11 GMT+0000 (Coordinated Universal Time)
cuid: cmaan4tu8000209lc3g3b4hqk
slug: how-i-deployed-a-three-tier-application-using-helm-in-kubernetes-cluster
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1746425495823/3122166a-c213-48d7-8fc8-43b66d0ae1f9.png
tags: kubernetes, devops, helm, three-tier-architecture

---

Managing Kubernetes deployments can often feel complex and tedious, isn’t it ? Dealing with multiple manifests (YAML files), manually applying each file ( `kubectl apply` ), updating the configuration by editing each YAML file, is definitely not what we want to deal with while deploying an application. Even for the small application, there are lots of YAMLs that we have to manage i.e.Deployments, Services, StatefulSets, ConfigMaps, Secrets, PersistentVolumeClaims, etc…

This is the place where HELM ( a package manager for Kubernetes) fits in. HELM bundles all the K8s manifests into a single package/artifact called HELM Chart which we can deploy on the cluster using a single command. Doesn’t it feel like magic ? Also, it provides a single place for managing all the configurations of K8s manifests, meaning, we can dynamically provide the values for different fields during deployment. Additionally, it allows rolling back to the previous `release` with one-liner command. So, using HELM makes K8s deployment much easier and faster.

NOTE: HELM Release is the logical representation of all the resources in cluster that has been deployed using a HELM chart.

Let’s look how I used HELM to simplify the deployment of three-tier application in Kubernetes.

## 1\. Understanding the 3 Tier Architecture

The application that we are talking about today is a simple File Sharing Application that allows the user to upload the files, generates the pin, and using that pin, anyone can download that file from anywhere on the Internet without requiring any login. Checkout the source code of application here: [https://github.com/greninja517/file-sharing-application.git](https://github.com/greninja517/file-sharing-application.git)

Also, there is an demo of application available here: [https://youtu.be/Hp-feeMzuzQ](https://youtu.be/Hp-feeMzuzQ)

i. Frontend-Tier: Used Plain JavaScript only, served the static page using nginx

ii. Backend-Tier: Java SpringBoot

iii. Database-Tier: MySQL database

**Workflow**: Whenever the upload event is triggered by a click on the webpage, frontend pod send an API request to the backend pod. Backend pod stores the file on the pod’s filesystem. Since this is temporary and gets deleted whenever the pod restarts, so this volume on pod is mounted on the node(where the pod is running) using the PersistentVolume(PV) and PersistentVolumeClaim(PVC). For local deployment, it’s super common to have PV on the node because using the Cloud Volumes isn’t free of cost. Now, let’s get back to the context, Backend then generates the 4-digit pin and also creates a database entry on the mysql pod. Each entry contains the pin associated with file location on the pod which will be used for retrieving the file when users request downloading using the PIN. For persistent storage, database pod is deployed using StatefulSet which also uses PV and PVC similar to the backend pod. All the internal routing is handled using Ingress Resource which is managed by `nginx-ingress` controller.

## 2\. Understanding the HELM Chart

HELM chart is simply a package of K8s manifests. Here’s the structure of chart looks like when we run the following command to create a helm chart:

```bash
helm create mychart
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746367176774/54448be2-fd5e-40e6-aaa6-38136c91cce0.png align="center")

i. `charts/` directory : This contains any dependencies( other HELM charts) if required for our chart.

ii. `Chart.yaml` file: This file contains the information about the chart like name, description, chart version, application version, etc. Basically, it acts as the metadata for our chart.

iii. `templates/` directory: This is the actual location where we keep all our K8s manifests either inside the nested sub-directories or at the root level.

iii. `values.yaml` file : In this file, we define all our values that we need to pass dynamically in our K8s manifests where the HELM fills in whenever we install the Chart. It’s the central location for managing all the configurations of entire Chart.

---

Let’s look at our file-sharer application helm chart instead. It would be more practical. Checkout the source code of helm chart here: [https://github.com/greninja517/file-sharer.git](https://github.com/greninja517/file-sharer.git)

As usual, `Chart.yaml` file contains the information about the file-sharer chart. There’s no `charts/` directory here as there aren’t any dependencies for file-sharer chart. In the `templates/` dir, we can see YAML files organized separately for frontend, backend and database. One thing to notice is that there are only PersistentVolumeClaims(PVCs) defined for both backend and database but no PersistentVolumes(PVs) and StorageClass(SC) are defined because while creating a HELM chart, we only create the application specific resources. Resources like PVs should be created separately either through dynamic provisioning (done by SCs) for non-local volumes or manually creating PVs for local volumes. It’s not possible to dynamically create local PVs whenever an unbounded PVC is created in the cluster.

The most important part of a HELM chart is the `values.yaml` file where we can customize the values based on our requirements. In the values.yaml file, we can see two sensitive values:

```yaml
database:
    # some other values......
    root_user: "root"
    root_password: "Root@123"
```

These type of sensitive information shouldn’t be populated in the values.yaml while creating the HELM chart. The most basic and simple approach to deal with this scenario is:

**i. Create a secrets.yaml file to store the secret values. Don’t push this file to helm chart. And, we can run:**

```bash
helm install <release-name> <chart-name> -f secrets.yaml
```

Using the -f flag, multiple yaml files can be passed for dynamic values.

**ii. Using the set flag:**

```bash
helm install <release-name> <chart-name> --set database.root_user="root" --set database.root_password="Root@123"
```

However, I have kept the secretes in the chart intentionally to let others know about what values are needed to be passed while installing the HELM chart. Just for the sake of simplicity only, these values are kept in the values.yaml file. You should **NEVER** do this.

## 3\. Installing the HELM Chart in the K8s Cluster

Before installing, there need to be some setup done beforehand.

**i.** **Setting up the Cluster locally**: Since we aren’t using any managed K8s services like GKE, EKS. So, we need to set up the local K8s cluster. I have used minikube with docker driver to set up the cluster. You can also use kind to set up the cluster as well.

— Install `minikube` from here: [https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Frpm+package](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Frpm+package)

**ii.** I**nstalling kubectl**: In order to interact with the Kubernetes API server, we need a client. And, kubectl is the most widely used one.

— Install `kubectl` from here based on your OS: [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)

**iii. Installing HELM**: We also need to need to install HELM tool for creating and installing the HELM charts.

— Install `HELM` from here based on your OS: [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

After the cluster has been set-up. For file-sharer application, we also need to install nginx-ingress controller since we are using the ingress resource in our app for internal routing. If using minikube, you can run this command to install nginx-ingress controller:

```bash
minikube addons enable ingress
# Verfiy the controller is running
kubectl get pods -n ingress-nginx
```

After the ingress controller pod’s status become running then it will watch for ingress resource and handle the routing based on the rules defined on ingress.

After this long setup has been completed, it’s finally the time to install helm chart in the cluster. First, we need to add the chart repository locally in our machine using this command.

```bash
helm repo add filesharer https://greninja517.github.io/file-sharer/
# filesharer is the local name for repo in the system
```

Verify the repo has been added on the system using:

```bash
helm repo list
# You can search for chart inside the repo
helm search repo file-sharer # can use any keyword in place of file-sharer to search
```

You might have noticed this ingress section in the values.yaml file inside the helm chart source code.

```yaml
ingress:
  # provide your domain name for the application
  domain_name : "file.cloud-ninja.xyz"
```

This is the domain name you need to set up in your system’s local dns resolver. For Linux and MacOs, there’s a file called `/etc/hosts` that acts as a local dns resolver for your computer. Similarly, for Windows, there’s file called `C:\Windows\System32\drivers\etc\hosts` that does the same thing. You need to add the below entry in this file:

```bash
<minikube-node-ip>    file.cloud-ninja.xyz
## get the minikube node IP using: 
## kubectl get nodes -o wide
```

This is because the file sharing application is designed like that. If you checkout the source code of file sharing application; Inside this file `frontend/script.js` , you will see like this in the 2nd line; `const API_URL = "`[`http://file.cloud-ninja.xyz:80`](http://file.cloud-ninja.xyz:80)`";` If different domain name is used, then ingress routing will fail and the application will crash. The reason for this value to be hardcoded is that we can’t supply environment variables’ value dynamically in the static JavaScript file. But, if you wish to use different domain name, you can change this line of code, rebuild the Docker image, push the image to your public Docker repository, and reference that repository and tag in the `values.yaml` file’s `frontend.image` object. Also, update the `ingress.domain_name` value. Then, everything will be good. You can explore whatever you want. But, for now let’s keep it simple.

As, I mentioned earlier, Helm chart doesn’t create PersistentVolume(PV) and StorageClass(SC) because these aren’t the application specific resources. So, we need to create PV and SC separately in the cluster. Go to the file-sharing-application source code from this URL: [https://github.com/greninja517/file-sharing-application.git](https://github.com/greninja517/file-sharing-application.git)

Then, You’ll see these four files;

i. `k8s-manifests/backend/backend-pv.yaml` ii. `k8s-manifests/backend/backend-sc.yaml`

iii. `k8s-manifests/mysql-db/db-pv.yaml` iv. `k8s-manifests/mysql-db/db-sc.yaml`

You need to apply these 4 files in the cluster using: `kubectl apply -f <file_name.yaml>`. And, the required PersistentVolumes(PVs) and StorageClass(SC) will be created in the cluster. Verify using this command:

```bash
kubectl get pv,sc
```

After the PVs and SC have been created, we are ready to install the helm chart. Use this command to install the helm chart in the cluster.

```bash
helm install filesharer-release filesharer/file-sharer
# you can use --namespace=<your-namespace> if you want the resources to be created in some other namespace
# --set field=<value> to override any field's value present in the values.yaml file
# --dry-run=server to check whether there will be failure or not while installing the chart.
# HELM actually validates with the K8s API server whether the applied YAML files are valid or not.
```

You’ll see like this output:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746383325899/c69ea919-3a61-454c-b84c-12b1142228b9.png align="center")

Now, you are able to access the application at `http://file.cloud-ninja.xyz:80`. BOOM !!!

All the resources will be created in the `default` namespace, unless you specified `—namespace=<your_namespace>` option while installing the HELM chart. You can verify the resources are created in the cluster using:

```bash
kubectl get all,pvc # -n <your_namespace> if installed in different namespace
```

Run: `helm list` command to get `release` information like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746421179872/91e4eb86-5dea-4119-9b1d-ba2a21f1d4c4.png align="center")

Here, the REVISION number means the version of `release` that is currently running on the cluster. It gets automatically incremented whenever you run: `helm upgrade <release-name> <chart-name>` command or you rollback to previous version of release using: `helm rollback <release-name> <revision-number-to-rollback-on>` .

Suppose, we updated our helm chart. Let’s say changing the replica count from 1 to 2 in the values.yaml file. Then, we can run:

```bash
helm upgrade filesharer-release filesharer/file-sharer 
```

And, the new version of release will be installed in the cluster. So, it’s even more easier to upgrade the application using HELM. You can verify by running: `helm list` command. We can also see the history of a release using this commad:

```bash
helm history filesharer-release
```

You’ll see output like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746421754060/4bbace3d-b2e0-4b7d-8030-c3eaedc1bc59.png align="center")

Let’s say if the latest `release` of your application that you have just deployed in the cluster crashes. Then, No Worries, you can move back to the previous stable `release` using:

```bash
helm rollback filesharer-release 1 # here 1 means the revision number of filesharer-release that you want to rollback on
```

If you want to delete all the resources created by HELM chart in the cluster. Simply, you can run :

```bash
helm uninstall filesharer-release
```

This command deletes all the resources that has been created by the HELM chart. See, how easier it is to manage the application lifecycle using HELM. With just simple one-line commands, we can control everything related to our application deployment. That’s why HELM is so popular, it makes life easier. Hehehee !

**BONUS TIP:**

Now, all the resources created by the HELM chart has been deleted. If you run: `kubectl get pv` then you’ll see something like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746413720967/8e03498c-1407-471e-8602-e0d41f5a48ac.png align="center")

Even though the `backend-pvc` has been deleted, the attached `backend-pv` is in `Released` state and still associated with that previous claim. It means, the PV is still not available for bounding by another new PVC. This is the main problem with the local PVs. K8s doen’t automatically clean up the disk and bring the local PVs to the `Available` state. So, we need to manually clean the PVs to make them available for binding with other PVCs. For this, we need to remove this `spec.claimRef` object in the definition of that PV.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746414457252/91add597-e898-49fc-b4c8-ee56c91487d3.png align="center")

Run: `kubectl edit pv <pv-name>` to obtain the YAML file for PV ,then remove this object and save the changes. After that, the PV will be in `Available` state:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746414696212/f434e854-b22a-473b-8171-579b1f940dac.png align="center")

We can also set up a `cronjob` that periodically looks up the Released PVs and cleans them up. Also, our own controller can also be deployed on the cluster that does the same thing. Anyways, now the PV is back for reuse. Hurray !!

## 4\. Wrapping Up the Things We Learned

Now, we've successfully deployed a three-tier application to Kubernetes using Helm! We started by understanding the pain points of manual manifest management and saw how Helm addresses them through packaging ( `charts/`, `templates/`), configuration (`values.yaml`), installation and many more additional tips.

Still, there is a major part left i.e. **How to publish your own HELM chart**. In the helm chart source code repo: [https://github.com/greninja517/file-sharer.git](https://github.com/greninja517/file-sharer.git) , I have covered the steps on how to publish your HELM Chart through `github-pages`. You can check that out as well.

So far, we have learned a lot about HELM but there is still many more to explore about HELM like HELM functions, Control Structures in HELM, etc. Checkout the HELM docs here: [https://helm.sh/docs/chart\_template\_guide/getting\_started/](https://helm.sh/docs/chart_template_guide/getting_started/) for more advanced HELM concepts. Keep Exploring !! Keep Learning

Connect with me on:  
🔗 **LinkedIn**: [linkedin.com/in/anjal-poudel-8053a62b8](https://linkedin.com/in/anjal-poudel-8053a62b8)