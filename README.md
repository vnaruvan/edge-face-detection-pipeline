# Edge Face Detection Pipeline (AWS IoT Greengrass + MQTT + SQS + Lambda)

[![AWS IoT Greengrass](https://img.shields.io/badge/AWS-IoT%20Greengrass-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/greengrass/)
[![AWS IoT Core](https://img.shields.io/badge/AWS-IoT%20Core-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/iot-core/)
[![MQTT](https://img.shields.io/badge/Protocol-MQTT-660066?logo=eclipsemosquitto&logoColor=white)](https://mqtt.org/)
[![Amazon SQS](https://img.shields.io/badge/AWS-SQS-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/sqs/)
[![AWS Lambda](https://img.shields.io/badge/AWS-Lambda-FF9900?logo=awslambda&logoColor=white)](https://aws.amazon.com/lambda/)
[![Amazon ECR](https://img.shields.io/badge/AWS-ECR-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/ecr/)
[![Amazon EC2](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonec2&logoColor=white)](https://aws.amazon.com/ec2/)
[![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)](https://www.python.org/)

Distributed face recognition pipeline that moves face detection to the edge. An IoT client publishes video frames over MQTT to a Greengrass Core device. A Greengrass component performs MTCNN face detection locally and sends detected faces to an SQS request queue, which triggers a cloud face recognition Lambda. Results are published to an SQS response queue and retrieved by the client.

> Source code is not public. This repository documents architecture, interfaces, and operational behavior.

---

## System overview

- Ingress: IoT client publishes frames to MQTT topic `clients/<ASU-ID>-IoTThing`
- Edge stage: Greengrass component `com.clientdevices.FaceDetection` runs MTCNN on the Core device and produces detected faces
- Cloud stage: detected faces are sent to `<ASU ID>-req-queue`, triggering `face-recognition` on AWS Lambda
- Egress: recognition results are published to `<ASU ID>-resp-queue` and retrieved by the IoT client

---

## Architecture

### Components
- Edge compute: AWS IoT Greengrass Core running on an EC2 instance named `IoT-Greengrass-Core`
- IoT client: EC2 instance named `IoT-Greengrass-Client` publishing frames over MQTT
- IoT identity: IoT Thing named `<ASU-ID>-IoTThing`
- Greengrass component: `com.clientdevices.FaceDetection` subscribing to `clients/<ASU-ID>-IoTThing`
- Cloud services: SQS request and response queues, Lambda face recognition

~~~mermaid
flowchart LR
  Client[IoT client] -->|MQTT publish| Core[Greengrass Core device]
  Core -->|delivered to component| FD[Greengrass component FaceDetection]
  FD -->|detected faces plus request_id| REQ[SQS request queue]
  REQ -->|triggers| FR[Lambda face-recognition]
  FR -->|request_id plus result| RESP[SQS response queue]
  Client <-->|poll results| RESP

  FD --> CW1[Greengrass component logs]
  FR --> CW2[CloudWatch logs]
~~~

---

## Interface contract

MQTT message to FaceDetection

Topic:
- `clients/<ASU-ID>-IoTThing`

Payload fields:
- `encoded`: base64 encoded input image
- `request_id`: unique ID for correlation
- `filename`: input image name

Example:

~~~json
{ "encoded": "<base64>", "request_id": "123", "filename": "frame_01.jpg" }
~~~

Edge to cloud handoff

- If a face is detected, the component sends the detected face data to the SQS request queue `<ASU ID>-req-queue` to trigger cloud recognition.
- Cloud recognition publishes results to `<ASU ID>-resp-queue` for client retrieval as:

~~~json
{ "request_id": "<id>", "result": "<classification result>" }
~~~

Optional edge optimization

- If no face is detected, the edge component can publish directly to the response queue with:

~~~json
{ "request_id": "<id>", "result": "No-Face" }
~~~

---

## Implementation notes

Edge first design

Face detection runs on the Greengrass Core device, reducing raw frame movement to the cloud and enabling faster response when faces are absent.

Loose coupling via SQS

Edge detection and cloud recognition are decoupled through SQS so each stage can scale and fail independently.

Request correlation

`request_id` is preserved from MQTT ingress through the response queue for end to end tracing and correct result matching.

---

## Technologies

Core

Python

Edge and IoT

AWS IoT Greengrass v2 (component runtime on core device)  
AWS IoT Core and MQTT (publish subscribe messaging)  
AWS IoT Device SDK v2 for Python (client discovery and MQTT connectivity)

Cloud

Amazon SQS (request and response queues)  
AWS Lambda (face recognition triggered by request queue)  
Amazon CloudWatch (logs and diagnostics)  
Amazon EC2 (core and client device emulation)

ML and dependencies

MTCNN (face detection)  
FaceNet based recognition in cloud stage  
Torch CPU builds and common utilities (numpy, boto3)

---

## Benchmark (representative run)

Workload

100 video frames published over MQTT to the FaceDetection topic and processed end to end.

Results

Requests completed: 100/100  
Failed requests: 0  
Correct predictions: 100/100  
Total runtime: 77.57 seconds  
Average end to end latency: 0.776 seconds per request

Notes

Latency depends on image size, edge CPU, Greengrass component startup state, queue depth, and Lambda memory allocation.

---

## Operational notes

Where to look for edge logs

Greengrass component stdout is persisted on the core device under:
- `/greengrass/v2/logs/`

Useful controls on the core device

- List components: `sudo /greengrass/v2/bin/greengrass-cli component list`
- Restart component: `sudo /greengrass/v2/bin/greengrass-cli component restart --names "com.clientdevices.FaceDetection"`

Queue hygiene and cost control

Ensure messages are consumed and deleted to avoid repeated triggering and unnecessary invocations.

---

## Repository notes

This repository is intended to present the architecture and design of the project only. Source code is not shared online due to course guidelines. Please contact me for the code.

:contentReference[oaicite:0]{index=0}



