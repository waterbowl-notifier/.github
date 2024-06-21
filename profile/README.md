# Water Bowl Monitoring System

## Introduction

Welcome to the Water Bowl Monitoring System project! This innovative solution leverages IoT (Internet of Things) technology to ensure that water bowls for pets or livestock are consistently filled and functional. The system is built around an ESP32 microcontroller equipped with an HCSR04 ultrasonic sensor, which measures the water level in a bowl and reports this data to an AWS-based backend for monitoring and alerting.

## Overview

The primary objective of this project is to automate the monitoring of water levels in bowls, providing real-time updates and alerts to ensure that the bowls are always adequately filled. This system minimizes the risk of water shortages for animals and ensures timely intervention when refills are needed or if any issues arise.

## System Components

### ESP32 Microcontroller with HCSR04 Sensor
The ESP32 microcontroller, running MicroPython, is equipped with an HCSR04 ultrasonic sensor. This sensor measures the distance from the top of the bowl to the water surface, allowing the system to calculate the current water level. Every 20 minutes, the ESP32 sends the measured water level as a float value to the AWS backend via an HTTPS REST request.

### AWS Backend
The AWS backend comprises a set of Lambda functions and a DynamoDB table to manage and process the incoming data. The backend is responsible for determining the state of each water bowl and sending notifications when necessary. The following Lambda functions support the system:

1. **State Lambda**: This function processes POST HTTP requests from the ESP32 devices, determines the water level status (green, yellow, red, black), and updates the DynamoDB table with the current status and timestamp.

2. **State Reporting Lambda**: This function scans the DynamoDB table for the latest water bowl statuses, evaluates the timeliness of the records, and generates detailed reports for non-green statuses. It sends notifications using AWS SNS.

3. **Keepalive Monitor Lambda**: This function ensures the continuous operation of the system by monitoring the timestamped entries in the DynamoDB table. If a water bowl has not reported a status update within a specified time limit, it sends an alert via SNS.

4. **DynamoDB Cleanup Lambda**: This function automatically deletes outdated entries from the DynamoDB table based on a predefined age threshold, ensuring the table remains manageable and performance is optimized.

5. **DDB Cleanup Monitor Lambda**: This function monitors the item count in the DynamoDB table and sends a notification if the number of items exceeds a predefined threshold, indicating potential issues.

## Water Bowl States

The system identifies four possible states for each water bowl based on the sensor readings:
- **Green**: The bowl is full.
- **Yellow**: The bowl needs refilling soon.
- **Red**: The bowl is nearly empty.
- **Black**: There is a malfunction or the water level cannot be quantified.

These states help prioritize maintenance actions and ensure that all bowls are properly monitored and maintained.

## Setup and Deployment

### Hardware Setup
1. Connect the HCSR04 sensor to the ESP32 microcontroller.
2. Program the ESP32 with MicroPython to measure the water level and send data to the AWS backend.

### AWS Backend Setup
1. Deploy the Lambda functions and set up the DynamoDB table.
2. Configure the environment variables for each Lambda function, such as DynamoDB table names, thresholds, and SNS topics.
3. Set up the necessary IAM roles and permissions to allow the Lambda functions to interact with DynamoDB and SNS.

### Testing and Monitoring
1. Test the ESP32 device to ensure it correctly measures and sends water level data.
2. Monitor the AWS backend to verify that data is being processed and statuses are updated correctly.
3. Check the SNS notifications to ensure alerts are sent as expected.

## Conclusion

The Water Bowl Monitoring System is a robust solution for ensuring the consistent availability of water for pets and livestock. By integrating IoT technology with AWS services, the system provides real-time monitoring, efficient data management, and timely notifications, making it a reliable tool for animal care management. This README provides an overview of the project setup and functionality, serving as a guide to understanding and deploying the system.
