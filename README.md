# virtual-clustervcluster connect vcluster-1 -n host-namespace-1
curl -s -L "https://github.com/loft-sh/vcluster/releases/latest" | sed -nE 's!.*"([^"]*vcluster-linux-arm64)".*!https://github.com\1!p' | xargs -n 1 curl -L -o vcluster && chmod +x vcluster;
sudo mv vcluster /usr/local/bin; 
# vcluster-1.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vcluster-1
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: vcluster-1
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "services", "services/proxy", "pods", "pods/proxy", "pods/attach", "pods/portforward", "pods/exec", "pods/log", "events", "endpoints", "persistentvolumeclaims"]
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["statefulsets"]
    verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: vcluster-1
subjects:
  - kind: ServiceAccount
    name: vcluster-1
roleRef:
  kind: Role
  name: vcluster-1
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Service
metadata:
  name: vcluster-1
spec:
  type: ClusterIP
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
  selector:
    app: vcluster-1
---
apiVersion: v1
kind: Service
metadata:
  name: vcluster-1-headless
spec:
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
  clusterIP: None
  selector:
    app: vcluster-1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vcluster-1
  labels:
    app: vcluster-1
spec:
  serviceName: vcluster-1-headless
  replicas: 1
  selector:
    matchLabels:
      app: vcluster-1
  template:
    metadata:
      labels:
        app: vcluster-1
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: vcluster-1
      containers:
      - image: rancher/k3s:v1.19.5-k3s2
        name: virtual-cluster
        command:
          - "/bin/k3s"
        args:
          - "server"
          - "--write-kubeconfig=/k3s-config/kube-config.yaml"
          - "--data-dir=/data"
          - "--disable=traefik,servicelb,metrics-server,local-storage"
          - "--disable-network-policy"
          - "--disable-agent"
          - "--disable-scheduler"
          - "--disable-cloud-controller"
          - "--flannel-backend=none"
          - "--kube-controller-manager-arg=controllers=*,-nodeipam,-nodelifecycle,-persistentvolume-binder,-attachdetach,-persistentvolume-expander,-cloud-node-lifecycle"  
          - "--service-cidr=10.96.0.0/12"  # This has to be the service CIDR of your main cluster's service CIDR
        volumeMounts:
          - mountPath: /data
            name: data
      - name: syncer
        image: "loftsh/vcluster:0.3.0"
        args:
          - --service-name=vcluster-1
          - --suffix=vcluster-1
          - --owning-statefulset=vcluster-1
          - --out-kube-config-secret=vcluster-1
        volumeMounts:
          - mountPath: /data
            name: data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 5Gi
