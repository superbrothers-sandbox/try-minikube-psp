# try-minikube-psp

Create a minikube cluster with PodSecurityPolicy.

```
$ minikube version
minikube version: v1.15.0
commit: 3e098ff146b8502f597849dfda420a2fa4fa43f0
$ minikube start --extra-config=apiserver.enable-admission-plugins=PodSecurityPolicy --addons=pod-security-policy
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.4", GitCommit:"d360454c9bcd1634cf4cc52d1867af5491dc9c5f", GitTreeState:"clean", BuildDate:"2020-11-11T13:09:17Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```

Create a nginx deployment contained a nginx container that runs as root.
```
$ kustomize version
{Version:kustomize/v3.6.1 GitCommit:c97fa946d576eb6ed559f17f2ac43b3b5a8d5dbd BuildDate:2020-05-27T20:47:35Z GoOs:darwin GoArch:amd64}
$ kustomize build ./privileged | kubectl apply -f-
```

Containers must run as non-root because PodSecurityPolicy `restricted` is set.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:restricted
subjects:
- kind: ServiceAccount
  name: nginx
```

You can see that a `nginx` Pod's status is `CreateContainerConfigError` because `nginx` image will run as root.

```
$ kubectl get po -l run-as=root
NAME                     READY   STATUS                       RESTARTS   AGE
nginx-66f4484f96-6jf9t   0/1     CreateContainerConfigError   0          2m48s
$ kubectl describe po -l run-as=root | grep -A 10 -e Events:
Events:
  Type     Reason       Age                   From               Message
  ----     ------       ----                  ----               -------
  Normal   Scheduled    3m40s                 default-scheduler  Successfully assigned default/nginx-66f4484f96-6jf9t to minikube
  Warning  FailedMount  3m39s                 kubelet            MountVolume.SetUp failed for volume "nginx-token-89psm" : failed to sync secret cache: timed out waiting for the condition
  Normal   Pulled       84s (x12 over 3m38s)  kubelet            Container image "nginx:1.18" already present on machine
  Warning  Failed       84s (x12 over 3m38s)  kubelet            Error: container has runAsNonRoot and image will run as root
```

Then create an `unprivileged-nginx` Deployment that runs as non-root (UID `101`).

```
$ kustomize build ./unprivileged | kubectl apply -f-
```

You can see that an `unprivileged-nginx` Pod is run well.

```
$ kubectl get po -l run-as=non-root
NAME                                  READY   STATUS    RESTARTS   AGE
unprivileged-nginx-77b89cb967-4mvxr   1/1     Running   0          24s
```

## References

- https://minikube.sigs.k8s.io/docs/tutorials/using_psp/
- https://kubernetes.io/docs/concepts/policy/pod-security-policy/
