# schedule-vms-with-kubevirt"
[Kubevirt](https://kubevirt.io/) opens a wide verity of options to bring up virtual machines on Kubernetes. This example demonstrates how to use KubeVirt to deploy a set of VMs across nodes. If you are on a cloud provider, you may want to run the VMs only on certain time windows to save costs. We can make use of Kubernetes `CronJobs` to schedules VMs to run during office hours and shut them down automatically after hours. For example, shutdown all the vm in the `dev` namespace after 6PM and bring them back up at 8AM next day.

We will be using KubeVirt [VMPools](https://kubevirt.io/user-guide/virtual_machines/pool/) with 3 replicas of Nginx instances. We will then define a `CronJob` that runs at 6PM daily to stop the `VM`s and another one that runs at 8AM to recreate it.

## Pre-requisites
You should have a Kubernetes cluster with cluster admin access, especially apiserver must have `--allow-privileged=true`. KubeVirt should also be installed and configured on the cluster. You may find the installation steps [here](https://kubevirt.io/user-guide/operations/installation/).


## Cloud-init configs 
We will define a `Secret`, [cloud-init-with-nginx-secret.yaml](cloud-init-with-nginx-secret.yaml) to customize the VM at boot time. This config will install nginx and write a message to identify the VM. More details about cloud-init [here](https://cloudinit.readthedocs.io/en/latest/) and [here](https://kubevirt.io/user-guide/virtual_machines/startup_scripts/#cloud-init-examples) 

Create a namespace called `dev`, we will be operating from this namespace. 
```
k create ns dev
```

Apply the secret:
```
k apply -f cloud-init-with-nginx-secret.yaml
```

## VirtualMachine Pools
`VirtualMachinePool` is a CRD from KubeVirt. Let's use this manifest [vm-pool-nginx.yaml](vm-pool-nginx.yaml) to create 3 replicas of VMs with the cloud-init config.

```
k apply -f vmpool-nginx.yaml
```
As mentioned in the VM pool manifest, it's downloading the Ubuntu image from https://cloud-images.ubuntu.com/jammy/20240403/jammy-server-cloudimg-amd64.img and store it as a  data volume (a kubevirt abstraction of PVC) to attach to the VMs. 

To check the status of related resources, 
```
k get vmpool,vm,vmi,dv
NAME                                               DESIRED   CURRENT   READY   AGE
virtualmachinepool.pool.kubevirt.io/vmpool-nginx   3         3         3       5m

NAME                                        AGE   STATUS    READY
virtualmachine.kubevirt.io/vmpool-nginx-0   5m   Running   True
virtualmachine.kubevirt.io/vmpool-nginx-1   5m   Running   True
virtualmachine.kubevirt.io/vmpool-nginx-2   5m   Running   True

NAME                                                AGE   PHASE     IP            NODENAME      READY
virtualmachineinstance.kubevirt.io/vmpool-nginx-0   4m   Running   10.42.2.250   kubevirt-02   True
virtualmachineinstance.kubevirt.io/vmpool-nginx-1   4m   Running   10.42.0.70    kubevirt-01   True
virtualmachineinstance.kubevirt.io/vmpool-nginx-2   4m   Running   10.42.1.172   kubevirt-03   True

NAME                                        PHASE       PROGRESS   RESTARTS   AGE
datavolume.cdi.kubevirt.io/vmpool-nginx-0   Succeeded   100.0%     1          4m
datavolume.cdi.kubevirt.io/vmpool-nginx-1   Succeeded   100.0%     1          4m
datavolume.cdi.kubevirt.io/vmpool-nginx-2   Succeeded   100.0%     1          4m
```
Notice that the VirtualMachineInstance (VMI)s are spred across nodes as mentioned in the VM pool manifest. Dont forget to check out the `podAntiAffinity` rule mentioned in the yaml.

Now, ssh to one of the vm using `virtctl` and check if Nginx is working as expected,

```
virtctl ssh ubuntu@vmpool-nginx-0

# Verify the nginx welcome page we set in the cloud-init config
ubuntu@vmpool-nginx-0:~$ curl localhost
Hello KubeVirts!!, I am being served from vmpool-nginx-0

```

 ## Accessing Kubernetes objects from within a Pod
 A question would arise in the mind on why the rolebinding is needed? Before that, we need to understand how does a pod get access to `kube-api`. Each pod gets a service account (usually the default service account from the namespace) token mounted which has certain `RBAC` permissions. Inside the pod, token is mounted under `/var/run/secrets/kubernetes.io/serviceaccount`. `Kubectl` and `virtctl` cli can use this token to talk to API server. More details regarding how to access Kubernetes API server from a Pod is [explained here ](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/). So we need to ensure this service account has the required permissions to manage KubeVirt resources like VMs.


Some pointers to consider:
 - A container image with kubectl and virtctl installed, using the [image from here](https://github.com/govindkailas/kubectl-virtctl/pkgs/container/kubectl-virtctl)
 - Service account with access to KubeVirt CRDs and [rolebinding](kubevirt-rolebinding.yaml)

KubeVirt already has `ClusterRole` called `kubevirt.io:edit` that allows managing VM related objects. More [details here](https://kubevirt.io/user-guide/operations/authorization/#default-edit-role). We can bind this role to the default service account in the current namespace:

Apply the `RoleBinding` for the `default` service account:
```
k apply -f kubevirt-rolebinding.yaml
```
If you like handling VMs from a different namespace, consider creating a `ClusterRoleBinding`. 

## Define a CronJob to shutdown VMs
So far so good. Now let's apply the `CronJob` [stopvm-cronjob.yaml](stopvm-cronjob.yaml)
```
k apply -f stopvm-cronjob.yaml
```

To check the newly created `CronJob`:
```
k get cj
NAME      SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stop-vm   0 18 * * 1-5   False     0        <none>          8s
```

Let's test the CronJob by manually triggering a `job`,
 ```
 kubectl create job --from=cronjob/stop-vm stop-vm-manual
 ```

Verify the job logs and VM status,
```
k logs -f stop-vm-manual-k9t-dk7x5 ## use the pod name created by the job
VM vmpool-nginx-0 was scheduled to stop
VM vmpool-nginx-1 was scheduled to stop
VM vmpool-nginx-2 was scheduled to stop

# Check VM status
k get vm
NAME             AGE   STATUS    READY
vmpool-nginx-0   1h    Stopped   False
vmpool-nginx-1   1h    Stopped   False
vmpool-nginx-2   1h    Stopped   False
```
VMs status has been changed to Stopped !!

## CronJob to start VMs
Apply the `CronJob` [startvm-cronjob.yaml](startvm-cronjob.yaml) to start the VMs at 8:00 AM every weekday:
```
k apply -f startvm-cronjob.yaml
```

To trigger the CronJob manually, 
```
k create job --from=cronjob/start-vm start-vm-manual

# check the logs from the start-vm pod created by the job
k logs -f start-vm-manual-gcs9b 
VM vmpool-nginx-0 was scheduled to start
VM vmpool-nginx-1 was scheduled to start
VM vmpool-nginx-2 was scheduled to start

# Check VM status
k get vm
NAME             AGE   STATUS    READY
vmpool-nginx-0   1h    Running   True
vmpool-nginx-1   1h    Running   True
vmpool-nginx-2   1h    Running   True
``` 

*If you have any questions, suggestions, please open an issue.*