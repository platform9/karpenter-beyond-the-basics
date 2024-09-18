# Artifacts and instructions for Karpenter Beyond the Basics labs

### Cluster setup:

* CAS and Karpenter pre-installed
* one managed node group with no taints, not managed by CAS
* one managed node group with a "cas" taint and labels, managed by CAS
* EMP pre-installed with one EVM running on one bare-metal node

## Lab 1: Environment setup and Karpenter refresher

### Lab materials; AWS CLI and kubectl setup

Before you begin on the labs, you should make sure you have the necessary files to complete the lab activities, and the AWS CLI and kubectl are installed and configured.

First, you need to download this repository itself.  If you have the `git` source control tool installed, you can clone the repo directly:

```
git clone https://github.com/platform9/karpenter-beyond-basics-artifacts
```

Otherwise, you can download the files as a zip file and extract them.

Once you have the repository files cloned or extracted, open a command window or terminal and go to the directory the files are in.  All commands to complete the labs will be run from here.

* For help installing or configuring the AWS CLI, see the [AWS Getting Started guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions).
* All clusters for this lab are located in the us-east-1 region.

Set up the AWS CLI with the credentials you are provided by your instructor.  It's a good idea to set this up as a separate profile -- it will be easier to delete later, and won't interfere with any other AWS credentials you have configured.  For assistance with this, see the AWS [guide on configuring credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-user.html#cli-authentication-user-configure-wizard) (use the second example, that sets up a named profile called `userprod`).

Once you have the AWS CLI set up, you should be able to run this command successfully:

`aws eks update-kubeconfig --region us-east-1 --kubeconfig karpenter-lab --name [your cluster name]`

This will create a kubeconfig file named `karpenter-lab` in the main lab materials directory.  Set the KUBECONFIG environment variable to `karpenter-lab` to enable `kubectl` to use this file for cluster access.

* For help installing kubectl, see the Kubernetes project's [guide to installing Kubernetes tools](https://kubernetes.io/docs/tasks/tools/#kubectl).

If kubectl is set up correctly, you should be able to run this command successfully and see a list of pods:

```
kubectl get po -A
```

### Testing Cluster Autoscaler and Karpenter

1. Deploy a sample workload with a toleration of the CAS taint and a nodeSelector that selects the CAS nodes.

```
kubectl apply -f lab-1/workload-cas.yaml
```

2. Observe the type of node provisioned, the time to provision, and the utilization on the node.

```
kubectl get nodes -l autoscaler=cas
```

`kubectl describe node [node name]`

3. Deploy a Karpenter EC2NodeClass and a NodePool that will provision nodes with a specific taint and labels.
```
kubectl apply -k lab-1/ec2nodeclass
kubectl apply -f lab-1/nodepool.yaml
```
4. Deploy a workload similar to the first sample workload but with toleration and nodeSelector that will cause Karpenter to scale up the NodePool.

```
kubectl apply -f lab-1/workload-karpenter.yaml
```

5. Observe the type of node provisioned, the time to provision, and the utilization on the node.

```
kubectl get nodes -l autoscaler=karpenter
```

`kubectl describe node [node name]`

6. Delete both workloads and the Karpenter NodePool.

```
kubectl delete -f lab-1/workload-cas.yaml
kubectl delete -f lab-1/workload-karpenter.yaml
kubectl delete -f lab-1/nodepool.yaml
```

## Lab 2: Disruption and consolidation

### Consolidation policy: WhenEmpty

1. Create a new NodePool like the NodePool from the previous lab, but with a `WhenEmpty` consolidation policy.

```
kubectl apply -f lab-2/nodepool-consolidation-whenempty.yaml
```

2. Deploy a two-replica primary workload with toleration and nodeSelector corresponding to the previous step, with a topology spread constraint that forces two nodes to be created but that does not fill all the resources on the new nodes.

```
kubectl apply -f lab-2/workload-base.yaml
```

3. Once all the topology-constrained pods are running, deploy a two-replica secondary workload with the same toleration and nodeSelector, sized to fit one pod in the unused resources on each of the two new nodes.

```
kubectl get pods -n karpenter
```
```
kubectl apply -f lab-2/workload-secondary.yaml
```

4. Delete the topology-constrained workload.  Observe that nodes do not consolidate.  Review the logs of the Karpenter controller pod to see the log entries related to the disruption calculation.

```
kubectl delete -f lab-2/workload-base.yaml
```
```
kubectl logs -l app.kubernetes.io/name=karpenter -n karpenter
```

5. Scale the secondary workload down to one replica and observe consolidation of the now-empty node.

```
kubectl scale deployment -n karpenter --replicas 1 lab-2-karpenter-workload-secondary
```
```
kubectl get nodes -l autoscaler=karpenter -w
```

6. Delete the remaining workload and the NodePool.

```
kubectl delete -f lab-2/workload-secondary.yaml
kubectl delete -f lab-2/nodepool-consolidation-whenempty.yaml
```

### Consolidation policy: WhenEmptyOrUnderutilized

1. Create a new NodePool like the previous one, but with the `WhenEmptyOrUnderutilized` consolidation policy.

```
kubectl create -f lab-2/nodepool-consolidation-whenemptyorunderutilized.yaml
```

2. Deploy the topology-constrained primary workload again.

```
kubectl apply -f lab-2/workload-base.yaml
```

3. Once all the base workload pods are running, deploy the secondary workload again.

```
kubectl get pods -n karpenter
```
```
kubectl apply -f lab-2/workload-secondary.yaml
```

4. Delete the topology-constrained workload.  Observe node consolidation to a single node.

```
kubectl delete -f lab-2/workload-base.yaml
```
```
kubectl get nodes -l autoscaler=karpenter -w
```

5. Delete the remaining workload and the NodePool.

```
kubectl delete -f lab-2/workload-secondary.yaml
kubectl delete -f lab-2/nodepool-consolidation-whenemptyorunderutilized.yaml
```

### ConsolidateAfter

1. Create a new NodePool like the previous one with the `WhenEmptyOrUnderutilized` consolidation policy and `consolidateAfter` set to 30 minutes.

```
kubectl create -f lab-2/nodepool-consolidateafter-30m.yaml
```

2. Deploy the topology-constrained primary workload again.

```
kubectl apply -f lab-2/workload-base.yaml
```

3. Once all the base workload pods are running, deploy the secondary workload again.

```
kubectl get pods -n karpenter
```
```
kubectl apply -f lab-2/workload-secondary.yaml
```

4. Delete the topology-constrained workload.  Observe the delay before node consolidation to a single node (there's no need to wait the full 30 minutes, the fact that ).

```
kubectl delete -f lab-2/workload-base.yaml
```
```
kubectl get nodes -l autoscaler=karpenter -w
```

5. Delete the remaining workload and the NodePool.

```
kubectl delete -f lab-2/workload-secondary.yaml
kubectl delete -f lab-2/nodepool-consolidateafter-30m.yaml
```

### Expiration

1. Create a new NodePool like the previous one with `expireAfter` set to 5 minutes.

```
kubectl apply -f lab-2/nodepool-expireafter-5m.yaml
```

2. Deploy the topology-constrained primary workload again.

```
kubectl apply -f lab-2/workload-base.yaml
```

3. Watch the list of nodes and observe when the nodes expire and are reprovisioned.

```
kubectl get nodes -l autoscaler=karpenter -w
```

4. Do not delete the workload or the NodePool yet.

### do-not-disrupt and involuntary disruption

1. Deploy a secondary workload similar to the previous instance, but with the `do-not-disrupt` annotation.

```
kubectl apply -f lab-2/workload-secondary-donotdisrupt.yaml
```

2. Delete the topology-constrained workload.  Observe that Karpenter does not consolidate these nodes even after expiration.

```
kubectl delete -f lab-2/workload-base.yaml
```
```
kubectl get nodes -l autoscaler=karpenter -w
```

Review the logs of the Karpenter controller pod to see the log entries related to the disruption calculation.

```
kubectl logs -l app.kubernetes.io/name=karpenter -n karpenter
```

   * You may notice that Karpenter keeps provisioning and disrupting an empty node -- this is because Karpenter tries to maintain resource headroom for pods on expired nodes even if the pods cannot be disrupted.

3. Delete the remaining workload and the NodePool.

```
kubectl delete -f lab-2/workload-secondary-donotdisrupt.yaml
kubectl delete -f lab-2/nodepool-expireafter-5m.yaml
```

## Lab 3: Karpenter and workload matching

### Matching nodes to workload architecture requirements

1. Recreate the NodePool from lab 1.

```
kubectl apply -f lab-1/nodepool.yaml
```

2. Deploy the sample Karpenter workload from lab 1 again.

```
kubectl apply -f lab-1/workload-karpenter.yaml
```

3. Observe the CPU architecture of the nodes provisioned (you may need to repeat the command below if you run it before the nodes have been created).

```
kubectl get no -o jsonpath='{range .items[*]}{.metadata.labels.kubernetes\.io/hostname}{"="}{.metadata.labels.node\.kubernetes\.io/instance-type}{", arch="}{.metadata.labels.kubernetes\.io/arch}{", capacity="}{.metadata.labels.karpenter\.sh/capacity-type}{"\n"}{end}'
```

4. Delete the sample workload and deploy a new version of it that uses an ARM-based container image.

```
kubectl delete -f lab-1/workload-karpenter.yaml
kubectl apply -f lab-3/workload-arm.yaml
```

5. Observe the CPU architecture of the nodes provisioned.

```
kubectl get no -o jsonpath='{range .items[*]}{.metadata.labels.kubernetes\.io/hostname}{"="}{.metadata.labels.node\.kubernetes\.io/instance-type}{", arch="}{.metadata.labels.kubernetes\.io/arch}{", capacity="}{.metadata.labels.karpenter\.sh/capacity-type}{"\n"}{end}'
```

6. Do not delete the test workload or the NodePool yet.

### Matching nodes to capacity type requirements

1. Observe the capacity type of the node provisioned for the workload in the previous exercise is a spot instance.

2. Delete the workload.

```
kubectl delete -f lab-3/workload-arm.yaml
```

3. Deploy a new workload like the previous one but with a nodeSelector that requires on-demand instances.

```
kubectl apply -f lab-3/workload-ondemand.yaml
```

4. Observe that Karpenter provisions an on-demand instance to schedule the new workload.

```
kubectl get no -o jsonpath='{range .items[*]}{.metadata.labels.kubernetes\.io/hostname}{"="}{.metadata.labels.node\.kubernetes\.io/instance-type}{", arch="}{.metadata.labels.kubernetes\.io/arch}{", capacity="}{.metadata.labels.karpenter\.sh/capacity-type}{"\n"}{end}'
```

5. Delete the workload and the NodePool.

```
kubectl delete -f lab-3/workload-ondemand.yaml
kubectl delete -f lab-1/nodepool.yaml
```

6. Create a NodePool like the one from lab 1 but with capacity type constrained to spot instances only.

```
kubectl apply -f lab-3/nodepool-spot.yaml
```

7. Recreate the on-demand workload and observe it does not schedule yet because no NodePool can satisfy its requirements.

```
kubectl apply -f lab-3/workload-ondemand.yaml
```
```
kubectl get pods -n karpenter
```

8. Recreate the NodePool from lab 1.

```
kubectl create -f lab-1/nodepool.yaml
```

9. Observe that the on-demand workload can now schedule again.

```
kubectl get pods -n karpenter
```

10. Delete the workload and both NodePools.

```
kubectl delete -f lab-3/workload-ondemand.yaml
kubectl delete -f lab-3/nodepool-spot.yaml
kubectl delete -f lab-1/nodepool.yaml
```

## Lab 4: EMP augmentation

1. Create a NodePool like the one from lab 1, but with the instance family restricted to `c5n` and the capacity type restricted to on-demand.

```
kubectl create -f lab-4/nodepool-ondemand-c5n.yaml
```

2. Deploy the lab 2 topology-constrained workload again.

```
kubectl apply -f lab-2/workload-base.yaml
```

3. Scale the workload to 32 replicas.  Observe the size of instances provisioned.  Note the total hourly cost based on the [EC2 On-Demand Instance Pricing](https://aws.amazon.com/ec2/pricing/on-demand/) page (note: the region is us-east-1).

```
kubectl scale deployment lab-2-karpenter-workload-base -n karpenter --replicas 32
```
```
kubectl get no -o jsonpath='{range .items[*]}{.metadata.labels.kubernetes\.io/hostname}{"="}{.metadata.labels.node\.kubernetes\.io/instance-type}{", arch="}{.metadata.labels.kubernetes\.io/arch}{", capacity="}{.metadata.labels.karpenter\.sh/capacity-type}{"\n"}{end}'
```

4. Delete the workload and the NodePool.

```
kubectl delete -f lab-2/workload-base.yaml
kubectl delete -f lab-4/nodepool-ondemand-c5n.yaml
```

5. Your cluster has Platform9 Elastic Machine Pool installed for you.  One of your instructors will share the credentials for your cluster with you so you can log in to [the AWS console](https://sales-opex-emp-karpenter-demo.signin.aws.amazon.com/console).  Go to the [Elastic Kubernetes Service page](https://us-east-1.console.aws.amazon.com/eks/home?region=us-east-1#/clusters) and open the info page for your EKS cluster.

6. In the [list of running instances](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running) on the EC2 service page, you will be able to see a single `c5n.metal` instance that is currently provisioned to support your cluster via EMP.  Look up its hourly cost on the [pricing](https://aws.amazon.com/ec2/pricing/on-demand/) page.

7. Deploy a workload like the previous one, but with a toleration and nodeSelector that cause it to run on EMP EVMs.

```
kubectl apply -f lab-4/workload-emp.yaml
```

8. Scale the workload to 32 replicas.  These will all run on EVMs built on AWS bare-metal.  All 16 should run on the single bare-metal node you already have provisioned.

```
kubectl scale deployment lab-4-evm-workload -n karpenter --replicas 32
```

9. What is the difference in hourly cost of the provisioned bare-metal node relative to the hourly cost of the equivalent in `c5n` on-demand instances that are necessary to run the same 16 replicas?