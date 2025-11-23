# Water Bowl Monitoring System

## Overview

This project implements an IoT-based monitoring system for water bowls (e.g., for pets or automated feeders). It uses ESP32 devices with ultrasonic sensors to measure water levels, sends data to AWS IoT Core via MQTT, processes the data in AWS Lambda to determine bowl states (green, yellow, red, black), stores statuses in DynamoDB, and controls RGB LEDs on the devices for visual feedback. A separate monitoring Lambda periodically checks DynamoDB for stale data or non-optimal states and sends notifications via SNS.

The system emphasizes reliability with features like exception handling, retry logic, timezone-aware timestamps, and configurable thresholds. It is built using MicroPython for the device firmware and Python for AWS Lambda functions.

Key goals:
- Real-time water level monitoring and visualization via LED colors.
- Alerts for low water levels or device failures.
- Low-power operation on battery-powered ESP32 devices.

## Architecture

1. **Device Layer (ESP32 with MicroPython)**:
   - Ultrasonic sensor (HC-SR04) measures distance to water surface (higher distance = lower level).
   - Connects to Wi-Fi, syncs time via NTP, publishes water level to AWS IoT MQTT.
   - Subscribes to MQTT for LED color commands.
   - Enters deep sleep/low-power modes between measurements.

2. **Cloud Ingestion (AWS IoT Core)**:
   - Receives telemetry via MQTT topics (e.g., `local/waterbowl/<hostname>/status/depth`).
   - Triggers Lambda via IoT Rules.

3. **Processing Layer (AWS Lambda - State Machine)**:
   - Validates input, determines state based on thresholds.
   - Updates DynamoDB with status and timestamp.
   - Publishes LED color back to device via MQTT.

4. **Storage (DynamoDB)**:
   - Stores latest status per device (hostname as primary key).

5. **Monitoring Layer (AWS Lambda - Monitor)**:
   - Scans DynamoDB periodically (e.g., via CloudWatch Events).
   - Checks for outdated timestamps or non-green statuses.
   - Sends SNS notifications with emoji-based summaries.

6. **Notifications (SNS)**:
   - Alerts for issues, simplified for quick reading (e.g., location from hostname).

## Components

### Device Firmware (MicroPython)

The ESP32 firmware is split across several files for modularity:

- **hcsr04.py**: Interfaces with the HC-SR04 ultrasonic sensor.
  - Provides methods for single and multiple measurements (returns most common value for noise reduction).
  - Uses `machine` and `utime` for GPIO and timing.
  - Distance calculation: `(pulse_time / 2) / 29.1` cm (approximates speed of sound).

- **exception_notifier.py**: Handles crashes by logging tracebacks and sending HTTP notifications.
  - Logs to a local file and POSTs to a configurable URL.
  - Uses `urequests` for HTTP and `uio` for traceback capture.

- **main.py**: Core script for device operation.
  - Loads config from `my_config.json` (Wi-Fi, pins, certs, etc.).
  - Connects to Wi-Fi with retries and LED blinking.
  - Measures water level, publishes to MQTT with TLS.
  - Subscribes to color topic and updates RGB LED via PWM.
  - Sleeps for configurable minutes (deep sleep if LED off).
  - Utilities: Prime-based backoff, MAC address formatting, MQTT connection with DER certs.

Configuration (`my_config.json` example structure):
```json
{
  "wifi_ssid": "your-ssid",
  "wifi_password": "your-password",
  "sleep_minutes": 5,
  "measure_count": 3,
  "iot_endpoint": "your-iot-endpoint.amazonaws.com",
  "crash_notify_url": "https://your-notify-url.com",
  "rgb_red_pin": 12,
  "rgb_green_pin": 13,
  "rgb_blue_pin": 14,
  "<mac-address>": {
    "hostname": "bowl-1",
    "gpio_hcsr04_trigger_pin": 5,
    "gpio_hcsr04_echo_pin": 18,
    "cert": "/path/to/cert.der",
    "key": "/path/to/key.der",
    "ca": "/path/to/ca.der"
  }
}
```

