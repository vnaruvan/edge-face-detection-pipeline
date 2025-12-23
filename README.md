# Edge Face Detection Pipeline

This project implements an end-to-end pipeline for performing face detection on image data from edge devices using AWS Greengrass, MQTT, and Lambda. The system was designed to reduce network usage and latency by processing images close to the source and sending only essential metadata to the cloud.

## Architecture Overview

- An edge device captures images and publishes them as base64-encoded messages over MQTT.
- A custom Greengrass component subscribes to the MQTT topic, parses the image payload, and sends the data to an SQS queue.
- A Lambda function, triggered by SQS, runs face detection using the MTCNN model (`facenet-pytorch`) and logs results or stores metadata in a downstream database or S3 bucket.

## Technologies

- Python, Shell scripting
- AWS Greengrass V2, IoT Core, Lambda, SQS, EC2
- MQTT protocol, facenet-pytorch, boto3

## Features

- Real-time data flow from edge to cloud with minimal delay
- Decoupled architecture using SQS for reliability and scalability
- Designed for low-bandwidth, latency-sensitive environments

## Notes

This repository is intended to present the architecture and design of the project only. Source code is not shared online due to course guidelines. Please contact me for the code.
