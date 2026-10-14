# Python data logger

`log_arduino_data.py` collects data from the Arduino and saves every reading with a timestamp, to a CSV file. This page explains what each part of the script does.

## The big picture

The Arduino sends two lines of text for every reading:

```
Temperature (°C): 22.27
Light Intensity (Raw): 998
```

The script reads these lines, pairs the temperature with the light value that follows it, adds the time (as a timestamp), and writes one row to the file. Then it waits for the next reading.

## Before you run it

Install the one library it needs, then check the three settings in the table below.

```
pip install -r python/requirements.txt
```

| Setting | Where | What to change |
|---|---|---|
| `SERIAL_PORT` | Near the top | Your Arduino's port (for example `COM4` on Windows). You can see it under **Tools → Port** in the Arduino IDE. |
| `log_folder` | Near the top | The folder where the CSV will be saved. |
| `while count < 122` | Inside the loop | The number of readings to record before stopping. |

## What each part does

### 1. Imports

```python
import serial
import time
import csv
import os
```

| Library | Used for |
|---|---|
| `serial` | Talking to the Arduino through the USB port (this is the `pyserial` library) |
| `time` | Pausing the script and creating timestamps |
| `csv` | Writing the rows in CSV format |
| `os` | Checking whether the log folder exists, and creating it |

### 2. Settings

```python
SERIAL_PORT = 'COM4'
BAUD_RATE = 9600
```

The port tells the script which device to listen to. The baud rate is the speed of the conversation, and it **must match** the number in the Arduino sketch (`Serial.begin(9600)`). If the two do not match, you will see unreadable symbols instead of readings.

### 3. The log folder and file

```python
log_folder = r'C:\DataLogs'
if not os.path.exists(log_folder):
    os.makedirs(log_folder)
LOG_FILE = os.path.join(log_folder, 'sensor_data.csv')
```

The script checks whether the folder exists and creates it if it does not. The readings are saved in `sensor_data.csv` inside that folder.

### 4. Opening the serial connection

```python
ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
time.sleep(2)
```

This opens the connection to the Arduino. `timeout=1` means that a read gives up after one second if nothing has arrived. The two-second pause gives the connection time to start up before the script begins listening.

### 5. Opening the CSV file

```python
with open(LOG_FILE, mode='a', newline='') as file:
    writer = csv.writer(file)

    if file.tell() == 0:
        writer.writerow(["Timestamp", "Temperature (°C)", "Light Intensity (Raw)"])
```

- `mode='a'` means *append*. New readings are added to the end of the file, so running the script again does not erase your earlier data.
- `newline=''` stops the `csv` library from adding blank lines between rows on Windows.
- `file.tell() == 0` checks whether the file is empty. If it is, the script writes the column names first. This way the header is written only once.

### 6. The logging loop

```python
count = 0
while count < 122:
    if ser.in_waiting > 0:
        line = ser.readline().decode('utf-8').strip()
```

The loop keeps going until 122 readings are saved (about an hour at one reading every 30 seconds, but this can of course be changed as per your needs). Each time round:

- `ser.in_waiting > 0` checks whether the Arduino has sent anything.
- `ser.readline()` reads one line of text.
- `.decode('utf-8')` turns the raw bytes into text.
- `.strip()` removes the invisible end-of-line characters.

You can change `122` to a smaller number for a quick test.

### 7. Reading the numbers out of the text

```python
if line.startswith("Temperature (°C):"):
    temperature = float(line.split(": ")[1])
elif line.startswith("Light Intensity (Raw):"):
    light_intensity = int(line.split(": ")[1])
```

The script looks at how each line begins to decide what it is. Then `line.split(": ")` cuts the line in two at the colon and space, and `[1]` takes the second piece, which is the number. For example:

| Step | Result |
|---|---|
| The line | `Temperature (°C): 22.27` |
| After `split(": ")` | `["Temperature (°C)", "22.27"]` |
| The second piece, `[1]` | `"22.27"` (still text) |
| After `float(...)` | `22.27` (a number) |

The temperature is a decimal number, so it uses `float`. The raw light value is a whole number, so it uses `int`.

### 8. Saving the reading

```python
timestamp = time.strftime("%Y-%m-%d %H:%M:%S")

writer.writerow([timestamp, temperature, light_intensity])
print(f"Logged: {timestamp}, {temperature}°C, {light_intensity}")
count += 1
```

This part sits inside the light-intensity branch. The Arduino sends the temperature data first and the light value after that, so the script holds on to the temperature and only writes the row once the light value arrives. That is how each row gets one temperature and one light reading from the same moment.

The timestamp is the time on your laptop when the line arrived. `%Y-%m-%d %H:%M:%S` gives a format like `2025-02-27 13:28:02`. The script also prints each reading on screen so you can follow along, and adds 1 to the count.

### 9. Handling problems and stopping

```python
except serial.SerialException as e:
    print(f"Serial port error: {e}")
    print("Ensure the correct port is selected and no other program is using it.")
except KeyboardInterrupt:
    print("Logging stopped.")
finally:
    if 'ser' in locals() and ser.is_open:
        ser.close()
```

- **`SerialException`:** if the port cannot be opened, the script prints a helpful message instead of crashing. The usual causes are the wrong port name, or the Serial Monitor still being open.
- **`KeyboardInterrupt`:** this happens when you press **Ctrl+C**. The script stops cleanly.
- **`finally`:** this part runs whatever happened. It closes the serial port so that other programs, such as the Arduino IDE, can use it again.

## What you will see

```
Logging data. Press Ctrl+C to stop...
Logged: 2025-02-27 13:28:02, 21.78°C, 1007
Logged: 2025-02-27 13:28:32, 22.27°C, 1000
```


