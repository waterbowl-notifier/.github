# Cat Water Bowl Sensor

## Summary

The Cat Water Bowl Sensor project is designed to help pet owners monitor the water levels in their cat's water bowl to ensure it never runs dry. By leveraging an ESP32 microcontroller paired with an HC-SR04 ultrasonic sensor, this system accurately measures the water level in the bowl. The data collected by the sensor is sent via HTTPS REST requests to AWS, where it is processed by Lambda functions and stored in DynamoDB for state tracking and reporting.

### Key Features
- **Real-time Water Level Monitoring:** Continuously measures the water level in the bowl and sends the data to AWS.
- **AWS Integration:** Uses AWS Lambda functions for data processing, state tracking, and cleanup, with data stored in DynamoDB.
- **MicroPython on ESP32:** Runs MicroPython on the ESP32 microcontroller for efficient and reliable performance.
- **Easy Setup:** Detailed setup instructions for both the hardware and software components.

### Components
- **ESP32 Microcontroller:** The main control unit running the MicroPython firmware.
- **HC-SR04 Ultrasonic Sensor:** Used to measure the distance to the water surface and determine the water level.
- **AWS Services:** API Gateway for receiving data, Lambda functions for processing, and DynamoDB for storage.

### How It Works
1. **Measurement:** The HC-SR04 sensor attached to the ESP32 measures the distance from the sensor to the water surface.
2. **Data Transmission:** The ESP32 sends the measurement data to AWS via HTTPS REST requests.
3. **Data Processing:** AWS Lambda functions process the incoming data, update the state in DynamoDB, and generate reports.
4. **Reporting and Cleanup:** Additional Lambda functions handle periodic reporting and cleanup of old data in DynamoDB.

This project ensures that pet owners can keep a close watch on their cat's water bowl, providing peace of mind that their pet always has access to fresh water.
