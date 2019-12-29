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
##### runAsUser
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
##### PrivilegedUser
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
kubectl exec kernel-change -- date
```
##### Restrict file access to the container file system

As the container file system is ephemeral, we can restrict the process from writing to local file system and enable it for volume as given in the below yaml. We need to set readOnlyRootFilesystem property of the securityContext.

```
apiVersion: v1
kind: Pod
metadata:
  name: read-only-pod
spec:
  containers:
  - name: read-only-pod
    image: alpine
    command: ['/bin/sleep','999999']
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: my-vol
      mountPath: /volume
      readOnly: false
  volumes:
  - name: my-vol
    emptyDir: {}
```
Once the Pod is created successfully, try to create a file in container's local file system as below and you will get an error.
```
kubectl exec read-only-pod touch /file.txt
```
Now create the file under the volume and you will be able to create it. You can also see the file if you list the contents of volume

```
kubectl exec read-only-pod touch /volume/file.txt

kubectl exec read-only-pod ls /volume
```
### SecurityContext for Pod
Its the same SecurityContext property which needs to be configured in pod level. Let us look at the different group level access to the container and Pod level as in the yaml below:

```
apiVersion: v1
kind: Pod
metadata:
  name: group-context
spec:
  securityContext:
    fsGroup: 555
    supplementalGroups: [666, 777]
  containers:
  - name: first
    image: alpine
    command: ['/bin/sleep','999999']
    securityContext:
      runAsUser: 1111
    volumeMounts:
    - name: shared-vol
      mountPath: /volume
      readOnly: false
  - name: second
    image: alpine
    command: ['/bin/sleep', '999999']
    securityContext:
      runAsUser: 2222
    volumeMounts:
    - name: shared-vol
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-vol
    emptyDir: {}
```

We have 2 containers in the spec. First container is assigned with the user 1111 and the second container is assigned with the user 2222. Also we are setting 555 as the fsGroup access in the Pod level.

Once the pod is created, open the shell to first container and execute the these commands.
```
kubectl exec -it group-context -c first sh
```
Look at the id in the shell command
```
id
```
You will see the user id as 1111 and the groups will be assigned with 555,666 and 777.
Create a file in the volume and see how the file is created. You will see that the 1111 is assigned to the user id of the file and the groupid is 555 as we specified in fsGroup.
```
echo test > /volume/sample.txt

ls -l /volume
```

Now, create a file in local file system and see how the userid and groupid are assigned.

```
echo test1 > /tmp/file.txt

ls -l /tmp
```
You will see that the file is assigned with same user id as 1111 but the groupid is root.

## Conclusion
In this article, we have seen different options for SecurityContext for both Container level and Pod Level. You can refer all the options for SecurityContext in [PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#podsecuritycontext-v1-core)
and [SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#securitycontext-v1-core)
