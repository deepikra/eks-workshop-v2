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

