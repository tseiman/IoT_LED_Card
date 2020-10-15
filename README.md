# IoT LED Card

IoT card for Sierra Wireless MangOH (https://mangoh.io) boards.
4 Controllable RGB LEDs. The RGB LEDs are driven via a 16 channel RGB controler PCA9685PW (https://www.nxp.com/docs/en/data-sheet/PCA9685.pdf). 
From this 16 channels 3 (RGB) x 4 LEDs = 12 Channels are used.
The IoT card uses the UART and a UART to I2C converter to communicate to the RGB controller.
UART to I2C conversion is done with SC18IM700 (https://www.nxp.com/docs/en/data-sheet/SC18IM700.pdf)

<img src="https://raw.githubusercontent.com/tseiman/IoT_Led_Card/main/IoT_LED_Card_pic2.png"  width="300">

![LED IoT Card](https://raw.githubusercontent.com/tseiman/IoT_Led_Card/main/IoT_LED_Card_pic.png)



![LED IoT Card](https://raw.githubusercontent.com/tseiman/IoT_Led_Card/main/IoT_Led_Card_Board.png)



![LED IoT Card](https://raw.githubusercontent.com/tseiman/IoT_Led_Card/main/Plastic-Parts/LEDDemo%20v8.png)


## Communicating to I2C via the SC18IM700 converter
The address of the RGB driver PCA9685PW is fixed wired to 80h (write) and 81h (read).
The communication via UART is in binary. The UART itself runs in default mode of the SC18IM799 UART<-->I2C converter which is 9600Baud 8N1, no flow control.
For each I2C start command a 53h is send, for a I2C stop a 50h is send.
An example communication to write:
```
0x53 0x80 0x02 0xfa 0x00 0x50 

0x53 --> I2C Start Condition (executed by SC18IM700)
0x80 --> This is the I2C address of the I2C slave
0x02 --> 2 bytes to write (length)
0xfa 0x00 --> the data bytes
0x50 --> let the UART<-->I2C converter SC18IM700 create a I2C stop condition
```

Another Example:

```
// This sets the read pointer to the first register (Register MODE1) 
// of the RGB driver by writing 1 byte (0x01) with the value 0x0.
0x53 0x80 0x01 0x00 0x50

// This reads one byte from the RGB drivers's first register 
0x53 0x81 0x01 0x50
```
Please follow the datasheets of the RGB driver's PCA9685PW and the datasheet of the UART to I2C converter SC18IM700 tightly as the communication with this IoT Card is fully based on it. 

## Example Initialisation
The UART to I2C converter is running with it power up default settings. It is possible to set e.g. baud rate - which is not done at the moment (but everybody who implements a driver or an Octave script is free to do).
However the RGB needs to be initilized (the commands are all documented in the PCA9685 datasheet):

```
						// This both lines reset all LED channels:
0x53 0x80 0x02 0xfa 0x00 0x50     		// addresses register ALL_LED_ON_L (0xfa)
0x53 0x80 0x03 0xfc 0x10 0x00 0x50		// addresses register ALL_LED_OFF_L (0xfc) 
						
0x53 0x80 0x02 0x01 0x10 0x50			// addressing MODE2 register (0x01). 
						// Due to the design of the LED driver 
						// with the LEDs (LEDs are directly connected 
						// to the driver without FETs) 
						// the logic is inverted. This logic negation 
						// can be configured in MODE2 
						// register of RGB driver. For this setup 
						// INVRT bit must be HIGH and 
						// OUTDRV bit must be LOW - see the PCA9685 
						// Rev.4 datasheet p.29, Table 12. for
						// further information

0x53 0x80 0x01 0x00 0x50 0x53 0x81 0x01 0x50	// this reads the first (0x00) MODE1 
						// register by first setting the read 
						// pointer to first (0x00) register 
						// and then read 1 byte. It should 
						// return 0x11.
						// as autoincrement of the register 
						// is disabled by default on power up
						// we can directly write now to 
						// MODE1 register:
0x53 0x80 0x02 0x00 0x31 0x50			// the value 0x31 in MODE1 register
						// which disables RESTART and enables 
						// Register Auto Increment Please 
						// see PCA9685 Rev.4 datasheet p.14, 
						// Table 5. 
						// for further information, the PWM 
						// oscilator is still of at this stage
						

0x53 0x80 0x02 0xFE 0x79 0x50			// This set the PRE_SCALE Register (0xfe) 
						// to 50hz (can be adapted to different freq.)

0x53 0x80 0x02 0x00 0x21 0x50			// Changes in MODE1 register from Low power 
						// mode (Oscilators=Off) to normal mode (Osc.=On)

/* now PCA9685 needs 500us to stabilize the oscilator */
                                    
0x53 0x80 0x02 0x00 0xa1 0x50			// after 500us the initialisation can be finalized,
						// After that we switch in MODE1 register the restart Bit (0xa1) 
						// - after restart the PCA9685 is operational
    
```

## Writing to LED channels

The LED Driver offers 16 channels. Each Channel can be individually controlled with PWM parameters (related to one central Oscilator). 
The time of the duty cycle period and it's offset to the Oscilator clock can be defined by various registers.
Each channel has a LED_OFF_H, LED_OFF_L and a LED_ON_H, LED_ON_L register. The Low (L) registers are 8bit, from the High register only 4 Bits are used so in total a range of 0-4095 can be achived.
In this example the OFF registers are always on 0x0 (please check the PCA9685 datasheet for more in depth information). 

An example to set the brightnes of one channel:

```
53 80 03 06 00 00 50    53 80 03 08 ff 0f 50  	// The LED0 channel (first LED, Red) 
						// the LED_OFF Register is at address 0x06
//  LED0_OFF to 0x0	 LED0_ON to 4095 (max)	// the LED0 channel LED_ON Register is at 0x08
						// This sequence sets LED0 channel 
						// to maximum brightness.
						// Note that the LOW byte is transfered 
						// first - then the HIGH byte

```

## LED Channel Mapping:

| LED on IoT Card | Channel | Channel on PCA9685 | PCA9685 LED_OFF Addr. | PCA9685 LED_ON Addr. |
|:---------------:|:-------:|:------------------:|:---------------------:|:--------------------:|
| 0		  | R	    | 0			 | 0x06			 | 0x08			|
| 0		  | G	    | 1			 | 0x0a			 | 0x0c			|
| 0		  | B	    | 2			 | 0x0e			 | 0x10			|
| 		  | 	    | 			 | 			 | 			|
| 1		  | R	    | 3			 | 0x12			 | 0x14			|
| 1		  | G	    | 4			 | 0x16			 | 0x18			|
| 1		  | B	    | 5			 | 0x1a			 | 0x1c			|
| 		  | 	    | 			 | 			 | 			|
| 2		  | R	    | 6			 | 0x1e			 | 0x20			|
| 2		  | G	    | 7			 | 0x22			 | 0x24			|
| 2		  | B	    | 8			 | 0x26			 | 0x28			|
| 		  | 	    | 			 | 			 | 			|
| 3		  | R	    | 9			 | 0x2a			 | 0x2c			|
| 3		  | G	    | 10		 | 0x2e			 | 0x30			|
| 3		  | B	    | 12		 | 0x32			 | 0x34			|


## Octave Example Code

### Initialisation 

Initialisation, Requires Counter Resource enabled, the Edge Action triggers on a Counter Observation 

```javascript
function(event) {

  if(event.value === 2) { // Huston we have a startup, triggering the first initialisation
    return {
      "dh://lcd/txt1": ["Init I2C"], 
      "vr://LED_I2C_is_init": [ false ], 
      "vr://ledI2CInitStep": [ 3 ], 
    }
  }



  var initDone = Datahub.read("/virtual/LED_I2C_is_init/value",0).value;

  if(initDone) {  // we're initialized don't do anything further
    return {"dh://lcd/txt1": ["no init"] }
  }
  var initStepValue = Datahub.read("/virtual/ledI2CInitStep/value",0).value; // if init is needed reading the counter value when init needs to be started

  var pca9685Init1 = { "data":  [
    0x53, 0x80, 0x02, 0xfa, 0x00, 0x50,
    0x53, 0x80, 0x03, 0xfc, 0x10, 0x00, 0x50,
    0x53, 0x80, 0x02, 0x01, 0x10, 0x50,
    0x53, 0x80, 0x01, 0x00, 0x50, 0x53, 0x81, 0x01, 0x50,
    0x53, 0x80, 0x02, 0x00, 0x31, 0x50,
    0x53, 0x80, 0x02, 0xFE, 0x79, 0x50,
    0x53, 0x80, 0x02, 0x00, 0x21, 0x50
  ]};

 var pca9685Init2 = { "data":  [
    0x53, 0x80, 0x02, 0x00, 0xa1, 0x50,
    0x53, 0x80, 0x03, 0x06, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x08, 0x00, 0x00, 0x50,  
    0x53, 0x80, 0x03, 0x0a, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x0c, 0x00, 0x00, 0x50,  
    0x53, 0x80, 0x03, 0x0e, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x10, 0x00, 0x00, 0x50,  
  ]};


  if(event.value === initStepValue) { // Huston we have a startup, sending the first initialisation
    return {
      "dh://lcd/txt1": ["Running init seq"],                                                                              
      "dh://usp/write": [pca9685Init1],
    }

  }
  if(event.value === (initStepValue + 1)) { // actually we need to wait 500us until the PCM clock of the PCA9685 stabilizes - then we finalize, here we wait until the next couter incement (a bit longer than 500us) 
    return {
      "dh://lcd/txt1": ["DONE Running init seq"],                                                                              
      "dh://usp/write": [pca9685Init2],
      "vr://LED_I2C_is_init": [ true ], 
    }

  }
  
  return {  
     "dh://lcd/txt1": ["wait for init:" + event.value] 

  }



}


```


Writing brightness information:

```javascript
function(event) {

	

	if(event.value.type === "reset") {
      var initStepValue = Datahub.read("/util/counter/value",0).value + 1; 
		
      return {
        "dh://lcd/txt1": ["Init I2C"], 
        "vr://LED_I2C_is_init": [ false ], 
        "vr://ledI2CInitStep": [ initStepValue ], 
      }
  	}


	if(event.value.type === "setled") {

      var data = new Array();

	    if(event.value.hasOwnProperty("LED_0")){
	        var ledVal = event.value.LED_0;
	        if(ledVal.hasOwnProperty("R")){
	            var LEDvalue = ledVal.R;
	            var LED_Red_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Red_L = (LEDvalue & 0xff);
	            var ledCtrl = [0x53, 0x80, 0x03, 0x06, 0x00, 0x00, 0x50, 0x53, 0x80, 0x03, 0x08, LED_Red_L,   LED_Red_H,   0x50];
	            data.push.apply(data,ledCtrl);
	        }
	        if(ledVal.hasOwnProperty("G")){
	            var LEDvalue = ledVal.G;
	            var LED_Green_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Green_L = (LEDvalue & 0xff);
	        
	            var ledCtrl = [0x53, 0x80, 0x03, 0x0a, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x0c, LED_Green_L, LED_Green_H, 0x50];
	            data.push.apply(data,ledCtrl);
	        }
	        if(ledVal.hasOwnProperty("B")){

	            var LEDvalue = ledVal.B;
	            var LED_Blue_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Blue_L = (LEDvalue & 0xff);
	        
	            var ledCtrl = [ 0x53, 0x80, 0x03, 0x0e, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x10, LED_Blue_L,  LED_Blue_H,  0x50];
	            data.push.apply(data,ledCtrl);
	        }
	    }


	    if(event.value.hasOwnProperty("LED_1")){
	        var ledVal = event.value.LED_1;
	        if(ledVal.hasOwnProperty("R")){
	            var LEDvalue = ledVal.R;
	            var LED_Red_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Red_L = (LEDvalue & 0xff);
	            var ledCtrl = [0x53, 0x80, 0x03, 0x12, 0x00, 0x00, 0x50, 0x53, 0x80, 0x03, 0x14, LED_Red_L,   LED_Red_H,   0x50];
	            data.push.apply(data,ledCtrl);
	        }
	        if(ledVal.hasOwnProperty("G")){
	            var LEDvalue = ledVal.G;
	            var LED_Green_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Green_L = (LEDvalue & 0xff);
	        
	            var ledCtrl = [0x53, 0x80, 0x03, 0x16, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x18, LED_Green_L, LED_Green_H, 0x50];
	            data.push.apply(data,ledCtrl);
	        }
	        if(ledVal.hasOwnProperty("B")){

	            var LEDvalue = ledVal.B;
	            var LED_Blue_H = (LEDvalue & 0x0fff) >> 8;
	            var LED_Blue_L = (LEDvalue & 0xff);
	        
	            var ledCtrl = [ 0x53, 0x80, 0x03, 0x1a, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x1c, LED_Blue_L,  LED_Blue_H,  0x50];
	            data.push.apply(data,ledCtrl);
	        }
	    }

		if(event.value.hasOwnProperty("LED_2")){
			var ledVal = event.value.LED_2;
			if(ledVal.hasOwnProperty("R")){
			    var LEDvalue = ledVal.R;
			    var LED_Red_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Red_L = (LEDvalue & 0xff);
			    var ledCtrl = [0x53, 0x80, 0x03, 0x1e, 0x00, 0x00, 0x50, 0x53, 0x80, 0x03, 0x20, LED_Red_L,   LED_Red_H,   0x50];
			    data.push.apply(data,ledCtrl);
			}
			if(ledVal.hasOwnProperty("G")){
			    var LEDvalue = ledVal.G;
			    var LED_Green_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Green_L = (LEDvalue & 0xff);

			    var ledCtrl = [0x53, 0x80, 0x03, 0x22, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x24, LED_Green_L, LED_Green_H, 0x50];
			    data.push.apply(data,ledCtrl);
			}
			if(ledVal.hasOwnProperty("B")){

			    var LEDvalue = ledVal.B;
			    var LED_Blue_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Blue_L = (LEDvalue & 0xff);

			    var ledCtrl = [ 0x53, 0x80, 0x03, 0x26, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x28, LED_Blue_L,  LED_Blue_H,  0x50];
			    data.push.apply(data,ledCtrl);
			}
		}


		if(event.value.hasOwnProperty("LED_3")){
			var ledVal = event.value.LED_3;
			if(ledVal.hasOwnProperty("R")){
			    var LEDvalue = ledVal.R;
			    var LED_Red_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Red_L = (LEDvalue & 0xff);
			    var ledCtrl = [0x53, 0x80, 0x03, 0x2a, 0x00, 0x00, 0x50, 0x53, 0x80, 0x03, 0x2c, LED_Red_L,   LED_Red_H,   0x50];
			    data.push.apply(data,ledCtrl);
			}
			if(ledVal.hasOwnProperty("G")){
			    var LEDvalue = ledVal.G;
			    var LED_Green_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Green_L = (LEDvalue & 0xff);

			    var ledCtrl = [0x53, 0x80, 0x03, 0x2e, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x30, LED_Green_L, LED_Green_H, 0x50];
			    data.push.apply(data,ledCtrl);
			}
			if(ledVal.hasOwnProperty("B")){

			    var LEDvalue = ledVal.B;
			    var LED_Blue_H = (LEDvalue & 0x0fff) >> 8;
			    var LED_Blue_L = (LEDvalue & 0xff);

			    var ledCtrl = [ 0x53, 0x80, 0x03, 0x32, 0x00, 0x00, 0x50,    0x53, 0x80, 0x03, 0x34, LED_Blue_L,  LED_Blue_H,  0x50];
			    data.push.apply(data,ledCtrl);
			}
		}



        if(data.length >= 14) {
          return {
            "dh://lcd/txt1": ["Write to LED " + data.length + " bytes"], 
            "dh://usp/write": [{ "data": data }],
          }
    	}


  	}

}

```



