# Infrastructure as Code (IaC) Testing Guide: Terratest & LocalStack

## Introduction

Modern cloud architectures are no longer configured through manual clicking in the AWS or Azure web consoles. Infrastructure is written as version-controlled code using **Infrastructure as Code (IaC)** tools like **HashiCorp Terraform**, **AWS CloudFormation**, and **OpenTofu**.

However, treating infrastructure as code means that infrastructure can contain bugs just like application software. A typo in a security group can expose internal databases to the public internet (`0.0.0.0/0`), an invalid CIDR block can break VPC routing, and misconfigured IAM policies can grant excessive privileges.

QA engineers and platform testers must validate infrastructure code through automated testing frameworks like **Terratest** and cloud emulators like **LocalStack**.

---

## The Infrastructure Testing Pyramid

```
┌─────────────────────────────────────────────────────────────┐
│                 End-to-End Infrastructure Testing           │
│                 (Terratest against Staging Cloud Account)   │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Local Integration Testing                   │
│                 (Terraform against LocalStack Docker)       │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Static Security & Policy Scanning           │
│                 (TFLint, tfsec, Checkov, OPA Conftest)      │
└─────────────────────────────────────────────────────────────┘
```

1. **Static Analysis & Linting (Fast & Cheap)**: Tools like `tflint` and `checkov` inspect raw `.tf` files for syntax errors and compliance violations (e.g., flagging unencrypted S3 buckets).
2. **Local Emulation (LocalStack)**: Provisions mock AWS cloud resources (S3, SQS, DynamoDB) inside Docker containers without incurring cloud bills.
3. **Automated Integration (Terratest)**: Deploys real infrastructure, asserts connectivity, and automatically destroys the environment upon test completion.

---

## What is Terratest?

**Terratest** is a Go-based open-source testing library created by Gruntwork. It allows QA and DevOps engineers to write automated tests for Terraform code, Docker images, and Kubernetes Helm charts.

### The Terratest Lifecycle:
```
[ Init & Apply ] ─────► Deploys real test infrastructure via Terraform
       │
       ▼
[ Assert Phase ] ─────► Makes HTTP calls, queries endpoints, checks SSL certificates
       │
       ▼
[ Defer Destroy ] ────► Guarantees complete infrastructure destruction to prevent bills!
```

---

## Practical Test Code: Validating an S3 Bucket & Web Server with Terratest

```go
package test

import (
	"crypto/tls"
	"fmt"
	"testing"
	"time"

	http_helper "github.com/gruntwork-io/terratest/modules/http-helper"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestWebServerAndBucketProvisioning(t *testing.T) {
	t.Parallel()

	// 1. Configure Terraform options and test variables
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../modules/web-server",
		Vars: map[string]interface{}{
			"environment": "qa-automated-test",
			"instance_type": "t3.micro",
		},
	})

	// 2. CRITICAL: Ensure infrastructure is ALWAYS destroyed upon test completion!
	defer terraform.Destroy(t, terraformOptions)

	// 3. Deploy infrastructure (terraform init && terraform apply)
	terraform.InitAndApply(t, terraformOptions)

	// 4. Extract outputs from Terraform state
	serverURL := terraform.Output(t, terraformOptions, "web_server_url")
	bucketName := terraform.Output(t, terraformOptions, "storage_bucket_name")

	assert.NotEmpty(t, bucketName)

	// 5. Assert: Verify the deployed web server responds with HTTP 200 OK
	tlsConfig := tls.Config{}
	maxRetries := 15
	timeBetweenRetries := 5 * time.Second

	http_helper.HttpGetWithRetry(
		t,
		serverURL,
		&tlsConfig,
		200,
		"Hello World from Automated QA!",
		maxRetries,
		timeBetweenRetries,
	)
}
```

---

## SQA Interview Questions & Answers

### Q: Why is `defer terraform.Destroy()` the most critical statement in a Terratest suite?
**Answer:**
Terratest provisions real cloud infrastructure (EC2 instances, Load Balancers, RDS databases) in a live cloud provider account (e.g., AWS). If the test encounters an assertion failure or panic and the teardown step is skipped, the cloud resources will remain running indefinitely, incurring massive, unexpected monthly cloud hosting charges. Using Go's `defer` statement guarantees that the destruction routine executes even if the test fails midway.

### Q: What is Checkov and how does it prevent security defects in IaC?
**Answer:**
Checkov is a static code analysis tool for Infrastructure as Code. It scans Terraform, CloudFormation, Kubernetes manifests, and Dockerfiles against hundreds of pre-built security benchmarks (CIS Benchmarks). Checkov flags insecure infrastructure configurations (such as public S3 buckets, unrestricted SSH port 22 access, unencrypted EBS volumes, or missing backup retention policies) before code is ever applied to a cloud account.

---

## Key Takeaways

* Treat infrastructure with the same software testing rigor as application code.
* Use static security scanners (Checkov, tfsec) to catch cloud compliance violations early.
* Use Terratest to programmatically provision, assert, and destroy test infrastructure automatically in CI/CD.

---

## Conclusion

Infrastructure as Code testing ensures that the foundation upon which software runs is secure, performant, and reliable. By combining static policy scanners with automated integration tests using Terratest and LocalStack, QA engineers prevent catastrophic cloud outages, data leaks, and unexpected hosting costs.
