# imu-mpu9250
The repo provides a bridge between MPU9250 and raspberry pi. It also lists various caliberation code and filters for getting an accurate orientation from MPU9250
This repo mostly concentrates on the problem of connecting IMU(MPU9250) to raspberry pi through I2C communication. 
# Pre-requisites
Some of the requirements are to enable I2C in rpi. 
Installing I2C tools and smbus
```bash
sudo apt-get install i2c-tools
sudo pip install smbus
```
Connect the MPU9250 with rpi using the below connections
| Rpi pin | MPU9250 pins |
| ------ | ---------|
| pin 3 ->| SDA pin |
| pin 5 ->| SCL pin |
| pin 6 ->| Ground(GND) |
| pin 1 ->| VCC |

After you have made the connections, type the following command - 
```bash
sudo i2cdetect -y 1
```
If you see 68 in the output, then that means the sensor is connected to the rpi and 68 is the address of the sensor. 

# Basic Usage
The below code is a basic starter for the library
```python
import smbus
import numpy as np
from MPU9250 import MPU9250
address = 0x68
bus = smbus.SMBus(1)
imu = MPU9250.MPU9250(bus, address)
imu.begin()
imu.caliberateGyro()
imu.caliberateAccelerometer()
# or load your own caliberation file
#imu.loadCalibDataFromFile("/home/pi/calib.json")

while True:
	imu.readSensor()
	accel_vals = imu.AccelVals() # 3*1 array with x-axis, y-axis and z-axis
  gyro_vals = imu.GyroVals() # 3*1 array with x-axis, y-axis and z-axis
  mag_vals = imu.MagVals() # 3*1 array with x-axis, y-axis and z-axis

	print ("Acc: {0} ; Gyro : {1} ; Mag : {2}".format(accel_vals, gyro_vals, mag_vals))
	time.sleep(0.1)

```
# Other Functionalities

## Setting Accelerometer Range
The accelerometer in MPU9250 has the following ranges of +-2g, +-4g, +-8g and +-16g  
You can set this setting by the below command
```python
imu.setAccelRange("AccelRangeSelect2G")
```
Simiarly for 4g use "AccelRangeSelect4G" and follow similary for 8g and 16g ranges.

## Setting Gyroscope Range
Gyroscope sensor in MPU9250 has the following ranges +-250DPS, +-500DPS, +-1000DPS and +-2000DPS  
You can set this setting by the below command
```python
imu.setGyroRange("GyroRangeSelect250DPS")
```
Simiarly for 500DPS use "GyroRangeSelect500DPS" and follow similary for 1000DPS and 2000DPS ranges.  
**Note:** DPS means degrees per second

## Setting internal low pass filter frequency
The sensor has an internal low pass filter to remove some basic noise in the values generated by accelerometer and gyrscope.  
Use the following command 
```python
imu.setLowPassFilterFrequency(AccelLowPassFilter184)
```
|frequency | str |
|---------| ----|
| 5Hz| AccelLowPassFilter5|
| 10Hz| AccelLowPassFilter10|
| 20Hz| AccelLowPassFilter20|
| 41Hz| AccelLowPassFilter41|
| 92Hz| AccelLowPassFilter92|
| 184Hz| AccelLowPassFilter184|

## Gyroscope Caliberation
Though most sensors are caliberated during manufacturing, however, there still could be a need for caliberation due to various cahnges like being soldered to a breakout board. Gyroscope normally comes with a bias. This can be found by averaging the values when it is kept still and then subtract those values to get the appropriate values.
```python
imu.caliberateGyro()
```
This will calculate the bias and it is stored in ```imu.GyroBias```
You can also set its value, but make sure you give 3x1 numpy array.

## Accelerometer Caliberation
This caliberation includes an extra parameter called scale apart from bias. Use the below command
```python
imu.caliberateAccelerometer()
```
The above function will store the scale and bias in the following variables ```imu.Accels``` and ```imu.AccelBias``` respectively.

## Magnometer Caliberation
This has two types of caliberation 
* ```imu.caliberateMagApprox()``` : As the name suggests, this is a near approximation of scale and bias parameters. It saves time however, might not be always accurate. In this the scale and bias are stored in ```imu.Mags``` and ```imu.MagBias``` respectively.
* ```imu.caliberateMagPrecise()``` : It tries to fit the data to an ellipsoid and is more complicated and time consuming. It gives a 3x3 symmetric transformation matrix(```imu.Magtransform```) instead of a common 3x1 scale values. The bias variable is ```imu.MagBias```  
For more details on this, have a look at mag_caliberation folder in examples. 
## IMU Orientation
The computed orientation is in terms of eurler angles. roll for x axis, pitch for y axis and yaw for z axis. We use NED format which basically means, the sensor's x-axis is aligned with north, sensor's y-axis is aligned with east and sensor's x-axis is aligned with down. 
```imu.computeOrientation()```
The roll, pitch and yaw can be accessed by ```imu.roll```, ```imu.pitch``` and ```imu.yaw```.
**Note:** The euler angles will only make sense when all the sensors are properly caliberated.

# Filters for sensorfusion
Orientation from accelerometer and magnetometer are noisy, while estimating orientation from gyroscope is noise free but accumulates drift over time. We will combining both of these to obtain more stable orientation. There are multiple ways to do it and we have given two options of kalman and madgwick. You are free to write your own algorithms. 

## Kalman
It uses gyroscope to estimate the new state. Accelerometer and magnetometer provide the new measured state. The kalman filter aims to find a corrected state from the above two by assuming that both are forms of gaussian distributions.
look at kalmanExample.py in examples
```python
kalman.computeAndUpdateRollPitchYaw(imu.AccelVals[0], imu.AccelVals[1], imu.AccelVals[2], imu.GyroVals[0], imu.GyroVals[1], imu.GyroVals[2],\
												imu.MagVals[0], imu.MagVals[1], imu.MagVals[2], dt)

```

## Madgwick
This is slightly better than kalman and more smooth in giving out the orientation. However, for this to work properly, the sensor fusion needs to run at least 10 times faster frequency than the sensor sampling frequency. 
look at madgwickExample.py in examples
```python
for i in range(10):
		newTime = time.time()
		dt = newTime - currTime
		currTime = newTime

		sensorfusion.updateRollPitchYaw(imu.AccelVals[0], imu.AccelVals[1], imu.AccelVals[2], imu.GyroVals[0], \
									imu.GyroVals[1], imu.GyroVals[2], imu.MagVals[0], imu.MagVals[1], imu.MagVals[2], dt)
```
For the detailed explanation -> [link](https://www.x-io.co.uk/res/doc/madgwick_internal_report.pdf)

## Filter comparison
We have also done a small filter comparison of all the filters. This data can be streamed to your computer using zmq and also you can visualize the imu orientation using pygame_viz.py in examples/filters_comparison. 
