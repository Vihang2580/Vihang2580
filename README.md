#include "Freenove_WS2812B_RGBLED_Controller.h"

#define I2C_ADDRESS  0x20
#define LEDS_COUNT   10  //it defines number of lEDs. 

Freenove_WS2812B_Controller strip(I2C_ADDRESS, LEDS_COUNT, TYPE_GRB); //initialization

#include "Freenove_4WD_Car_for_Arduino.h"

#define TK_STOP_SPEED          0
#define TK_FORWARD_SPEED        (90 + tk_VoltageCompensationToSpeed    )
#define PIN_SONIC_TRIG      7
#define PIN_SONIC_ECHO      8
//define different speed levels
#define TK_TURN_SPEED_LV4       (180 + tk_VoltageCompensationToSpeed   )
#define TK_TURN_SPEED_LV3       (150 + tk_VoltageCompensationToSpeed   )
#define TK_TURN_SPEED_LV2       (-140 + tk_VoltageCompensationToSpeed  )
#define TK_TURN_SPEED_LV1       (-160 + tk_VoltageCompensationToSpeed  )
#define OBSTACLE_DISTANCE   40
#define OBSTACLE_DISTANCE_LOW 15
#define MAX_DISTANCE    300   //cm
#define SONIC_TIMEOUT   (MAX_DISTANCE*60)
#define SOUND_VELOCITY    340   //soundVelocity: 340m/s
#include <SPI.h>
#include "RF24.h"

#define PIN_SPI_CE      9
#define PIN_SPI_CSN     10
#define PIN_BUZZER      A0
RF24 radio(PIN_SPI_CE, PIN_SPI_CSN); // define an object to control NRF24L01
const byte addresses[6] = "Free1";   //set commucation address, same to remote controller
int nrfDataRead[8];            //define an array to save data from remote controller
int tk_VoltageCompensationToSpeed;  //define Voltage Speed Compensation
  
int stopped = 0;
int distance = 0;

void setup() {
// while (!strip.begin());  
//  pinsSetup(); //set up pins
pinMode(PIN_BUZZER, OUTPUT);
  pinMode(PIN_SONIC_TRIG, OUTPUT);// set trigPin to output mode
  pinMode(PIN_SONIC_ECHO, INPUT); // set echoPin to input mode
  getTrackingSensorVal();//Calculate Voltage speed Compensation
  Serial.begin(9600);
 // strip.setAllLedsColor(0xFF0000); //Set all LED color to red
  delay(20);
  // NRF24L01
  if (radio.begin()) {                  // initialize RF24
    radio.setPALevel(RF24_PA_MAX);      // set power amplifier (PA) level
    radio.setDataRate(RF24_1MBPS);      // set data rate through the air
    radio.setRetries(0, 15);            // set the number and delay of retries
    radio.openWritingPipe(addresses);   // open a pipe for writing
    radio.openReadingPipe(1, addresses);// open a pipe for reading
    radio.startListening();             // start monitoringtart listening on the pipes opened
    Serial.println("Start listening remote data ... ");
  }
  else {
    Serial.println("Not found the nrf chip!");
  }
  delayMicroseconds(1000);
  
 /* if(nrfDataRead[2]== 1){
   strip.setAllLedsColor(0,255,50);
   delay(20);
  }
  else{
    strip.setAllLedsColor(0, 0, 0);    //set all LED off .
    delay(20);
  }*/

      delay(200);
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);
      delay(200);
      while(stopped != 4){
        distance = getSonar();
        Serial.println(distance);
        if(distance <= 18){
           motorRun(TK_STOP_SPEED, TK_STOP_SPEED);
        }
        else{
        run();
        }
      }
      
    
     
    motorRun(TK_STOP_SPEED, TK_STOP_SPEED); //car stop
      delay(20);       
      strip.setAllLedsColor(0xFF0000); //Set all LED color to red
      delay(5000);
      stopped = 0;
    
  
}

void loop(){
   delay(200);
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);
      delay(200);
      while(stopped != 4){
        distance = getSonar();
        Serial.println(distance);
        if(distance <= 18){
           motorRun(TK_STOP_SPEED, TK_STOP_SPEED);
        }
        else{
        run1();
        }
      }
      
    
     
    motorRun(TK_STOP_SPEED, TK_STOP_SPEED); //car stop
      delay(20);       
      strip.setAllLedsColor(0xFF0000); //Set all LED color to red
      delay(5000);
      stopped = 0;
  
}



