---
title: Implementation on sample application
sidebar_position: 50
weight: 5
---

Create a new namespace and label it for automatic sidecar injection
```bash
kubectl create namespace testing
kubectl label namespace testing istio-injection=enabled --overwrite
```
You can then deploy a sample Hello world application:
```bash
kubectl create deploy helloworld --image=gcr.io/tetratelabs/hello-world:1.0.0 -n testing
```
Wait for the Pod to start and then Check that the secret mounted to your pod have the same issuer attributes we specified in the CAS earlier.
```bash
kubectl exec -n testing $(kubectl get pod -n testing -l app=helloworld -o jsonpath={.items..metadata.name}) \
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

Delete the testing namespace
```bash
kubectl delete namespace testing
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
