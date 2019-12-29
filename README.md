# Kubernetes SecurityContext

## Introduction
In Kubernetes, SecurityContext is used to configure access control settings for a process to Container or Pod. We can limit the processes doing certain things on Container or Pod.

In this article, we will see how to set the security context at container and pod level.

#### Pod with default security:
Create a Pod without any SecurityContext as a base Pod so that we can  compare it later.

Let us create an alpine container with default security:
```
kubectl run pod-default-security --image alpine --restart Never -- /bin/sleep 999999
```
Once the pod is created, check the UserId on the container as below:
```
kubectl exec -it pod-default-security id
```
You will see the below result. The userid, groupid is 0 which is a root user. The container gets this information from the docker file of alpine image.
```
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)

```
If you do not want to run this container as root, you need to specify securityContext.runAsUser property for your container which is explained in the next section.

### SecurityContext for Container
Create a Pod and assign the guest user(405) to the securityContext.runAsUser property. Here is the yaml file for the pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: alpine-runas-user
spec:
  containers:
  - name: alpine-runas-user
    image: alpine
    command: ['/bin/sleep', '999999']
    securityContext:
      runAsUser: 405
```
Once the pod is created, check the UserId on the container as below:
```
kubectl exec alpine-runas-user id
```
You will see that the container is assigned with the guest user id in the result as given here.

```
uid=405(guest) gid=100(users)
```
If you want to run your Pod to use the kernel features of the node, you can run the Pod with Privileged user as given in the yaml below

```
apiVersion: v1
kind: Pod
metadata:
  name: privileged-user
spec:
  containers:
  - name: priv-user-main
    image: alpine
    command: ['/bin/sleep', '999999']
    securityContext:
     privileged: true
```
Once the pod is created, list the /dev directory using pod-default-security that was created before and compare it with the privileged-user Pod.

```
kubectl exec pod-default-security ls /dev

kubectl exec privileged-user ls /dev
```

You will notice privileged-user pod displays the contents of all the nodes devices and also the Pods devices directory.

##### Setting access to container at kernel level
If you want to change the system time on container as below, you will get the operation is not permitted error.

```
kubectl exec pod-default-security -- date +%T -s "12:00:00"
```
We can solve this by adding SYS_TIME capabilities to securityContext as given in the yaml below:

```
apiVersion: v1
kind: Pod
metadata:
  name: kernel-change
spec:
  containers:
  - name: kernel-change
    image: alpine
    command: ['/bin/sleep','999999']
    securityContext:
      capabilities:
        add:
        - SYS_TIME
```
Once the pod is created, you can change the time with the command as below;
```
kubectl exec kernel-change -- date +%T -s "12:00:00"
```
