# Setup cert-manager with existing valid ssl
## 1. Prequiesite
- Kubernetes Cluster relevant with cert-manager, for more information see [here](https://cert-manager.io/docs/releases/)
- Ingress Controller, in this documentation author using envoy by contour
- Registered SSL certificate as a kubernetes secret

## 2. Create a secret for ssl certificate
From valid ssl certificate that already registered, create a secret in spesific namespace where deployment would running
```bash
kubectl create secret tls dtc-best --cert=wildcard.crt --key=wildcard.key

# or
kubectl create secret tls wildcard-cert --cert=wildcard.crt --key=wildcard.key --namespace namespaceName
```

## 3. Create ingressclass 
Envoy ingress controller by contour, not automate create an ingressclass, here the step how to make an ingress class with envoy by contour, edit the contour deployment **kubectl edit deploy contour -n projectcontour**
```bash
...
spec:
    template:
        spec:
            containers:
            - args:
                - --ingress-class-name=<nameOfIngressClass>  # add this line

# save the editing configuration, and running this command
kubectl roolout restart deploy contour -n projectcontour
```
Create and ingressclass
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: <nameOfIngressClass>
  annotations:
    # set true if waould create this ingressclass as a default ingressclass
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: projectcontour.io/ingress-controller
```

## 4. Setup Cert-manager
To install cert-manager in kubernetes cluster
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml
```
Wait for all pods running by **kubectl get pods --namespace cert-manager**
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```
Then Create 
- Cluster Issuer to enable issue accross namespace
- Deployment, Service, and Ingress
- Certficate

Cluster issuer (issuer.yaml)
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cl-issuer
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: youremail
    privateKeySecretRef:
      name: cl-issuer
    solvers:
      - http01:
          ingress:
            ingressClassName: <nameOfIngressClass>
```
Deployment, Service, and Ingress (app.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: laird
  name: demo-nginx
spec:
  ports:
  - name: http-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: demo-nginx-label
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: laird
  name: demo-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-nginx-label
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: demo-nginx-label
    spec:
      containers:
      - env:
        - name: key_a
          value: value_a
        - name: key_b
          value: value_b
        image: docker.io/nginx:stable
        imagePullPolicy: Always
        name: demo-nginx
        ports:
        - containerPort: 80
          name: http-80
          protocol: TCP
        resources:
          limits:
            cpu: 256m
            memory: 256Mi
          request:
            cpu: 128m
            memory: 128Mi
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: demo-pvc-1
      volumes:
      - name: demo-pvc-1
        persistentVolumeClaim:
          claimName: pvcName
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: laird
  name: demo-nginx
spec:
  rules:
  - host: endpointUrl
    http:
      paths:
      - backend:
          service:
            name: demo-nginx
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - endpointUrl
    secretName: dtc-best
```
Apply the configuration
```bash
kubectl apply -f app.yaml
kubectk apply -f issuer.yaml
```
Wait for the resources are running, then create the cerfificate (cert.yaml)
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cl-cert
  namespace: laird
spec:
  isCA: true
  commonName: endpointUrl
  secretName: dtc-best
  dnsNames:
  - endpointUrl 
  issuerRef:
    name: cl-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```
Describe the Cerfiticate by **kubectl describe certificate cl-cert -n laird**
```bash
Name:         cl-cert
Namespace:    laird
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2024-11-06T06:22:04Z
  Generation:          1
  Resource Version:    75012050
Spec:
  Common Name:  endpointUrl
  Dns Names:
    endpointUrl
  Is CA:  true
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       cl-issuer
  Secret Name:  dtc-best
Status:
  Conditions:
    Last Transition Time:  2024-11-06T06:22:07Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-02-04T05:23:35Z
  Not Before:              2024-11-06T05:23:36Z
  Renewal Time:            2025-01-05T05:23:35Z
  Revision:                1
Events:
  Type    Reason     Age    From                                       Message
  ----    ------     ----   ----                                       -------
  Normal  Issuing    3m42s  cert-manager-certificates-trigger          Issuing certificate as Secret was previously issued by "Issuer.cert-manager.io/"
  Normal  Reused     3m42s  cert-manager-certificates-key-manager      Reusing private key stored in existing Secret resource "dtc-best"
  Normal  Requested  3m42s  cert-manager-certificates-request-manager  Created new CertificateRequest resource "cl-cert-1"
  Normal  Issuing    3m39s  cert-manager-certificates-issuing          The certificate has been successfully issued
```
## 5. Edit the ingress
Edit the ingress configuration with **kubectl edit ing demo-nginx -n laird**
```yaml
...
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "cl-issuer"  # ----> add this line
```
Describe the ingress, and the events of output would similiarity like this
```bash
Events:
  Type    Reason             Age                 From                       Message
  ----    ------             ----                ----                       -------
  Normal  CreateCertificate  4s (x3 over 2d23h)  cert-manager-ingress-shim  Successfully created Certificate "dtc-best"
```
Run the following wget command to send a request to endpointUrl and print the response headers to STDOUT by **wget --save-headers -O- endpointUrl**
```bash
--2024-11-07 02:59:07--  http://endpointUrl/
Resolving endpointUrl (endpointUrl)... <IPLoadBalancer>
Connecting to endpointUrl (endpointUrl)|<IPLoadBalancer>|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14 [text/html]
Saving to: ‘STDOUT’
HTTP/1.1 200 OK
server: envoy
date: Thu, 07 Nov 2024 02:59:07 GMT
content-type: text/html
content-length: 14
last-modified: Tue, 10 Sep 2024 04:30:13 GMT
etag: "66dfcb55-e"
accept-ranges: bytes
x-envoy-upstream-service-time: 1


-                                          0%[                                                                                  ]       0  --.-KB/s               h
-                                        100%[=================================================================================>]      14  --.-KB/s    in 0s

2024-11-07 02:59:07 (2.40 MB/s) - written to stdout [14/14]
```
Try to make a new deployment and ingress with endpoint to endpointUrl2, without adding certificate to endpointUrl2, and run wget command **wget --save-headers -O- endpointUrl2**, output would be like :
```bash
--2024-11-07 03:08:24--  http://endpointUrl2/
Resolving endpointUrl2 (endpointUrl2)... <IPLoadBalancer>
Connecting to endpointUrl2 (endpointUrl2)|<IPLoadBalancer>|:80... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://endpointUrl2/ [following]
--2024-11-07 03:08:24--  https://endpointUrl2/
Connecting to endpointUrl2 (endpointUrl2)|<IPLoadBalancer>|:443... connected.
ERROR: cannot verify endpointUrl2's certificate, issued by ‘CN=R11,O=Let's Encrypt,C=US’:
  Unable to locally verify the issuer's authority.
To connect to endpointUrl2 insecurely, use `--no-check-certificate'.
```
