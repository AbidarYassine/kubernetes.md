# Kubernetes Commands

Helper setup to edit .yaml files with Vim:

- [VIM Setup for Yaml files](#vim-setup-for-yaml-files)

List of general purpose commands for Kubernetes management:

- [PODS](#pods)
- [Create Deployments](#create-deployments)
- [Scaling PODs](#scaling-pods)
- [POD Upgrade / History](#pod-upgrade-and-history)
- [Services](#services)
- [Volumes](#volumes)
- [Secrets](#secrets)
- [ConfigMaps](#configmaps)
- [Ingress](#ingress)
- [Horizontal Pod Autoscalers](#horizontal-pod-autoscalers)
- [Scheduler](#scheduler)
- [Taints and Tolerations](#tains_and_tolerations)
- [Troubleshooting](#troubleshooting)
- [Role Based Access Control (RBAC)](#role_based_access_control)
- [Security Contexts](#security_contexts)
- [Pod Security Policies](#pod_security_policies)
- [Network Policies](#network_policies)
- [YAML](#yaml)
- [Kubeadm](#KUBEADM)
- [ETCD](#ETCD)


## VIM Setup for Yaml files

Put the following lines in ~/.vimrc:

```
" Yaml file handling
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab
filetype plugin indent on
autocmd FileType yaml setl indentkeys-=<:>

" Copy paste with ctr+c, ctr+v, etc
:behave mswin
:set clipboard=unnamedplus
:smap <Del> <C-g>"_d
:smap <C-c> <C-g>y
:smap <C-x> <C-g>x
:imap <C-v> <Esc>pi
:smap <C-v> <C-g>p
:smap <Tab> <C-g>1> 
:smap <S-Tab> <C-g>1<
```

Keyboard hints:

- ctrl + f: auto indent line (requires INSERT mode)

## PODS

```
$ kubectl get pods
$ kubectl get pods --all-namespaces
$ kubectl get pod monkey -o wide
$ kubectl get pod monkey -o yaml
$ kubectl describe pod monkey
```

## Create Deployments

Create single deployment

```
$ kubectl run monkey --image=monkey --record
```

## Scaling PODs

```bash
$ kubectl scale deployment/POD_NAME --replicas=N
```

## POD Upgrade and history

#### List history of deployments

```
$ kubectl rollout history deployment/DEPLOYMENT_NAME
```

#### Jump to specific revision

```
$ kubectl rollout undo deployment/DEPLOYMENT_NAME --to-revision=N
```

## Services

List services

```
$ kubectl get services
```

Expose PODs as services (creates endpoints)

```
$ kubectl expose deployment/monkey --port=2001 --type=NodePort
```

## Volumes

Lits Persistent Volumes and Persistent Volumes Claims:

```
$ kubectl get pv
$ kubectl get pvc
```

## Secrets

```
$ kubectl get secrets
$ kubectl create secret generic --help
$ kubectl create secret generic mysql --from-literal=password=root
$ kubectl get secrets mysql -o yaml
```
## ConfigMaps

```
$ kubectl create configmap foobar --from-file=config.js
$ kubectl get configmap foobar -o yaml
```

## DNS

List DNS-PODs:

```
$ kubectl get pods --all-namespaces |grep dns
```

Check DNS for pod nginx (assuming a busybox POD/container is running)

```
$ kubectl exec -ti busybox -- nslookup nginx
```

> Note: kube-proxy running in the worker nodes manage services and set iptables rules to direct traffic.

## Ingress

Commands to manage Ingress for ClusterIP service type:

```
$ kubectl get ingress
$ kubectl expose deployment ghost --port=2368
```

Spec for ingress:

- [backend](https://github.com/kubernetes/ingress/tree/master/examples/deployment/nginx)
 
## Horizontal Pod Autoscaler

When heapster runs:

```
$ kubectl get hpa
$ kubectl autoscale --help
```

## DaemonSets

```
$ kubectl get daemonsets
$ kubectl get ds
```

## Scheduler

NodeSelector based policy:

```
$ kubectl label node minikube foo=bar
```

Node Binding through API Server:

```
$ kubectl proxy 
$ curl -H "Content-Type: application/json" -X POST --data @binding.json http://localhost:8001/api/v1/namespaces/default/pods/foobar-sched/binding
```

## Tains and Tolerations

```
$ kubectl taint node master foo=bar:NoSchedule
```

## Troubleshooting

```
$ kubectl describe
$ kubectl logs
$ kubectl exec
$ kubectl get nodes --show-labels
$ kubectl get events
```

Docs Cluster: 
- https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/
- https://github.com/kubernetes/kubernetes/wiki/Debugging-FAQ

## Role Based Access Control

- Role
- ClusterRule
- Binding
- ClusterRoleBinding

```
$ kubectl create role fluent-reader --verb=get --verb=list --verb=watch --resource=pods
$ kubectl create rolebinding foo --role=fluent-reader --user=minikube
$ kubectl get rolebinding foo -o yaml
```

## Security Contexts

Docs: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

- spec
 - securityCOntext
   - runAsNonRoot: true
   
## Pod Security Policies

Docs: https://github.com/kubernetes/kubernetes/blob/master/examples/podsecuritypolicy/rbac/README.md

## Network Policies

Network isolation at Pod level by using annotations

```
$ kubectl annotate ns <namespace> "net.beta.kubernetes.io/network-policy={\"ingress\": {\"isolation\": \"DefaultDeny\"}}"
```

More about Network Policies as a resource: 

https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/

## YAML

### Single Container
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
    - image: busybox
      name: busybox
      command:
        - sleep
        - "3600"
```

### Double container in a pod
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

### Single init container
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

### Persistent Volume
```YAML
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv0001
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/Users/nahi/kubernetes/pv"
```

### Namespace
```YAML
kind: Namespace
metadata:
  name: myns
```
### Jobs
```YAML
apiVersion: batch/v1
kind: Job
metadata:
  name: busybox
spec:
  template:
    metadata:
      name: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["echo",  "hello", "world"]
      restartPolicy: Never
  completions: 10
  parallelism: 2
```
  
### Busybox w/ secret
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: busybox-with-secret
spec:
  containers:
    - image: busybox-with-secret
      name: busybox
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: /busy
          name: test
      env:
        - name: MY_SECRET
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
      command:
        - sleep
        - "3600"
  volumes:
    - name: test
      secret:
        secretName: mysecret
```

### Pod w/ Namespace
```YAML
apiVersion: v1
kind: Pod
metadata:
  namespace: myns
  name: redis
spec:
  containers:
  - image: redis:3.2
    imagePullPolicy: IfNotPresent
    name: redis
  restartPolicy: Always
```
 
### DaemonSet
```YAML
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: busybox-daemon
spec:
  template:
    metadata:
      labels:
        name: busybox-daemon
    spec:
      hostNetwork: true
      hostPID: true
      containers:
        - name: busybox
          image: busybox
          command:
            - sleep
            - "3600"
          securityContext:
            privileged: true
```

### Tolerations
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  tolerations:
    - key: node-role.kubernetes.io/master
      effect: NoSchedule
  containers:
    - image: alpine
      name: alpine
      command:
        - sleep
        - "3600"
```
## ETCD
```
etcdctl
https://coreos.com/etcd/docs/latest/v2/admin_guide.html
```

## Kubeadm
```
$> kubeadm token list
$> cat /etc/kubernetes/pki/tokens.csv
```
