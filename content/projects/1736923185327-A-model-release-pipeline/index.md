---
title: "A model release pipeline"
date: 2025-01-15
draft: false
description: "a description"
tags: ["example", "tag"]
summary: "A model release pipeline for our semantic search"
---
 
## Reference Links
[PR by me](https://github.com/opensearch-project/opensearch-py-ml/pull/394)

[The model released](https://huggingface.co/opensearch-project)
## Introduction

In AWS OpenSearch, we provide pre-trained models for users to download. These pre-trained models are managed through a dedicated pipeline, where we conduct various validations, from training to inference and deployment. The models are then uploaded to a designated server, and the relevant documentation is updated accordingly.

After releasing a new semantic model, we found that the original pipeline could not effectively validate and deploy it since it is not a conventional dense model. As a result, we decided to implement a new pipeline to manage and maintain our neural sparse model.

## The work I did	
Automated the upload process for Sparse models:
I wrote (or modified) a complete GitHub Actions workflow along with the necessary Python code to perform the following actions under specific conditions (when model_type = Sparse):
- Prepare input parameters (model source, ID, version, tracking format, embedding dimension, pooling mode, description, etc.)
- Automatic tracing: Convert the original model into TorchScript or ONNX format, or generate both formats simultaneously.
- Embedding validation: Ensure that the converted model's embedding dimensions and functionalities are correct.
- Check if the model already exists: If it exists and cannot be overwritten, reject the upload.
- Manual approval: Officially upload to S3 only after approval from key personnel.
- Update repository documentation and records: Append the upload action to MODEL_UPLOAD_HISTORY.md, supported_models.json, and update CHANGELOG.md.
- Trigger downstream Jenkins processes to further complete the model's release or integration in ml-models.

## Technical Used
- Python
- GitHub Actions
- Huggingface
- Jenkins
- AWS S3
- AWS OpenSearch
- AWS IAM
- AWS Secrets Manager