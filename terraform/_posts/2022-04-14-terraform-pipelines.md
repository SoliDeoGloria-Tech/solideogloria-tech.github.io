---
title: Deploying Terraform via a DevOps Pipeline
layout: post
tags:
  - Azure DevOps
  - Pipeline
  - Terraform
---

Not everyone is privileged to be able to use Terraform Cloud for deploying their Terraform infrastructure.
This means that teams need to use their existing DevOps tooling to deploy their infrastructure via Terraform.

While I've seen many examples of pipelines for deploying Terraform code with various services, it felt like something was missing.
Most example pipelines were designed to just run once a code review had occurred, and often would automatically deploy the changed code without any intervention.
This wasn't going to fly for us in a recent project.
We needed a more robust plan for deployment, one that would cater for not only deployment of the infrastructure, but an opportunity to wait for approval of a specific plan, plus checks to make sure that the newly-committed code was up to standard.

And so I built an [Azure DevOps Pipeline][ado-pipeline] with the following features:

- Requirement for someone to manually approve the deployment of a Terraform plan.
- Email notification of pending approval of a specific plan.
- Support for deploying multiple environments (e.g. Dev, Test, and Production) from the same Terraform code in one pipeline.
- Force a `terraform fmt` and `terraform validate` during a pull request to reduce the number of errors committed to the main branch.
- Generate a speculative plan on a pull request so it can be analysed during the code review.
- Skip approval and deployment when there are no changes required to be made during deployment.

This provides a pipeline that only runs the plan, along with the validation on a pull request…

![Pipeline run from a pull request](/solideogloria-tech.github.io/images/terraform-pipelines/pull-request.png)

…while also allowing a full deployment with approval and notification.

![Pipeline run from a pull request](/solideogloria-tech.github.io/images/terraform-pipelines/apply-after-approval.png)

For the approval process, I considered splitting the pipeline into two stages — one to create the Terraform plan and the other to deploy it.
This would have allowed me to use Environmental approvals rather than the `ManualValidation` approval, but I decided against it as it would have cluttered the display of deploying multiple environments.
A separate pipeline could have been used per environment to work around this, but we wanted a single pipeline with full code promotion.

[ado-pipeline]: https://github.com/oWretch/terraform-pipelines/tree/main/azure-devops
