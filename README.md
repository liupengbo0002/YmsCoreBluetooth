# YmsCoreBluetooth 0.93 (beta)
A framework for building Bluetooth 4.0 Low Energy (aka Smart or LE) iOS applications using the CoreBluetooth API. Includes Deanna, an iOS application using YmsCoreBluetooth to communicate with a [TI SensorTag](http://processors.wiki.ti.com/index.php/Bluetooth_SensorTag).

* [YmsCoreBluetooth API Reference](http://kickingvegas.github.io/YmsCoreBluetooth/appledoc/index.html)

## YmsCoreBluetooth Design Intent: Or Why You Want To Use This Framework
### tl;dr 
* ObjectiveC Block-based API for Bluetooth LE communication.
* Operations (e.g. scanning, retrieval, connection, reads, writes) map to the data object hierarchy of CoreBluetooth.

### A More Detailed Explaination

#### Blocks are cool.
Transactions in Bluetooth LE (BLE) are two-phase (request-response) in nature: CoreBluetooth abstracts this protocol so that request behavior is separated from response behavior. The two phases are reconciled using a delegation pattern: the object initiating the request phase has a delegate object with a delegate method to handle the corresponding response phase. While functional, the delegation pattern can be cumbersome to use because the behavior for a two-phase transaction is split into two different locations in code.

A more convenient programming pattern is to use a callback block which is defined with the request. When the response is received, the callback block can be executed to handle it. *The design intent of YmsCoreBluetooth is use Objective-C blocks to define response behavior to BLE requests*. Such requests include:

* scanning and/or retrieving peripheral(s)
* connecting to a peripheral
* discovering a peripheral's services
* discovering a service's characteristics
* write and read of a characteristic
* setting the notification status of a characteristic

#### Hierarchical operations are cool.
The data object hierachy of CoreBluetooth can be described as such:

* A CBCentralManager instance can connect to multiple CBPeripheral instances.
    * A CBPeripheral instance can have multiple CBService instances.
	    * A CBService instance can have multiple CBCharacteristic instances.
		    * A CBCharacteristic instance can have multiple CBDescriptor instances.
			
However the existing CoreBluetooth API does not map BLE requests to the data object hierarchy. For example connection to a CBPeripheral instance is accomplished from a CBCentralManager instance instead of from a CBPeripheral. Writes, reads, and setting the notification state of a CBCharacteristic are issued from a CBPeripheral instance, instead of from CBCharacteristic. *YmsCoreBluetooth provides an API that more naturally maps operations to the data object hierarchy*.

YMSCoreBluetooth defines container classes which map to the CoreBluetooth object hierarchy:

* YMSCBCentralManager - A BLE central which manages the BLE peripherals it either discovers via scanning or retrieval.
    * YMSCBPeripheral - A BLE peripheral can have multiple BLE services
        * YMSCBService - A BLE service can have multiple BLE characteristics
             * YMSCBCharacteristic - A BLE characteristic can have multiple BLE descriptors
			     * YMSCBDescriptor - A BLE descriptor (TBD)



However, they differ from CoreBluetooth in that operations are done with respect to the object type:

* YMSCBCentralManager
  - scans for peripherals
  - retrieves peripherals
* YMSCBPeripheral
  - connects and disconnects to central
  - discovers services associated with this peripheral
* YMSCBService
  - discovers characteristics associated with this service
  - handles notification updates for characteristics which are set to notify
* YMSCBCharacteristic
  - set notification status (on, off)
  - write value to characteristic
  - read value of characteristic
  - discover descriptors associated with this characteristic (TBD)
  
  
### Show Code

#### Scanning for Peripherals

In the following code sample, `self` is an instance of a subclass of YMSCBCentralManager.

    [self scanForPeripheralsWithServices:nil
                                 options:options
                               withBlock:^(CBPeripheral *peripheral, NSDictionary *advertisementData, NSNumber *RSSI, NSError *error) {
                                   if (error) {
                                       NSLog(@"Something bad happened with scanForPeripheralWithServices:options:withBlock:");
                                       return;
                                   }
                                   
                                   NSLog(@"DISCOVERED: %@, %@, %@ db", peripheral, peripheral.name, RSSI);
                                   [this handleFoundPeripheral:peripheral];
                               }];

#### Retrieving Peripherals

In the following code sample, `self` is an instance of a subclass of YMSCBCentralManager.

	[self retrievePeripherals:peripheralUUIDs
					withBlock:^(CBPeripheral *peripheral) {
						[this handleFoundPeripheral:peripheral];
					}];
  
  
#### Connecting to a Peripheral

In the following code sample, `self` is an instance of a subclass of YMSCBPeripheral. Note that in the callbacks, discovering services and characteristics are handled in a nested fashion:

	- (void)connect {
		// Watchdog aware method
		[self resetWatchdog];

		[self connectWithOptions:nil withBlock:^(YMSCBPeripheral *yp, NSError *error) {
			if (error) {
				return;
			}

			[yp discoverServices:[yp services] withBlock:^(NSArray *yservices, NSError *error) {
				if (error) {
					return;
				}

				for (YMSCBService *service in yservices) {
					if ([service.name isEqualToString:@"simplekeys"]) {
						__weak DEASimpleKeysService *thisService = (DEASimpleKeysService *)service;
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError 	*error) {
							[thisService turnOn];
						}];

					} else if ([service.name isEqualToString:@"devinfo"]) {
						__weak DEADeviceInfoService *thisService = (DEADeviceInfoService *)service;
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError 	*error) {
							[thisService readDeviceInfo];
						}];

					} else {
						[service discoverCharacteristics:[service characteristics] withBlock:^(NSDictionary *chDict, NSError 	*error) {
						}];
					}
				}
			}];
		}];
	}

