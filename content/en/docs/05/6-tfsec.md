---
title: "tfsec"
weight: 56
sectionnumber: 5.6
---


There tools to help detect common misconfigurations in terraform, on such tools is `tfsec` which has become part of the familiar `trivy` suite.

## Task {{% param sectionnumber %}}.1: Create a Terraform Configuration with a Misconfiguration

As you have learned, the Terraform state represents the applied objects that have been successfully applied. Here is the example from our first applied config:

```bash
mkdir $LAB_ROOT/advanced/tfsec
cd $LAB_ROOT/advanced/tfsec
```

Create a file named `main.tf` with the following content:

```bash
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "bad_example" {
  bucket = "my-unsecure-bucket"
  acl    = "public-read"  # Warning: Publicly readable bucket
}
```

This Terraform configuration defines an AWS S3 bucket with an acl (Access Control List) set to public-read, which is considered insecure because it allows public access to the bucket's content.

## Task {{% param sectionnumber %}}.2: Scan the Configuration Using Trivy

Run Trivy with tfsec-like functionality to scan the main.tf file:

```bash
trivy config .
```

The scan should detect that the S3 bucket has been configured with a public-read ACL, which is a potential security risk.

```
2024-10-21T10:15:35.123Z    INFO    Detected config file type: terraform

main.tf
========
Result: AWS S3 bucket allows public access via ACL (public-read)
       Severity: MEDIUM
       Line: 6
       Resource: aws_s3_bucket.bad_example
       Resolution: Set a more restrictive ACL or bucket policy
```

## Task {{% param sectionnumber %}}.3: Fix the misconfiguration

Update the `main.tf` file to make the S3 bucket private:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "fixed_example" {
  bucket = "my-secure-bucket"
  acl    = "private"  # Private bucket
}
```

and then run `trivy` again

```bash
trivy config .
```

Make sure to run an analyzing tool with approriate policies for the pipeline of your IaC. By default trivy will only exit with a returncode of 0 when all checks succeed.
