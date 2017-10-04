#include "HX711.h"

//HX711 scale(D5, D6, 128);
byte PD_SCK = D5;
byte DOUT = D6;
byte GAIN;
long OFFSET;
float SCALE;

float calibration_factor = 2230; // this calibration factor is adjusted according to my load cell
float units;
float ounces;

bool is_ready() {
  return digitalRead(DOUT) == LOW;
}

void set_gain(byte gain) {
  switch (gain) {
    case 128:   // channel A, gain factor 128
      GAIN = 1;
      break;
    case 64:    // channel A, gain factor 64
      GAIN = 3;
      break;
    case 32:    // channel B, gain factor 32
      GAIN = 2;
      break;
  }

  digitalWrite(PD_SCK, LOW);
  read();
}

long read() {
  // wait for the chip to become ready
  while (!is_ready());

    unsigned long value = 0;
    byte data[3] = { 0 };
    byte filler = 0x00;

  // pulse the clock pin 24 times to read the data
    data[2] = shiftIn(DOUT, PD_SCK, MSBFIRST);
    data[1] = shiftIn(DOUT, PD_SCK, MSBFIRST);
    data[0] = shiftIn(DOUT, PD_SCK, MSBFIRST);

  // set the channel and the gain factor for the next reading using the clock pin
  for (unsigned int i = 0; i < GAIN; i++) {
    digitalWrite(PD_SCK, HIGH);
    digitalWrite(PD_SCK, LOW);
  }

    // Datasheet indicates the value is returned as a two's complement value
    // Flip all the bits
    data[2] = ~data[2];
    data[1] = ~data[1];
    data[0] = ~data[0];

    // Replicate the most significant bit to pad out a 32-bit signed integer
    if ( data[2] & 0x80 ) {
        filler = 0xFF;
    } else if ((0x7F == data[2]) && (0xFF == data[1]) && (0xFF == data[0])) {
        filler = 0xFF;
    } else {
        filler = 0x00;
    }

    // Construct a 32-bit signed integer
    value = ( static_cast<unsigned long>(filler) << 24
            | static_cast<unsigned long>(data[2]) << 16
            | static_cast<unsigned long>(data[1]) << 8
            | static_cast<unsigned long>(data[0]) );

    // ... and add 1
    return static_cast<long>(++value);
}

long read_average(byte times = 1) {
  long sum = 0;
  for (byte i = 0; i < times; i++) {
    sum += read();
  }
  return sum / times;
}

void set_scale(float scale = 1.f) {
  SCALE = scale;
}

void set_offset(long offset = 0) {
  OFFSET = offset;
}

void tare(byte times = 10) {
  double sum = read_average(times);
  set_offset(sum);
}

double get_value(byte times = 1) {
  return read_average(times) - OFFSET;
}

float get_units(byte times = 1) {
  return get_value(times) / SCALE;
}




void setup() {
  Serial.begin(9600);
  Serial.println("HX711 readings");

//  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(PD_SCK, OUTPUT);
  pinMode(DOUT, INPUT);
  
  set_gain(128);
  set_scale();
  tare();  //Reset the scale to 0

  long zero_factor = read_average(); //Get a baseline reading
  Serial.print("Zero factor: "); //This can be used to remove the need to tare the scale. Useful in permanent scale projects.
  Serial.println(zero_factor);
}

void loop() {
//  Serial.println(digitalRead(D5));
  // put your main code here, to run repeatedly:
//  digitalWrite(LED_BUILTIN, LOW);   // Turn the LED on (Note that LOW is the voltage level
                                    // but actually the LED is on; this is because 
                                    // it is acive low on the ESP-01)
  set_scale(calibration_factor);
  Serial.print("Reading: ");
  units = get_units(), 10;
  if (units < 0)
  {
    units = 0.00;
  }
  ounces = units * 0.035274;
  
  Serial.print(units);
  Serial.print(" grams"); 
  Serial.print(" calibration_factor: ");
  Serial.print(calibration_factor);
  Serial.println();
  
//  delay(1000);                      // Wait for a second
//  digitalWrite(LED_BUILTIN, HIGH);  // Turn the LED off by making the voltage HIGH
//  delay(2000);                      // Wait for two seconds (to demonstrate the active low LED)
}