#### Read a Characteristic

In the following code sample, `self` is an instance of a subclass of YMSCBService. All discovered characteristics are stored in [YMSCBService characteristicDict].

	- (void)readDeviceInfo {

		YMSCBCharacteristic *system_idCt = self.characteristicDict[@"system_id"];
		__weak DEADeviceInfoService *this = self;
		[system_idCt readValueWithBlock:^(NSData *data, NSError *error) {
			NSMutableString *tmpString = [NSMutableString stringWithFormat:@""];
			unsigned char bytes[data.length];
			[data getBytes:bytes];
			for (int ii = data.length; ii >= 0;ii--) {
				[tmpString appendFormat:@"%02hhx",bytes[ii]];
				if (ii) {
					[tmpString appendFormat:@":"];
				}
			}

			NSLog(@"system id: %@", tmpString);
			this.system_id = tmpString;
		}];

		YMSCBCharacteristic *model_numberCt = self.characteristicDict[@"model_number"];
		[model_numberCt readValueWithBlock:^(NSData *data, NSError *error) {
			if (error) {
				NSLog(@"ERROR: %@", error);
				return;
			}

			NSString *payload = [[NSString alloc] initWithData:data encoding:NSStringEncodingConversionAllowLossy];
			this.model_number = payload;
			NSLog(@"model: %@", payload);

		}];
	}

#### Write to a Characteristic

In the following code sample, `self` is an instance of a subclass of YMSCBService. In this example, a write to the 'config' characteristic followed by a read of the 'calibration' characteristic is done.

	- (void)requestCalibration {
		if (self.isCalibrating == NO) {

			__weak DEABarometerService *this = self;

			YMSCBCharacteristic *configCt = self.characteristicDict[@"config"];
			[configCt writeByte:0x2 withBlock:^(NSError *error) {
				if (error) {
					NSLog(@"ERROR: write request to barometer config to start calibration failed.");
					return;
				}

				YMSCBCharacteristic *calibrationCt = this.characteristicDict[@"calibration"];
				[calibrationCt readValueWithBlock:^(NSData *data, NSError *error) {
					if (error) {
						NSLog(@"ERROR: read request to barometer calibration failed.");
						return;
					}

					this.isCalibrating = NO;
					char val[data.length];
					[data getBytes:&val length:data.length];

					int i = 0;
					while (i < data.length) {
						uint16_t lo = val[i];
						uint16_t hi = val[i+1];
						uint16_t cx = ((lo & 0xff)| ((hi << 8) & 0xff00));
						int index = i/2 + 1;

						if (index == 1) self.c1 = cx;
						else if (index == 2) this.c2 = cx;
						else if (index == 3) this.c3 = cx;
						else if (index == 4) this.c4 = cx;
						else if (index == 5) this.c5 = cx;
						else if (index == 6) this.c6 = cx;
						else if (index == 7) this.c7 = cx;
						else if (index == 8) this.c8 = cx;

						i = i + 2;
					}

					this.isCalibrated = YES;

				}];
			}];

		}
	}


### Handling Characteristic Notification Updates

One place where YmsCoreBluetooth does *not* use blocks to handle BLE responses is with characteristic notification updates. The reason for this is because such updates are asynchronous and non-deterministic. As such the handler method [YMSCBService notifyCharacteristicHandler:error:] must be implemented for any subclass of YMSCBService to handle such updates.

