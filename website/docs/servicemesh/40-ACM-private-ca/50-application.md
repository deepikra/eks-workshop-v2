---
title: Implementation on sample application
sidebar_position: 50
weight: 5
---

Let's label *ui* namespace for automatic sidecar injection
```bash
kubectl label namespace ui istio-injection=enabled --overwrite
```
You can then deploy the sample application. Let's test the certs with *ui* service. 
```bash
kubectl apply -n ui -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-v1
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
    version: ui-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ui
      app.kubernetes.io/instance: ui
      app.kubernetes.io/component: service
  template:
    metadata:
      annotations:
        prometheus.io/path: /actuator/prometheus
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/name: ui
        app.kubernetes.io/instance: ui
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
        version: ui-v1
    spec:
      serviceAccountName: ui
      securityContext:
        fsGroup: 1000
      containers:
        - name: ui
          env:
            - name: JAVA_OPTS
              value: -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
            - name: RETAIL_UI_BANNER
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          envFrom:
            - configMapRef:
                name: ui
          securityContext:
            capabilities:
              add:
              - NET_BIND_SERVICE
              drop:
              - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          image: "public.ecr.aws/aws-containers/retail-store-sample-ui:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 20
          resources:
            limits:
              memory: 1.5Gi
            requests:
              cpu: 250m
              memory: 1.5Gi
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
EOF
```
Wait for the Pod to start and then Check that the secret mounted to your pod have the same issuer attributes we specified in the CAS earlier.
```bash
kubectl exec -n ui $(kubectl get pod -n ui -l version=ui-v1 -o jsonpath={.items..metadata.name}) \
-c istio-proxy -- cat /var/run/secrets/istio/root-cert.pem | openssl x509 -text -noout
```
Output:
```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            84:46:b8:74:79:a0:fc:af:70:84:94:dc:73:93:6a:f1
    Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=CA, O=Istio, OU=abc, CN=www.example.com
        Validity
            Not Before: Jul 27 23:04:37 2022 GMT
            Not After : Jul 28 00:04:37 2032 GMT
        Subject: C=CA, O=Istio, OU=abc, CN=www.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
...
```

## Cleanup

To avoid incurring future charges on your AWS account, perform the following steps to remove the scenario.

Delete the ui namespace
```bash
kubectl delete namespace ui
```
Comment the Terraform lines listed under the section “Setting up ACM Private CA”, and disable tetrate_istio, cert-manager, and acm_privateca_issuer modules by setting those values to false, and commenting the value “aws_privateca_acmca_arn”
```bash
#K8s Add-ons
  enable_tetrate_istio                =false
  enable_cert_manager                 =false
  
  # The AWS PrivateCA Issuer plugin acts as an addon to cert-manager that 
  # signs certificate requests using ACM Private CA.
  enable_aws_privateca_issuer         =false
  
  # The ARN of the Root CA
  # aws_privateca_acmca_arn             = aws_acmpca_certificate_authority.istio.arn 
```
Then run teraform apply in the root module directory
```bash
terraform apply
```
