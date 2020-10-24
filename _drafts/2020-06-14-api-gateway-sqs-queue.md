---
layout: post
title: "AWS - Asynchronous API request processing using a message queue"
date: 545345545
tags:
- AWS
- SQS
- Howto guide
excerpt: This article explains how to push messages from an API in AWS API Gateway to an SQS queue
---

Integrating an SQS Queue with an API in AWS is a good way to process API requests asynchronously while returning a response to the caller that their request has been received and will be processed in due course.

We are assuming that the following have been created:
- An SQS queue (Standard or FIFO)
- An API in API Gateway that will be pushing messages to the SQS queue

## IAM Resources

We will need an IAM Policy to allow the API to push messages to the SQS queue. If the policy won't be being shared with other resources or won't be being modified often, I'd keep it within the API itself as an inline policy in order to keep everything in one place.