In the following code sample, `self` is an instance of a subclass of YMSCBService. 

	- (void)turnOn {
	    __weak DEABaseService *this = self;
		YMSCBCharacteristic *configCt = self.characteristicDict[@"config"];
		[configCt writeByte:0x1 withBlock:^(NSError *error) {
			if (error) {
				NSLog(@"ERROR: %@", error);
				return;
			}

			NSLog(@"TURNED ON: %@", this.name);
		}];

		YMSCBCharacteristic *dataCt = self.characteristicDict[@"data"];
		[dataCt setNotifyValue:YES withBlock:^(NSError *error) {
		    if (error) {
			    return;
			}
			NSLog(@"Data notification for %@ on", this.name);
		}];

		self.isOn = YES;
	}

	- (void)notifyCharacteristicHandler:(YMSCBCharacteristic *)yc error:(NSError *)error {
		if (error) {
			return;
		}

		if ([yc.name isEqualToString:@"data"]) {
			NSData *data = yc.cbCharacteristic.value;
			char val[data.length];
			[data getBytes:&val length:data.length];

			int16_t v0 = val[0];
			int16_t v1 = val[1];
			int16_t v2 = val[2];
			int16_t v3 = val[3];
			int16_t amb = ((v2 & 0xff)| ((v3 << 8) & 0xff00));
			int16_t objT = ((v0 & 0xff)| ((v1 << 8) & 0xff00));

			double tempAmb = calcTmpLocal(amb);
			
			self.ambientTemp = [NSNumber numberWithDouble:tempAmb];
			self.objectTemp = [NSNumber numberWithDouble:calcTmpTarget(objT, tempAmb)];
		}
	}

### Block Callback Design

The callback pattern used by YmsCoreBluetooth uses a single callback to handle both successfull and failed responses. This is accomplished by including an `error` parameter. If an `error` object is not `nil`, then behavior to handle the failure can be implemented. Otherwise behavior to handle success is implemented.

	^(NSError *error) {
	   if (error) {
		  // Code to handle failure
		  return;
	   }
	   // Code to handle success
	}


## File Naming Conventions
The YmsCoreBluetooth framework is the set of files prefixed with `YMSCB` located in the directory `YmsCoreBluetooth`.

The files for the iOS application **Deanna** are prefixed with `DEA` located in the directory `Deanna`.

## Recommended Code Walk Through

To better understand how YmsCoreBluetooth works, it is recommended to first read the source of the following BLE service implementations:

* DEAAccelerometerService, DEABarometerService, DEAGyroscopeService, DEAHumidityService, DEAMagnetometerService, DEASimpleKeysService, DEATemperatureService, DEADeviceInfoService

Then the BLE peripheral implementation of the TI SensorTag:

* DEASensorTag

Then the application service which manages all known peripherals:

* DEACentralManager

The [Class Hierarchy](hierarchy.html) is very instructive in showing the relationship of the above classes to the YmsCoreBluetooth framework.


## Writing your own Bluetooth LE service with YmsCoreBluetooth

Learn how to write your own Bluetooth LE service by reading the example of how its done for the TI SensorTag in the Tutorial.

## Questions

While quite functional, YmsCoreBluetooth is still very much in an early state and there's always room for improvement. Please submit any questions or [issues to the GitHub project for YmsCoreBluetooth](https://github.com/kickingvegas/YmsCoreBluetooth/issues?labels=&milestone=&page=1&state=open).


## Notes

Code tested on:

* iPhone 4S, iOS 6.1.3
* TI SensorTag firmware 1.2, 1.3.

## Known Issues
* No support is offered for the iOS simulator due to the instability of the CoreBluetooth implementation on it. Use this code only on iOS hardware that supports CoreBluetooth. Given that Apple does not provide technical support for CoreBluetooth behavior on the iOS simulator [TN2295](http://developer.apple.com/library/ios/#technotes/tn2295/_index.html), I feel this is a reasonable position to take. Hopefully in time the iOS simulator will exhibit better CoreBluetooth fidelity.

## Latest Changes
### Sun May 26 2013 - Rare Groove Release (ver 0.93)
* Issue #47 - rename YMSCBAppService to YMSCBCentralManager. 
  Associated changes include:
    - renaming DEAAppService to DEACentralManager
	- rename variable cbAppService to centralManager

* API change to fully embrace ObjectiveC block-based callbacks.
* API change to map BLE operations to data object hierarchy.

**API 0.93 is a BIG CHANGE.** It is largely a rethink of YmsCoreBluetooth to support the pattern for hierarchical block-based operations. This will very much *break code* written for prior releases of YmsCoreBluetooth. That said, the changes should be fairly comprehensible and address the following points going forward:

* Improved understanding of API behavior; prior releases had too much implicit behavior ("magic") which made involuntary decisions for the developer using the API. Using blocks gives the developer a clearer and more structured view of the BLE transactions to be made.
* With the pattern for hierarchical block-based operations in place, a more structured way to add new API features. Like Winter, characteristic descriptors are coming.

View [Prior Change List](https://github.com/kickingvegas/YmsCoreBluetooth/blob/master/CHANGES.md)


### License

    Copyright 2013 Yummy Melon Software LLC

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

    Author: Charles Y. Choi <charles.choi@yummymelon.com>