### AWS Lambda Functions (Python)

- **State Machine Lambda** (e.g., `water_bowl_state_machine.py`):
  - Triggered by AWS IoT events with `water_level` and `hostname`.
  - Validates input, ignores tests, rounds levels to 1 decimal.
  - Determines state: black (full/error), green, yellow, red (based on distance thresholds).
  - Updates DynamoDB (hostname PK, status, US/Central ISO timestamp).
  - Publishes MQTT color command (e.g., `{"LED": "0,255,0"}` for green).
  - Logging with WBSxxx codes for easy CloudWatch filtering.
  - Environment vars: Thresholds, DynamoDB table, IoT endpoint, etc.
  - Notes: S3 remnants unused; handles errors by logging/early return.

- **Monitoring Lambda** (e.g., `water_bowl_monitor.py`):
  - Scans DynamoDB for all items.
  - Checks timestamps against `TIME_DELTA_HOURS` in US/Central.
  - If outdated: SNS alert listing offenders.
  - If valid but non-green: SNS summary with emojis (e.g., 🟡 for yellow, 🚨 for red).
  - Simplifies hostnames (e.g., 'bowl-1' → 'bowl').
  - Environment vars: Table name, region, SNS topic ARN, delta hours.
  - Trigger: Periodic via CloudWatch.

DynamoDB Table Schema (`water-bowl-status`):
- Primary Key: `hostname` (String)
- Attributes:
  - `waterbowl_status` (String: 'green'|'yellow'|'red'|'black')
  - `waterbowl_timestamp` (String: ISO-8601 in US/Central)

### AWS Resources

- **IoT Core**: Custom endpoint for MQTT; rules to trigger state Lambda on depth topics.
- **SNS Topic**: For notifications; subscribe via email/SMS.
- **DynamoDB**: Simple table for status persistence.
- **CloudWatch**: For logs and scheduling monitor Lambda.

## Setup and Deployment

1. **Device Setup**:
   - Flash MicroPython on ESP32.
   - Upload firmware files and `my_config.json`.
   - Wire HC-SR04 (trigger/echo pins) and RGB LED (PWM pins).
   - Generate/provision AWS IoT certs (DER format).

2. **AWS Setup**:
   - Create IoT Thing per device with certs.
   - Set up IoT Rule: SQL SELECT on depth topic → Lambda action.
   - Deploy Lambdas with IAM roles (IoT, DynamoDB, SNS access).
   - Create DynamoDB table.
   - Create SNS topic and subscribe.
   - Schedule monitor Lambda (e.g., every hour).

3. **Testing**:
   - Simulate events in Lambda console.
   - Use test hostnames (ignored if contain 'test').
   - Monitor CloudWatch logs (search WBSxxx).

## Environment Variables

- **State Lambda**:
  - BUCKET_NAME (legacy)
  - REGION
  - GREEN_THRESHOLD_CM (e.g., 6.0)
  - YELLOW_THRESHOLD_CM (e.g., 10.0)
  - RED_THRESHOLD_CM (e.g., 11.0)
  - DYNAMODB_TABLE
  - IOT_ENDPOINT

- **Monitor Lambda**:
  - TABLE_NAME
  - REGION
  - TOPIC_ARN
  - TIME_DELTA_HOURS (e.g., 1.0)

## Error Handling and Logging

- Device: Notifies crashes via HTTP; resets on exceptions.
- State Lambda: Logs failures; continues on MQTT errors.
- Monitor Lambda: Logs scan errors; returns 500 on failures.
- All use US/Central for consistency.

## Notes

- Thresholds assume distance sensor: Adjust for bowl size.
- LEDs: 0-255 RGB mapped to 16-bit PWM.
- Power: Deep sleep for efficiency.
- Scalability: Supports multiple bowls via hostnames.
- Future: Add S3 for historical data; temperature compensation for sensor.

For contributions or issues, see repository guidelines.
