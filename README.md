Kubernetes SecurityContext

# Introduction
In Kubernetes, SecurityContext is used to configure access control settings for a process to Container or Pod. We can limit the processes doing certain things on Container or Pod.

In this article, we will see how to set the security context at container and pod level.

### Pod with default security:
We will create a Pod with default security as a base Pod so that we can  compare it later.

Let us create an alpine container with default security:
```
kubectl run pod-default-security --image alpine --restart Never -- /bin/sleep 999999
```
Once the pod is created, check the UserId on the container as below:
```
kubectl exec -it pod-default-security id
```