void run1() {
  strip.setAllLedsColor(0x00FF00);  // LED green
  u8 trackingSensorVal = 0;
  
  trackingSensorVal = getTrackingSensorVal(); //get sensor value
  switch (trackingSensorVal)
  {
    case 0:   //000
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED); //car move forward
      break;
    case 7:   //111
      motorRun(TK_STOP_SPEED, TK_STOP_SPEED); //car stop
      if (stopped <= 3){
        delay(20);
        strip.setAllLedsColor(255, 255, 0); //set all LED color to yellow. this is just deffent form of rgb value.        
        for (int i = 0; i < 4; i++) {
    digitalWrite(PIN_BUZZER, HIGH); //turn on buzzer
    delay(300);
    digitalWrite(PIN_BUZZER, LOW);  //turn off buzzer
    delay(300);
  }
        motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);
        strip.setAllLedsColor(0x00FF00);  // LED green         
        delay(200);
              }
      stopped = stopped +1;
      break;
    case 1:   //001
      motorRun(TK_TURN_SPEED_LV4, TK_TURN_SPEED_LV1); //car turn
      break;
    case 3:   //011
      motorRun(TK_TURN_SPEED_LV3, TK_TURN_SPEED_LV2); //car turn right
      break;
    case 2:   //010
    case 5:   //101
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);  //car move forward
      break;
    case 6:   //110
      motorRun(TK_TURN_SPEED_LV2, TK_TURN_SPEED_LV3); //car turn left
      break;
    case 4:   //100
      motorRun(TK_TURN_SPEED_LV1, TK_TURN_SPEED_LV4); //car turn right
      break;
    default:
      break;
  }
  return stopped;
}
void run() {
  strip.setAllLedsColor(0x00FF00);  // LED green
  u8 trackingSensorVal = 0;
  
  trackingSensorVal = getTrackingSensorVal(); //get sensor value
  switch (trackingSensorVal)
  {
    case 0:   //000
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED); //car move forward
      break;
    case 7:   //111
      motorRun(TK_STOP_SPEED, TK_STOP_SPEED); //car stop
      if (stopped <= 2){
        delay(20);
        strip.setAllLedsColor(255, 255, 0); //set all LED color to yellow. this is just deffent form of rgb value.        
        for (int i = 0; i < 4; i++) {
    digitalWrite(PIN_BUZZER, HIGH); //turn on buzzer
    delay(300);
    digitalWrite(PIN_BUZZER, LOW);  //turn off buzzer
    delay(300);
  }
        motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);
        strip.setAllLedsColor(0x00FF00);  // LED green         
        delay(200);
              }
      stopped = stopped +1;
      break;
    case 1:   //001
      motorRun(TK_TURN_SPEED_LV4, TK_TURN_SPEED_LV1); //car turn
      break;
    case 3:   //011
      motorRun(TK_TURN_SPEED_LV3, TK_TURN_SPEED_LV2); //car turn right
      break;
    case 2:   //010
    case 5:   //101
      motorRun(TK_FORWARD_SPEED, TK_FORWARD_SPEED);  //car move forward
      break;
    case 6:   //110
      motorRun(TK_TURN_SPEED_LV2, TK_TURN_SPEED_LV3); //car turn left
      break;
    case 4:   //100
      motorRun(TK_TURN_SPEED_LV1, TK_TURN_SPEED_LV4); //car turn right
      break;
    default:
      break;
  }
  return stopped;
}

void tk_CalculateVoltageCompensation() {
  getBatteryVoltage();
  float voltageOffset = 7 - batteryVoltage;
  tk_VoltageCompensationToSpeed = 30 * voltageOffset;
}

//when black line on one side is detected, the value of the side will be 0, or the value is 1  
u8 getTrackingSensorVal() {
  u8 trackingSensorVal = 0;
  trackingSensorVal = (digitalRead(PIN_TRACKING_LEFT) == 1 ? 1 : 0) << 2 | (digitalRead(PIN_TRACKING_CENTER) == 1 ? 1 : 0) << 1 | (digitalRead(PIN_TRACKING_RIGHT) == 1 ? 1 : 0) << 0;
  return trackingSensorVal;
}

float getSonar() {
  unsigned long pingTime;
  float distance;
  digitalWrite(PIN_SONIC_TRIG, HIGH); // make trigPin output high level lasting for 10μs to triger HC_SR04,
  delayMicroseconds(10);
  digitalWrite(PIN_SONIC_TRIG, LOW);
  pingTime = pulseIn(PIN_SONIC_ECHO, HIGH, SONIC_TIMEOUT); // Wait HC-SR04 returning to the high level and measure out this waitting time
  if (pingTime != 0)
    distance = (float)pingTime * SOUND_VELOCITY / 2 / 10000; // calculate the distance according to the time
  else
    distance = MAX_DISTANCE;
  return distance; // return the distance value
}
