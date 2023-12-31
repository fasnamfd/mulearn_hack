# mulearn_hack

Grove - VOC and eCO2 Gas Sensor(SGP30) using code written below and this code was sourced from seeedstudio official documentation.
```#include <Arduino.h>

#include "sensirion_common.h"
#include "sgp30.h"


void setup() {
    s16 err;
    u16 scaled_ethanol_signal, scaled_h2_signal;
    Serial.begin(115200);
    Serial.println("serial start!!");

   
    /*  Init module,Reset all baseline,The initialization takes up to around 15 seconds, during which
        all APIs measuring IAQ(Indoor air quality ) output will not change.Default value is 400(ppm) for co2,0(ppb) for tvoc*/
    while (sgp_probe() != STATUS_OK) {
        Serial.println("SGP failed");
        while (1);
    }
    /*Read H2 and Ethanol signal in the way of blocking*/
    err = sgp_measure_signals_blocking_read(&scaled_ethanol_signal,
                                            &scaled_h2_signal);
    if (err == STATUS_OK) {
        Serial.println("get ram signal!");
    } else {
        Serial.println("error reading signals");
    }
    err = sgp_iaq_init();
    //
}

void loop() {
    s16 err = 0;
    u16 tvoc_ppb, co2_eq_ppm;
    err = sgp_measure_iaq_blocking_read(&tvoc_ppb, &co2_eq_ppm);
    if (err == STATUS_OK) {
        Serial.print("tVOC  Concentration:");
        Serial.print(tvoc_ppb);
        Serial.println("ppb");

        Serial.print("CO2eq Concentration:");
        Serial.print(co2_eq_ppm);
        Serial.println("ppm");
    } else {
        Serial.println("error reading IAQ values\n");
    }
    delay(1000);
}
```
![image](https://user-images.githubusercontent.com/30889567/188283975-b6774e83-1bf0-4c3a-b7e7-9929ee7dfddc.png)


Code for Grove Temp&Hum using sht40 and this is also copied from seeedstudio official doc

```#include <Arduino.h>
#include <SensirionI2CSht4x.h>
#include <Wire.h>

SensirionI2CSht4x sht4x;

void setup() {

    Serial.begin(115200);
    while (!Serial) {
        delay(100);
    }

    Wire.begin();

    uint16_t error;
    char errorMessage[256];

    sht4x.begin(Wire);

    uint32_t serialNumber;
    error = sht4x.serialNumber(serialNumber);
    if (error) {
        Serial.print("Error trying to execute serialNumber(): ");
        errorToString(error, errorMessage, 256);
        Serial.println(errorMessage);
    } else {
        Serial.print("Serial Number: ");
        Serial.println(serialNumber);
    }
}

void loop() {
    uint16_t error;
    char errorMessage[256];

    delay(1000);

    float temperature;
    float humidity;
    error = sht4x.measureHighPrecision(temperature, humidity);
    if (error) {
        Serial.print("Error trying to execute measureHighPrecision(): ");
        errorToString(error, errorMessage, 256);
        Serial.println(errorMessage);
    } else {
        Serial.print("Temperature:");
        Serial.print(temperature);
        Serial.print("\t");
        Serial.print("Humidity:");
        Serial.println(humidity);
    }
}

```

![image](https://user-images.githubusercontent.com/30889567/188283664-5a6a692f-45f6-43f6-9b10-9bac514c7fb4.png)


Code for soil moisture sensor

```
int sensorPin = A0;
int sensorValue = 0;
 
void setup() {
    Serial.begin(9600);
}
void loop() {
    // read the value from the sensor:
    sensorValue = analogRead(sensorPin);
    Serial.print("Moisture = " );
    Serial.println(sensorValue);
    delay(1000);
}
```
![image](https://user-images.githubusercontent.com/30889567/188284115-cbceb47a-3ad1-488d-8a67-9e9ec11bd983.png)


Code for data collection in edge impulse
``` #include <Wire.h>

#include <SensirionI2CSht4x.h>
#include "sensirion_common.h"
#include "sgp30.h"

// Settings
#define BTN_START           0                         // 1: press button to start, 0: loop
#define BTN_PIN             WIO_5S_PRESS              // Pin that button is connected to
#define SAMPLING_FREQ_HZ    4                         // Sampling frequency (Hz)
#define SAMPLING_PERIOD_MS  1000 / SAMPLING_FREQ_HZ   // Sampling period (ms)
#define NUM_SAMPLES         8                         // 8 samples at 4 Hz is 2 seconds

int sensorPin = A0;
int sensorValue = 0;

SensirionI2CSht4x sht4x;

void setup() {
  // Initialize button
  pinMode(BTN_PIN, INPUT_PULLUP);
  
  // Start serial
  Serial.begin(115200);

  // Initialize environmental sensor
 while (!Serial) {
        delay(100);
    }
    Wire.begin();

    uint16_t error;
    char errorMessage[256];
    sht4x.begin(Wire);
    uint32_t serialNumber;
    error = sht4x.serialNumber(serialNumber);
    if (error) {
        Serial.print("Error trying to execute serialNumber(): ");
        errorToString(error, errorMessage, 256);
        Serial.println(errorMessage);
    } 

  // Initialize VOC and eCO2 sensor
  while (sgp_probe() != STATUS_OK) {
    Serial.println("Trying to initialize SGP30...");
    delay(1000);
  }

  
}

void loop() {


  uint16_t error;
  int16_t sgp_err;
  uint16_t sgp_tvoc;
  uint16_t sgp_co2;
  unsigned long timestamp;
  char errorMessage[256];

    float temperature;
    float humidity;
    error = sht4x.measureHighPrecision(temperature, humidity);

sensorValue = analogRead(sensorPin);

  // Wait for button press
#if BTN_START
  while (digitalRead(BTN_PIN) == 1);
#endif

  // Print header
  Serial.println("timestamp,temp,humd,co2,voc1,soilmst");

  // Transmit samples over serial port
  for (int i = 0; i < NUM_SAMPLES; i++) {

    // Take timestamp so we can hit our target frequency
    timestamp = millis();

  
    // Read BME680 environmental sensor
   if (error) {
        Serial.print("Error trying to execute measureHighPrecision(): ");
        errorToString(error, errorMessage, 256);
        Serial.println(errorMessage);
    }
  
    // Read SGP30 sensor
    sgp_err = sgp_measure_iaq_blocking_read(&sgp_tvoc, &sgp_co2);
    if (sgp_err != STATUS_OK) {
      Serial.println("Error: Could not read from SGP30");
      return;
    }

    // Print CSV data with timestamp
    Serial.print(timestamp);
    Serial.print(",");
    Serial.print(temperature);
    Serial.print(",");
    Serial.print(humidity);
    Serial.print(",");
    Serial.print(sgp_co2);
    Serial.print(",");
    Serial.print(sgp_tvoc);
    Serial.print(",");
    Serial.print(sensorValue);
    Serial.println();

    // Wait just long enough for our sampling period
    while (millis() < timestamp + SAMPLING_PERIOD_MS);
  }

  // Print empty line to transmit termination of recording
    Serial.println();

  // Make sure the button has been released for a few milliseconds
#if BTN_START
  while (digitalRead(BTN_PIN) == 0);
  delay(100);
#endif
}
```
Final code for inferencing 
``` 
#include <Wire.h>

#include <SensirionI2CSht4x.h>
#include "sensirion_common.h"
#include "sgp30.h"
#include "TFT_eSPI.h"                                 // Comes with Wio Terminal package

#include "FightFire-Zain-Project_inferencing.h"                      // Name of Edge Impulse library

// Settings
#define DEBUG               1                         // 1 to print out debugging info
#define DEBUG_NN            false                     // Print out EI debugging info
#define ANOMALY_THRESHOLD   0.3                       // Scores above this are an "anomaly"
#define SAMPLING_FREQ_HZ    4                         // Sampling frequency (Hz)
#define SAMPLING_PERIOD_MS  1000 / SAMPLING_FREQ_HZ   // Sampling period (ms)
#define NUM_SAMPLES         EI_CLASSIFIER_RAW_SAMPLE_COUNT  // 4 samples at 4 Hz
#define READINGS_PER_SAMPLE EI_CLASSIFIER_RAW_SAMPLES_PER_FRAME // 8


int sensorPin = A0;
int sensorValue = 0;
SensirionI2CSht4x sht4x;

// Preprocessing constants (drop the timestamp column)
float mins[] = {
  29.01, 79.28, 400.0, 0.0, 2.0
};
float ranges[] = {
   0.08, 0.47, 27.0, 136.0, 481.0
};


TFT_eSPI tft;                         // Wio Terminal LCD

void setup() {
  

  // Start serial
  Serial.begin(115200);

  // Configure LCD
  tft.begin();
  tft.setRotation(3);
  tft.setFreeFont(&FreeSansBoldOblique24pt7b);
  tft.fillScreen(TFT_BLACK);



  // Initialize environmental sensor
 while (!Serial) {
        delay(100);
    }
    Wire.begin();

    uint16_t error;
    char errorMessage[256];
    sht4x.begin(Wire);
    uint32_t serialNumber;
    error = sht4x.serialNumber(serialNumber);
    if (error) {
        Serial.print("Error trying to execute serialNumber(): ");
        errorToString(error, errorMessage, 256);
        Serial.println(errorMessage);
    } 

  // Initialize VOC and eCO2 sensor
  while (sgp_probe() != STATUS_OK) {
    Serial.println("Trying to initialize SGP30...");
    delay(1000);
  }

 
}

void loop() {
  
  uint16_t error;
  int16_t sgp_err;
  uint16_t sgp_tvoc;
  uint16_t sgp_co2;
  unsigned long timestamp;
  static float raw_buf[NUM_SAMPLES * READINGS_PER_SAMPLE];
  static signal_t signal;
  float temp;
  int max_idx = 0;
  float max_val = 0.0;
  char str_buf[40];
  char errorMessage[256];
  // Collect samples and perform inference
  for (int i = 0; i < NUM_SAMPLES; i++) {

    // Take timestamp so we can hit our target frequency
    timestamp = millis();

       float temperature;
    float humidity;
    error = sht4x.measureHighPrecision(temperature, humidity);

sensorValue = analogRead(sensorPin);
  
    // Read SGP30 sensor
    sgp_err = sgp_measure_iaq_blocking_read(&sgp_tvoc, &sgp_co2);
    if (sgp_err != STATUS_OK) {
      Serial.println("Error: Could not read from SGP30");
      return;
    }

    // Store raw data into the buffer
    raw_buf[(i * READINGS_PER_SAMPLE) + 0] = temperature;
    raw_buf[(i * READINGS_PER_SAMPLE) + 1] = humidity;
    raw_buf[(i * READINGS_PER_SAMPLE) + 2] = sgp_co2;
    raw_buf[(i * READINGS_PER_SAMPLE) + 3] = sgp_tvoc;
    raw_buf[(i * READINGS_PER_SAMPLE) + 4] = sensorValue;


    // Perform preprocessing step (normalization) on all readings in the sample
    for (int j = 0; j < READINGS_PER_SAMPLE; j++) {
      temp = raw_buf[(i * READINGS_PER_SAMPLE) + j] - mins[j];
      raw_buf[(i * READINGS_PER_SAMPLE) + j] = temp / ranges[j];
    }

    // Wait just long enough for our sampling period
    while (millis() < timestamp + SAMPLING_PERIOD_MS);
  }

  // Print out our preprocessed, raw buffer
#if DEBUG
  for (int i = 0; i < NUM_SAMPLES * READINGS_PER_SAMPLE; i++) {
    Serial.print(raw_buf[i]);
    if (i < (NUM_SAMPLES * READINGS_PER_SAMPLE) - 1) {
      Serial.print(", ");
    }
  }
  Serial.println();
#endif

  // Turn the raw buffer in a signal which we can the classify
  int err = numpy::signal_from_buffer(raw_buf, EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE, &signal);
  if (err != 0) {
      ei_printf("ERROR: Failed to create signal from buffer (%d)\r\n", err);
      return;
  }

  // Run inference
  ei_impulse_result_t result = {0};
  err = run_classifier(&signal, &result, DEBUG_NN);
  if (err != EI_IMPULSE_OK) {
      ei_printf("ERROR: Failed to run classifier (%d)\r\n", err);
      return;
  }

  // Print the predictions
  ei_printf("Predictions ");
  ei_printf("(DSP: %d ms., Classification: %d ms., Anomaly: %d ms.)\r\n",
        result.timing.dsp, result.timing.classification, result.timing.anomaly);
  for (int i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
    ei_printf("\t%s: %.3f\r\n", 
              result.classification[i].label, 
              result.classification[i].value);
  }

  // Print anomaly detection score
#if EI_CLASSIFIER_HAS_ANOMALY == 1
  ei_printf("\tanomaly acore: %.3f\r\n", result.anomaly);
#endif

  // Find maximum prediction
  for (int i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
    if (result.classification[i].value > max_val) {
      max_val = result.classification[i].value;
      max_idx = i;
    }
  }

  // Print predicted label and value to LCD if not anomalous
  tft.fillScreen(TFT_BLACK);
  if (result.anomaly < ANOMALY_THRESHOLD) {
    tft.drawString(result.classification[max_idx].label, 20, 60);
    sprintf(str_buf, "%.3f", max_val);
    tft.drawString(str_buf, 60, 120);
  } else {
    tft.drawString("Unknown", 20, 60);
    sprintf(str_buf, "%.3f", result.anomaly);
    tft.drawString(str_buf, 60, 120);
  }
}
````
![image](https://user-images.githubusercontent.com/30889567/190928530-bc018e4e-5279-4bd2-a2f5-a363f9d421dd.png)



Final data collection code

```
#include <Wire.h>
#include <SHT1x.h>
#include <SensirionI2CSht4x.h>
// Settings
#define BTN_START           0                         // 1: press button to start, 0: loop
#define BTN_PIN             WIO_5S_PRESS              // Pin that button is connected to
#define SAMPLING_FREQ_HZ    4                         // Sampling frequency (Hz)
#define SAMPLING_PERIOD_MS  1000 / SAMPLING_FREQ_HZ   // Sampling period (ms)
#define NUM_SAMPLES         8                         // 8 samples at 4 Hz is 2 seconds
#define dataPin 6
#define clockPin 8
SHT1x sht1x(dataPin, clockPin);
SensirionI2CSht4x sht4x;


void setup() {
  // Initialize button
  pinMode(BTN_PIN, INPUT_PULLUP);
  
  // Start serial
  Serial.begin(115200);
   while (!Serial) {
        delay(100);
    }
    Wire.begin();
    uint16_t error;
    char errorMessage[256];
    sht4x.begin(Wire);
    uint32_t serialNumber;
  }
  
void loop() {
  uint16_t error; 
  unsigned long timestamp;
  char errorMessage[256];

  float soil_temp;
  float soil_humidity;
  float temperature;
  float humidity;
  
  // Read values from the sensor
  error = sht4x.measureHighPrecision(temperature, humidity); 
  soil_temp = sht1x.readTemperatureC();
  soil_humidity = sht1x.readHumidity();

  // Wait for button press
#if BTN_START
  while (digitalRead(BTN_PIN) == 1);
#endif

  // Print header
  Serial.println("timestamp,soiltemp,soilhum,atmtemp,atmhum");

  // Transmit samples over serial port
  for (int i = 0; i < NUM_SAMPLES; i++) {

    // Take timestamp so we can hit our target frequency
    timestamp = millis();

    // Print CSV data with timestamp
    Serial.print(timestamp);
    Serial.print(",");
    Serial.print(soil_temp);
    Serial.print(",");
    Serial.print(soil_humidity);
    Serial.print(",");
    Serial.print(temperature);
    Serial.print(",");
    Serial.print(humidity);
    Serial.println();

    // Wait just long enough for our sampling period
    while (millis() < timestamp + SAMPLING_PERIOD_MS);
  }

  // Print empty line to transmit termination of recording
    Serial.println();

  // Make sure the button has been released for a few milliseconds
#if BTN_START
  while (digitalRead(BTN_PIN) == 0);
  delay(100);
#endif
}
```

Data collection python code

```
#!/usr/bin/env python
"""
Serial Data Collection CSV

Collects raw data in CSV form over a serial connection and saves them to files.

Install dependencies:

    python -m pip install pyserial
cle
The first line should be header information. Each sample should be on a newline.
Here is a raw accelerometer data sample (in m/s^2):

    accX,accY,accZ
    -0.22,0.82,10.19
    -0.05,0.77,9.63
    -0.01,1.10,8.50
    ...

The end of the sample should contain an empty line (e.g. \r\n\r\n).

Call this script as follows:

    python serial-data-collect-csv.py
    
Author: Shawn Hymel (EdgeImpulse, Inc.)
Date: June 17, 2022
License: Apache-2.0 (apache.org/licenses/LICENSE-2.0)
"""

import argparse
import os
import time

# Third-party libraries
import serial
import serial.tools.list_ports

# Settings
DEFAULT_BAUD = 115200       # Must match transmitting program baud rate
DEFAULT_LABEL = "_unknown"  # Label prepended to all CSV files

# Create a file with unique filename and write CSV data to it
def write_csv(data, dir, label):

    # Keep trying if the file exists
    exists = True
    while exists:

        # Unique ID is epoch time in ms
        uid = str(round(time.time() * 1000))
        filename = label + "." + uid + ".csv"
        
        # Create and write to file if it does not exist
        out_path = os.path.join(dir, filename)
        if not os.path.exists(out_path):
            exists = False
            try:
                with open(out_path, 'w') as file:
                    file.write(data)
                print("Data written to:", out_path)
            except IOError as e:
                print("ERROR", e)
                return
    

# Command line arguments
parser = argparse.ArgumentParser(description="Serial Data Collection CSV")
parser.add_argument('-p',
                    '--port',
                    dest='port',
                    type=str,
                    required=True,
                    help="Serial port to connect to")
parser.add_argument('-b',
                    '--baud',
                    dest='baud',
                    type=int,
                    default=DEFAULT_BAUD,
                    help="Baud rate (default = " + str(DEFAULT_BAUD) + ")")
parser.add_argument('-d',
                    '--directory',
                    dest='directory',
                    type=str,
                    default=".",
                    help="Output directory for files (default = .)")
parser.add_argument('-l',
                    '--label',
                    dest='label',
                    type=str,
                    default=DEFAULT_LABEL,
                    help="Label for files (default = " + DEFAULT_LABEL + ")")
                    
# Print out available serial ports
print()
print("Available serial ports:")
available_ports = serial.tools.list_ports.comports()
for port, desc, hwid in sorted(available_ports):
    print("  {} : {} [{}]".format(port, desc, hwid))
    
# Parse arguments
args = parser.parse_args()
port = args.port
baud = args.baud
out_dir = args.directory
label = args.label

# Configure serial port
ser = serial.Serial()
ser.port = port
ser.baudrate = baud

# Attempt to connect to the serial port
try:
    ser.open()
except Exception as e:
    print("ERROR:", e)
    exit()
print()
print("Connected to {} at a baud rate of {}".format(port, baud))
print("Press 'ctrl+c' to exit")

# Serial receive buffer
rx_buf = b''

# Make output directory
try:
    os.makedirs(out_dir)
except FileExistsError:
    pass

# Loop forever (unless ctrl+c is captured)
try:
    while True:
        
        # Read bytes from serial port
        if ser.in_waiting > 0:
            while(ser.in_waiting):
                
                # Read bytes
                rx_buf += ser.read()
                
                # Look for an empty line
                if rx_buf[-4:] == b'\r\n\r\n':

                    # Strip extra newlines (convert \r\n to \n)
                    buf_str = rx_buf.decode('utf-8').strip()
                    buf_str = buf_str.replace('\r', '')

                    # Write contents to file
                    write_csv(buf_str, out_dir, label)
                    rx_buf = b''

# Look for keyboard interrupt (ctrl+c)
except KeyboardInterrupt:
    pass

# Close serial port
print("Closing serial port")
ser.close()
```



Final ai_fightfire code

```#include <SHT1x.h>
#include <Wire.h>
#include "TFT_eSPI.h"   
#include <SensirionI2CSht4x.h>
#include "FightFire-Zain-Project_inferencing.h"                

#include <Arduino.h>
#include<SoftwareSerial.h>
SoftwareSerial e5(0, 1);
static char recv_buf[512];
int counter = 0;
static int at_send_check_response(char *p_ack, int timeout_ms, char *p_cmd, ...)
{
  int ch;
  int num = 0;
  int index = 0;
  int startMillis = 0;
  va_list args;
  memset(recv_buf, 0, sizeof(recv_buf));
  va_start(args, p_cmd);
  e5.print(p_cmd);
  Serial.print(p_cmd);
  va_end(args);
  delay(200);
  startMillis = millis();
  if (p_ack == NULL)
 return 0;
  do
 {
   while (e5.available() > 0)
 {
  ch = e5.read();
  recv_buf[index++] = ch;
  Serial.print((char)ch);
  delay(2);
 }
  if (strstr(recv_buf, p_ack) != NULL)
 return 1;
 }  
  while (millis() - startMillis < timeout_ms);
  Serial.println();
 return 0;
}

#define dataPin 6
#define clockPin 8
SHT1x sht1x(dataPin, clockPin);
SensirionI2CSht4x sht4x;

// Settings
#define DEBUG               1                         // 1 to print out debugging info
#define DEBUG_NN            false                     // Print out EI debugging info
#define ANOMALY_THRESHOLD   0.3                       // Scores above this are an "anomaly"
#define SAMPLING_FREQ_HZ    4                         // Sampling frequency (Hz)
#define SAMPLING_PERIOD_MS  1000 / SAMPLING_FREQ_HZ   // Sampling period (ms)
#define NUM_SAMPLES         EI_CLASSIFIER_RAW_SAMPLE_COUNT  // 4 samples at 4 Hz
#define READINGS_PER_SAMPLE EI_CLASSIFIER_RAW_SAMPLES_PER_FRAME // 8

// Preprocessing constants (drop the timestamp column)
float mins[] = {
  29.06, 60.93, 28.88, 13.46
};
float ranges[] = {
  50.17, 16.45, 45.18, 64.8
};

TFT_eSPI tft;   // Wio Terminal LCD

void setup() {
  // Start serial
  Serial.begin(9600);
  e5.begin(9600);
  Serial.print("E5 LOCAL TEST\r\n");
  at_send_check_response("+TEST: RFCFG", 1000, "AT+TEST=RFCFG,866,SF12,125,12,15,14,ON,OFF,OFF\r\n");
  delay(200);
    Wire.begin();
    uint16_t error;
    char errorMessage[256];

    sht4x.begin(Wire);

  // Configure LCD
  tft.begin();
  tft.setRotation(3);
  tft.setFreeFont(&FreeSansBoldOblique24pt7b);
  tft.fillScreen(TFT_BLACK);
}

void loop() {
  uint16_t error;
  unsigned long timestamp;
  char errorMessage[256];
  static float raw_buf[NUM_SAMPLES * READINGS_PER_SAMPLE];
  static signal_t signal;
  float temp;
  int max_idx = 0;
  float max_val = 0.0;
  char str_buf[40];
  // Collect samples and perform inference
  for (int i = 0; i < NUM_SAMPLES; i++) {

    // Take timestamp so we can hit our target frequency
    timestamp = millis();

  float soil_temp;
  float soil_humidity;
  float atmtemp;
  float atmhum;
  error = sht4x.measureHighPrecision(atmtemp,atmhum); 
  // Read values from the sensor
  soil_temp = sht1x.readTemperatureC();
  soil_humidity = sht1x.readHumidity();
  

    // Store raw data into the buffer
    raw_buf[(i * READINGS_PER_SAMPLE) + 0] = soil_temp;
    raw_buf[(i * READINGS_PER_SAMPLE) + 1] = soil_humidity;
    raw_buf[(i * READINGS_PER_SAMPLE) + 2] = atmtemp;
    raw_buf[(i * READINGS_PER_SAMPLE) + 3] = atmhum;
    
    // Perform preprocessing step (normalization) on all readings in the sample
    for (int j = 0; j < READINGS_PER_SAMPLE; j++) {
      temp = raw_buf[(i * READINGS_PER_SAMPLE) + j] - mins[j];
      raw_buf[(i * READINGS_PER_SAMPLE) + j] = temp / ranges[j];
    }

    // Wait just long enough for our sampling period
    while (millis() < timestamp + SAMPLING_PERIOD_MS);
  }

  // Print out our preprocessed, raw buffer
#if DEBUG
  for (int i = 0; i < NUM_SAMPLES * READINGS_PER_SAMPLE; i++) {
    Serial.print(raw_buf[i]);
    if (i < (NUM_SAMPLES * READINGS_PER_SAMPLE) - 1) {
      Serial.print(", ");
    }
  }
  Serial.println();
#endif

  // Turn the raw buffer in a signal which we can the classify
  int err = numpy::signal_from_buffer(raw_buf, EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE, &signal);
  if (err != 0) {
      ei_printf("ERROR: Failed to create signal from buffer (%d)\r\n", err);
      return;
  }

  // Run inference
  ei_impulse_result_t result = {0};
  err = run_classifier(&signal, &result, DEBUG_NN);
  if (err != EI_IMPULSE_OK) {
      ei_printf("ERROR: Failed to run classifier (%d)\r\n", err);
      return;
  }

 
  for (int i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
    ei_printf("\t%s: %.3f\r\n",result.classification[i].label,result.classification[i].value);
  }


  // Find maximum prediction
  for (int i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
    if (result.classification[i].value > max_val) {
      max_val = result.classification[i].value;
      max_idx = i;
    }
  }
  tft.fillScreen(TFT_BLACK);
   tft.drawString(result.classification[max_idx].label, 20, 60);
    sprintf(str_buf, "%.3f", max_val);
    tft.drawString(str_buf, 60, 120);

char cmd[128];   
sprintf(cmd, "AT+TEST=TXLRSTR,\"%s\"\r\n",result.classification[max_idx].label);
int ret = at_send_check_response("TX DONE", 5000, cmd);
    if (ret == 1)
    {
        Serial.print("Sent successfully!\r\n");
    }
    else
    {
        Serial.print("Send failed!\r\n");
    }

}
```
![image](https://user-images.githubusercontent.com/30889567/192161535-a8f1eea8-8b69-4733-9577-27ab0ce9d249.png)
