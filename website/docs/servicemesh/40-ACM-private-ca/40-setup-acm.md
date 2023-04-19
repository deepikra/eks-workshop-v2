---
title: Setting up ACM Private CA and Creating Intermediate CA
sidebar_position: 40
weight: 5
---
## Setting up ACM Private CA and Creating Intermediate CA

Add the following lines under the above lines in the root Terraform module's main.tf file. Those resources will create, configure and activate a Root CA and create the CAS.

```bash
## Setting up ACM Private CA
############################

# To create the Root CA.
resource "aws_acmpca_certificate_authority" "istio" {
  type = "ROOT"

  certificate_authority_configuration {
    key_algorithm     = "RSA_2048"
    signing_algorithm = "SHA256WITHRSA"

    subject {
      common_name = "www.example.com"
      country     = "CA"
      organization = "Istio"
      organizational_unit = "abc"
    }
  }
  
}

# To issue a certificate using ACM PCA.
resource "aws_acmpca_certificate" "istio" {
  certificate_authority_arn   = aws_acmpca_certificate_authority.istio.arn
  certificate_signing_request = aws_acmpca_certificate_authority.istio.certificate_signing_request
  signing_algorithm           = "SHA512WITHRSA"

  template_arn = "arn:aws:acm-pca:::template/RootCACertificate/V1"

  validity {
    type  = "YEARS"
    value = 10
  }
}

# Associates a certificate with an AWS Certificate Manager Private Certificate 
# Authority (ACM PCA Certificate Authority). An ACM PCA Certificate Authority 
# is unable to issue certificates until it has a certificate associated with it. 
# A root level ACM PCA Certificate Authority is able to self-sign its own root 
# certificate.
resource "aws_acmpca_certificate_authority_certificate" "istio" {
  certificate_authority_arn = aws_acmpca_certificate_authority.istio.arn

  certificate       = aws_acmpca_certificate.istio.certificate
  certificate_chain = aws_acmpca_certificate.istio.certificate_chain
}

data "aws_caller_identity" "current" {}

# To attache a resource based policy to the private CA.
resource "aws_cloudformation_stack" "istio" {
  depends_on  = [aws_acmpca_certificate_authority.istio]
  name = "istio-acm-pca-policy"
  template_body = <<STACK
{
  "Resources" : { 
    "istioAcmPcaPolicy": {
        "Type" : "AWS::ACMPCA::Permission",
        "Properties" : {
            "Actions" : ["GetCertificate", "IssueCertificate", "ListPermissions"],
            "CertificateAuthorityArn" : "${aws_acmpca_certificate_authority.istio.arn}",
            "Principal" : "acm.amazonaws.com",
            "SourceAccount" : "${data.aws_caller_identity.current.account_id}"
          }
      }
  }
}
STACK
}
```

## Generating intermediate CA from AWS ACM Private CA using Getmesh Cli
To sign the certificates of ISTIO-managed workloads, you must first create an intermediate CA from AWS ACM Private CA. The getmesh cli tool is used to create those configurations.

To do so, you must provide the SSL certificate's attributes, the ARN of the ACM Private CA (Root CA), the istiod namespace (istio-system by default), the validity days, key length, and, most importantly, override the existing cacert secret.


Add the following lines sample under the above lines in the root Terraform module's main.tf file. Those resources will install getmesh cli tool, and create this intermediate CA.

```bash
## Generating the Intermediate CA 
#################################

resource "null_resource" "install_getmesh" {
  depends_on  = [aws_cloudformation_stack.istio]
  
  provisioner "local-exec" {
    command = "curl -sL https://istio.tetratelabs.io/getmesh/install.sh | bash"
  }
}

locals {
  acmProvider                   = "aws"
  signingCa                     = aws_acmpca_certificate_authority.istio.arn
  templateArn                   = "arn:aws:acm-pca:::template/SubordinateCACertificate_PathLen0/V1"
  signingAlgorithm              = "SHA512WITHRSA"
  commonName                    = "www.abc.com"
  country                       = "CA"
  organization                  = "Istio"
  organizationalUnit            = "Security"
  email                         = "username@abc.com"
  istioCaNamespace               = "istio-system" 
  overrideExistingCaCertSecret  = true
  validityDays                  = 365
  keyLength                     = 2048
}

resource "null_resource" "acm_config" {
  depends_on = [null_resource.install_getmesh]
  
  provisioner "local-exec" {
    command = <<EOF
      getmesh gen-ca \
      -p "${local.acmProvider}" \
      --signing-ca "${local.signingCa}" \
      --template-arn "${local.templateArn}" \
      --signing-algorithm "${local.signingAlgorithm}" \
      --common-name "${local.commonName}" \
      --country "${local.country}" \
      --organization "${local.organization}" \
      --organizational-unit "${local.organizationalUnit}" \
      --email "${local.email}" --istio-ca-namespace "${local.istioCaNamespace}" \
      --override-existing-ca-cert-secret ${local.overrideExistingCaCertSecret} \
      --validity-days ${local.validityDays} \
      --key-length ${local.keyLength};
    EOF
  }
}
```
Initiate your terraform module:
```bash
terraform init
```
Check what would be deployed:
```bash
terraform plan
```
Now it’s time to deploy this terraform module after you have just adjusted it.
```bash
terraform apply --auto-approve 
```
After the deployment is done, run the following command to get connected to the cluster, replacing the region and cluster name with yours:
```bash
aws eks --region [region-name] update-kubeconfig --name [eks cluster name]        
```
Check that the cacerts come from acm-private CA and have the issuer attributes we specified in the CAS earlier.
```bash
kubectl get secret cacerts -n istio-system -o json | jq -r '.data."cert-chain.pem"' | base64 --decode | openssl x509 -text -noout
```
Output:
```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7f:54:94:46:a3:35:7b:e8:22:99:bc:8b:26:5b:49:0b
    Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=CA, O=Istio, OU=abc, CN=www.example.com
        Validity
            Not Before: Jul 27 23:04:52 2022 GMT
            Not After : Jul 28 00:04:52 2023 GMT
        Subject: C=CA, O=Istio, OU=Security, CN=www.abc.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
...
```
Before proceeding, ensure that the istiod Pod in the istio-system namespace is deleted to force ISTIO using the newly created cacerts for the workload.
```bash
kubectl delete po -l app=istiod -n istio-system
```
