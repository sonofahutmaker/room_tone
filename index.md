# Room Tone
## by Jenni Hutson

Room Tone is a piece about how I've found that the environment I'm in profoundly affects my mental health. I've been thinking about how this last year in quarantine, where I've been at home more than ever before, has shaped my experience with anxiety. Room Tone is a digital musical instrument that explores this experience.

## Components

The physical components of the instrument are 3 soil-moisture sensors which are in three plants, and a gyroscope which I sewed onto a gardening glove. These sensors are hooked up to an Arduino Uno, which sends their data back into my computer. This data is processed by a Max MSP patcher. See full Arduino and Max MSP code below.
![max_Shot](https://user-images.githubusercontent.com/39632502/120905803-3aa0d400-c61a-11eb-9233-7bdedeb83840.png)


### Arduino Code
<details>
  <summary>Expand to see full Arduino code</summary>
  
  ```
  
  #include "Adafruit_seesaw.h"
  #include<Wire.h>
  #include "I2Cdev.h"
  #include "MPU6050_6Axis_MotionApps20.h"

  MPU6050 mpu;
  #define OUTPUT_READABLE_YAWPITCHROLL

  #define LED_PIN 13 // (Arduino is 13, Teensy is 11, Teensy++ is 6)
  bool blinkState = false;

  // MPU control/status vars
  bool dmpReady = false;  // set true if DMP init was successful
  uint8_t mpuIntStatus;   // holds actual interrupt status byte from MPU
  uint8_t devStatus;      // return status after each device operation (0 = success, !0 = error)
  uint16_t packetSize;    // expected DMP packet size (default is 42 bytes)
  uint16_t fifoCount;     // count of all bytes currently in FIFO
  uint8_t fifoBuffer[64]; // FIFO storage buffer

  // orientation/motion vars
  Quaternion q;           // [w, x, y, z]         quaternion container
  VectorInt16 aa;         // [x, y, z]            accel sensor measurements
  VectorInt16 aaReal;     // [x, y, z]            gravity-free accel sensor measurements
  VectorInt16 aaWorld;    // [x, y, z]            world-frame accel sensor measurements
  VectorFloat gravity;    // [x, y, z]            gravity vector
  float euler[3];         // [psi, theta, phi]    Euler angle container
  float ypr[3];           // [yaw, pitch, roll]   yaw/pitch/roll container and gravity vector

  // packet structure for InvenSense teapot demo
  uint8_t teapotPacket[14] = { '$', 0x02, 0,0, 0,0, 0,0, 0,0, 0x00, 0x00, '\r', '\n' };

  volatile bool mpuInterrupt = false;     // indicates whether MPU interrupt pin has gone high
  void dmpDataReady() {
      mpuInterrupt = true;
  }

  Adafruit_seesaw ss;
  Adafruit_seesaw ss1;
  Adafruit_seesaw ss2;

  const int MPU=0x68; 
  int16_t Tmp,GyX,GyY,GyZ;
  int yaw, pitch, roll;
  //float yaw, pitch, roll;
  int t00, t01, t10, t11, t20, t21;
  int b00, b01, b10, b11, b20, b21;
  int x0, x1, y0, y1, z0, z1;

  void setup() {
    // put your setup code here, to run once:
    #if I2CDEV_IMPLEMENTATION == I2CDEV_ARDUINO_WIRE
          Wire.begin();
          TWBR = 24; // 400kHz I2C clock (200kHz if CPU is 8MHz)
    #elif I2CDEV_IMPLEMENTATION == I2CDEV_BUILTIN_FASTWIRE
        Fastwire::setup(400, true);
    #endif

    Serial.begin(115200);

    mpu.initialize();
    mpu.testConnection();
    devStatus = mpu.dmpInitialize();

    Wire.begin();
    Wire.beginTransmission(MPU);
    Wire.write(0x6B); 
    Wire.write(0);    
    Wire.endTransmission(true);

    mpu.setXGyroOffset(-2046);
    mpu.setYGyroOffset(-42);
    mpu.setZGyroOffset(-10);
    mpu.setZAccelOffset(1688);

     if (devStatus == 0) {
      mpu.setDMPEnabled(true);
      attachInterrupt(0, dmpDataReady, RISING);
      mpuIntStatus = mpu.getIntStatus();
      dmpReady = true;
      packetSize = mpu.dmpGetFIFOPacketSize();
     }

     if (!ss.begin(0x36)) {
      Serial.println("ERROR! seesaw 36 not found");
      while(1);
    } else {
      Serial.print("seesaw started! version: ");
      Serial.println(ss.getVersion(), HEX);
    }

    if (!ss1.begin(0x37)) {
      Serial.println("ERROR! seesaw 37 not found");
      while(1);
    } else {
      Serial.print("seesaw started! version: ");
      Serial.println(ss.getVersion(), HEX);
    }

    if (!ss2.begin(0x39)) {
      Serial.println("ERROR! seesaw 37 not found");
      while(1);
    } else {
      Serial.print("seesaw started! version: ");
      Serial.println(ss.getVersion(), HEX);
    }

    pinMode(LED_PIN, OUTPUT);
  }

  void loop() {
    // put your main code here, to run repeatedly:
    if (!dmpReady) return;

    uint16_t tempC = ss.getTemp();
    uint16_t capread = ss.touchRead(0);

    uint16_t tempC1 = ss.getTemp();
    uint16_t capread1 = ss1.touchRead(0);

    uint16_t tempC2 = ss.getTemp();
    uint16_t capread2 = ss2.touchRead(0);

    Wire.beginTransmission(MPU);
    Wire.write(0x3B);  
    Wire.endTransmission(false);
    Wire.requestFrom(MPU,12,true); 
    GyX=Wire.read()<<8|Wire.read();  
    GyY=Wire.read()<<8|Wire.read();  
    GyZ=Wire.read()<<8|Wire.read();   

    t00 = tempC >> 3;
    t01 = tempC & 7;

    t10 = tempC1 >> 3;
    t11 = tempC1 & 7;

    t20 = tempC2 >> 3;
    t21 = tempC2 & 7;

    Serial.write(t00); //0th temp sensor
    Serial.write(t01); 
    Serial.write(t10); //1st temp sensor
    Serial.write(t11);
    Serial.write(t20); //2nd temp sensor
    Serial.write(t21);

    b00 = capread >> 3;
    b01 = capread & 7;

    b10 = capread1 >> 3;
    b11 = capread1 & 7;

    b20 = capread2 >> 3;
    b21 = capread2 & 7;

    Serial.write(b00); //0th moisture sensor
    Serial.write(b01); 
    Serial.write(b10); //1st moisture sensor
    Serial.write(b11);
    Serial.write(b20); //2nd moisture sensor
    Serial.write(b21);

    mpuInterrupt = false;
    mpuIntStatus = mpu.getIntStatus();

    // get current FIFO count
    fifoCount = mpu.getFIFOCount();

    if ((mpuIntStatus & 0x10) || fifoCount == 1024) {
          // reset so we can continue cleanly
          mpu.resetFIFO();
    } else if (mpuIntStatus & 0x02) {
          // wait for correct available data length, should be a VERY short wait
          while (fifoCount < packetSize) fifoCount = mpu.getFIFOCount();

          // read a packet from FIFO
          mpu.getFIFOBytes(fifoBuffer, packetSize);

          // track FIFO count here in case there is > 1 packet available
          // (this lets us immediately read more without waiting for an interrupt)
          fifoCount -= packetSize;

           #ifdef OUTPUT_READABLE_YAWPITCHROLL
              // display Euler angles in degrees
              mpu.dmpGetQuaternion(&q, fifoBuffer);
              mpu.dmpGetGravity(&gravity, &q);
              mpu.dmpGetYawPitchRoll(ypr, &q, &gravity);

              yaw = ypr[0] * 180/M_PI;
              pitch = ypr[1] * 180/M_PI;
              roll = ypr[2] * 180/M_PI;

          #endif

          Serial.write(yaw);
          Serial.write(pitch);
          Serial.write(roll);

          blinkState = !blinkState;
          digitalWrite(LED_PIN, blinkState);
    }

    Serial.write(255);  
    delay(100);
  } 
  ```
</details>


### Max MSP Code
<details>
  <summary>Expand to see full compressed Max MSP code</summary>
  
  ```
<pre><code>
----------begin_max5_patcher----------
52903.3oM681zbjbjjkfmq9WQHTZoOrKoSS+x9nkYOrhTqHybnON6dXpQJAL
ynHQWHAxF.Y8QO8z+1WU8.fLQFpmvCDtag5UwjUkIY.jvc0dl9LUMSM88+5e
327M+vc+k8O7M69m28+X2u427+5e327aF+H6C9MO8e+a9lOb0e4c2b0Cieae
ys6+y28C+qey2d3K839+xiie7Ce7lqebGzR6PJ87W81O8gqu8l8ON92jd5Cu
98i+EzeHeGvU3y9du6SO972L9zmd3id7u9w8GdG+lqu8wu4aO7G69e9z2zGu
5w28SWe6O96ue+6d7v2G.oxPq.D.Rtg.UykuU+TXfKBmxYNWvp.4ucWoLj91
cHNj18+z948+9e3ev9sucliGeX+COb0Ot+nAD+AAzaPnj8GDfoFD95lNkvgB
kpLjobVJEPT6ingbJKRJ2HAfVl91cR5ha5oE0zQnNTEjQFXtvbsPl8kMy7xa
r45xhyOAoPC0eIMAA09xz.niqrB7IopXe8rM8Ib4+O9ONAS+Tmh+pN3DqN3R
oxYHqSiHJ2TGbIOzR579l9wBTHhU6tMHWXfWVXG71.IBpyyqfvMVXiaS+PrT
04CHUzY9k1pY5vIX5zx5fmqCrRlmvFxDqT6qjo+UWlCQcYNQl+xb40ZYN0se
HWUJublRrNyGTNfFMP4Joqx0xJQXiqq1pbsSXlvhR0iXAGZrZ+UgLyVx5Z4U
RV.p9Iv9at918yFxoZ8DgbCu+gqt8G+plsTYY.oLoyywBKPsfpCPVTashXqI
MrkSsjB475rfGjlOyGUaKInKUcobc5dR8wUu8Jzzk2ghHl6uR12zkAHfx7ZM
E3d88c+NXGti1w6jc4ckc0cscPRIgTuQkadGv5xPJj3NHAEuQo1TtFku1jky
5++UGl0nGFZUhkTJynszpMNS4z.KkrFTQhrUba5TOcMn0YV17mi0fEcNVoAC
5LKjsLEXaDvL9rsFCk0PJ.n1T+MRN60X9zG1e6md9c+i2u+g8293UOd8c29h
E6AVG0yoh4VKhNpqKtoQ7LjfRiU+co0porR6qNl+BT77.0i6+vSo28M+g6t6
wGdb+GeX3Oe0exlD7s1u8S6u59G+g8W83K9zOb06d4+80u696zOX+u+G1u+i
vDesGdT+Y8hu1s6e7Oby0+kW7YeTw6e+ie59a0A9W7E929z06e76zwgOd88i
iCu3qp.3W9W3Oe+0O97m8yl7K82bmvf+7f9U2e0G1+396+86u8pe3lwIHI2I
SjCL8pAvNKmsrxSWSIMb8FAI8eqXgzj04VJfhbCp0lnS4ZeABufNa28w82t6
e7Tb4nk0kC0LYDLwXKoQ01nhkrdVWLW81zz3z37kFP1Rcx4Q27Ce5wGUfa9S
U3SepxjCBub08e3Ge2c2b28G9Zogr5EqyL03XpYbbJfYn1TiO6SEX7y+pClJ
ef82DvZZ.zrDFyBDqXpnwDzz3iHMYwDYzaDlEuO6BkmD0Vz7jjLgCp+iljjx
gOZglODUssGQL+MMWAfDTmrQEMU5JjJZXjUMqwVdMVSCq6NonmZ4EN5IMjYA
rU0QMZoDOtpg6Jamoa19+82e069OOACs3anouFoQUYHqZdfZTeUE53hYMZrg
CZpFPUznTZbQiXTiDdLq.62eSVyOd0029e9U4L9vmt4wqe2Oc0s2t+l+zU2e
8U293m+5+Rasd5TJSlzvCW+i2d0MyHhN+fZzOsnoPzrX4XA0gwCft90rrG1B
gN2hUny0DNnyHUlDMgDTxEwnh03oYawb8ixMalZa8hb9eDOkkx4TZY4XbseD
FrHY.MPF.EnUU6mZqQFZe7p28G2k1cBaNWBV+A.MQ0AkfkYTWzQ+3jQwxW3D
m3DtrQwUZC4BUxBqS3qoJRSX5mYdSmXPbbhVsf37WXBzkYIPCkB3wEUGO5IL
Ob3zW3TUWDnIGVXJmreoeq0rnCYde1BLHMU1kLVUzIYgdgUE4ZiGWfFgjF4a
p0Hc4zFy0wUVGJLTpFOFkykp38YyOKLNcFgVO6rvlwFpog+LFKnlngRTwYZ7
bDXMLhR1vIctql1swXM2gf2.Z83c+3Ody92FZYSXTeMMV+FoKr2.dMPKoCn0
qsc2JXoKjpFeQyRlRjv1lgXSkFJ4wS7wNY.c5Krpf0DK87iW83aDAyZ5uJiY
SSxupAInKQZgGLnS9Rrnwly0ltlosUmeVtet76+g6t8wGt9eeb.sLLNLAkbI
qIOmYWb8URwX1X2axKCkgBp1lFDOfIMUW5qY3qGKHPCM0yoTPr1TbYbGj6CK
XIFrfIYfx5en1IvrTCLInKX0GRvZHHA8hvbKwA5BfcfCTI6ZtvZqObft9XQg
BLkFpVH70w5IPyQU5EEHjBAEXootURMSohNOQmGZGkdT4.cQqtvABPD3.827
usDInKB1i.AaIWRP.6BI3DdYAgETmKoKsZuTBiE80DvYwBVP14iNEuJJFbfk
lMWgYpHBB4JUmGG3TC.qFCnOR85LfmORwgf+qjGxJCAqNKkDVxiNGyf+aAAp
yf8yG8lA6WVnyh8S.1m8S5C6mu+0LX+9LCe8X+JhMufQVZ1l0hIpSre4Xv9k
KCp0qoToP.SRcLTgPx94hT8f8qDB1uLOjxYQD6364bgkME6mK50iDfgrKnV6
C4mu6UTB8KamPdRWLRyqPHLYkERWH+Zwf7SnATWOVSnpwslFddXC8yEo5.4G
lBA4mjF.y6oVP0SxJMrME4mK50gP+3Z0MzOD5C6mu+UTB8iaZZ4nni7Zl4RJ
Cc6bfQLF7ebZPGoYqJHIqVEFuTUwb6+7Qqtr8eHEBNPpLnISURPQMeTG.3s0
1+4ifcIDPMkSObk6CKnuWVThAjxCsrtxpNuJQIKx39DCHJwfCDqCTtYSJqXI
mj4xA1+X.cQpdDCXND7epCCpyI03n.VIQ34x+EjX.cQudv9gEe1uReX+78uh
B6m9LYvpPQ8sS+ufV2NBXrFC9OPzbSL.QZ1dyvXbOBXezpOw.1BAGHLdOdTq
sgYkJDqari.1GA6Rs.pC.N3Jk5CKnuWVTXAAXnTopN3iPw5pDkdwBRPLXASz
.T0odblXyKZ71eGTVPWzpKrfDFBVvTZ.RMk9qkEizXbyh1RrftHXWpFPxshn
IpOrf9dYAgEDaUkilraKZEpVmFH2mLgoXbqPxsz.BrNwTMcMT8rsK0gLSXej
pCYBSg3FgjqkAql1TFNLmpV2eZKkIrO50iX.0bucQ09bePlv+Z0Y+Ti5G1e+
7u5fTYAur1yqCOUzEDE6hA0JIq09Ysh.+duy5ziid2ce5V0L2AScQr8ycXhZ
HfmS+Q4amSfN98jHzpodRC.xZaHVqxP+zhrFiKO7G93MW8WmeSMfo127VuL+
yoZ0cuP+Y0ImJ5mkKi7g1Jzb4n1BzCW8m1+9eu9Zp+.+8W83i2esFVvg9.6u
4mGUzwoqd35283mrF0yHVxOMg23Ut4l69y+3M28CWcyi6+vGu6ybGru58e3p
ae7c2cu8F+DSwO+U+vcue+gPQre7eyye7c2e8OdsZ+2r+1e7we5ot7lUlzoO
6WVwKqqLd869iO7yiO+xe0e9MQIOdwesDSO+89wq0AyIdy929zU2b8i+UmWt
Gu9C6e3w62q+cepgsbXp0g4VKXqUaGcBMHDEPV16tdgGTZmLCfUW5p2UckZP
HyKJTIoScqkRxZiGkpUp+eVTnEcltROx1UnbgisgiQl3UM5.tVE0DLlslfu3
7Xrd3BzRj04CuvQg5iTOGEpUwCkltJ9xmu.GhrvqMYfATSikzEnE07ew4wPV
UNasvsVLiB0G8dNJzZRXMNTHQKe4XmXeTsO4fOg+0yQgpXmFcp5icQJIGMA6
ATDKvlp5hm0grNQ9EjFyPUYODwZjSsLniBsZPI+7QpdP9EilxfNmTrBQTDrz
HEwpaIxOezqCjeZ.VtfZm5HC9tWAg6qBCLhDiVmGw52GPm39BR6XnfCF2QgI
Qw.qmEFUtOWjpGbewnWLjapIn+ClqbEsVx+lh6yE85A2WQ7SnsSshAe+qfP9
kyVmsOWMuG1RqnMKxuE3LMkXzJFzHHFraHDj0+o1Zsx7X+tDchAWv504+VBv
JDchgpHCVRiluqk7aklGAXXZDCt.3Ln.O+Sfgb0JDoOMhAeerWmArKMiFgFp
jFStnqK0Z5ysaUhnDiFwfuLeDzlQiKZ0kZvQBQyXnx3.jSYxZS8n0aswMVyn
wEA6SM33FFnzm1wvDdYQokbwvfIqMPEyjUdCsN0LZjXzNFpDNXAVHowyyL0f
ndij8QpNTCNRHZGCUrMnLb5bwZxjTgRYScij8QudzLZzQEWTsO8igI7uV8aj
7hHuZPYUjWMxtk1GKuZUQWernygApXUquUipqj7pkOA8AAVTAEUAW1Qc0pMS
Z4xikZRIoCHzFSc0ffIQDnvdUxjcLDJ4it5KWU2NaCxhf5pgKaIlfslWcKot
cJ+ihUY02yT0QtOpql85br5poDi+p5p8VUWMDhj5pAb0Sc0x3P0zFKS0HUOP
Bij5pgKqtbngf5otZ4l0CsJIRRPKWGSZsypqFRaN0UyzUdSdjHqIVCIHqQOJ
YkuhBu5p8Zs3gSbZkoPTGqtZ7nBMjTddAzfHS0ZtWpqVJcZpqFJKbzSIO0Uy
aksNqtZX9zUWMrU7TWMppAoHVY2oKVhZ7wRvTWMrbATWM+fZrlz.Wnh5HLJG
2VxtaKwUCqwJxYcEbGwEySmKig3pQ3xJ+091unddIVMa8eMiMkdMLhqFQoNL
.f4gFVG86PSzGSs.ntZDsr5JGVKNpqlqs2W4UiNmKB8aXagwZySd0xoAJoQh
Al7PUx55sQPd0.xStmJq+AYRTLNHScAYOgepnwQngPK5hikp98v0fnrPGiVZ
Xcq9AYRTHNHywNRxwxqlFzm4VojOZTsIBpgVZgbDHu5kRd0HpOGj4DdYRafq
r4sQVk9RsKTKUoU8D7otvBFiixz1sfik9oPRB5CV8gDLDmloaLlaHNPe.rCb
fSHuZD0miyz2GKJTf1IHdrbO0EJvXze4oTxS3mhIGnKZ0ENPND8Xd+c+aKQB
5hf8HPPe4Ui39zk4mvKKJrfExS1mJqcEsQbL5w71d33H+SuNG3EPfg7PpWmA
77QpPze4wJ6IuZuN+WTDXHOzaFreqj7pQbe5t7S3eMC1utnwF4jmrO0A1uXz
c40Yhd5+TDY+bQpdv9EhtKOVPO4Uayv94hd8HAXW4Ui39zb4mv8JJg9wYOYe
pCjewn0xiYvS9mhH4mKR0CxuPzV40rm7jWsMC4mK50gP+lPd0HoOMU9I7uhR
neD6I2S8X6+jXzJ6PaJ3wB+TL29OWzpKa+mDh1YmF4jm7psg19OWDrKg.5Ju
ZjzmFZ2DdYQIFPD7j8o0OFPIFczN881S9mhXLftHUGhATBQGsCIxSd01Lw.5
hd8f8yWd0HoOsztI7uhB6Wp4I2ScIFvXzU6Pj7D9oXFCnKZ0mX.CQmsCwjm7
psghAzEA6Rs.5JuZjzmda2DdYAgEDaYO4dpGrf4Xzb6P.7D9oPxB5iVcgELG
htaGlpdxq11gEzGA6R0.5JuZTtOs2tI7xhBKXk7j8o0OS3bLtUH5Xrm7OEvL
g8QpNjIbND2HDkjySd01JYB6id8HFPe4Uix849fLg+0py9chxqltjX2kWMMO
.G4Uyq46DH4UScdVY4UahlRjNiAfwNBkcG1sqWcTjWM80YEkWsItQ+IXf0Oz
5W5RVSqj21xqlX2c9kPd0.dKHuZ4SSc0Tefk8xqWQO0UyqAgvRGTWMclrmpO
UVaQ1fJwHQbNk8T+omCB0TCOcznfQPfg7PpmCBc8DYCpDhjv0HL8TWsmCB0p
QaMfKPpQUfg7PumCBs+pql5W2kfPmv+54fPy5xqDlnxkQjMJMOUepCjeAouL
zHO4eJhjetHUOH+hQOYnk7TWsMC4mK50AxOe0UiJcpgL36dEDtub1S0m5.2W
P5FCZVONp+TD49bQpdv8EiVwPo3otZaFtOWzqGbe9pqFU5TmXv2+JHjeB3o2
Sk0Wc0nRP5DC5zxiU9oWm86hHsPdf0qy+s.fUMDMhAJSNpq1qS.FGkExC.mA
E3JotZ5fUen.c8wdcFvtnqP5zGG0dpGkfSMF8gAeU9HnMkPWzpKkfSMD8hAR
Rdpq1FpoD5hf8oDbbCCr1mtwvDdYAoDbzEk7T8o0uDbpwnaLPbxS8mBXI33i
TcnDbpgnaLPipmvQpq1VoDb7QudzKZ7UWMp1m1wvD9Wq9ERdITWMjyqg5pYy
fuvpq1oHPH3q03NNsYDbMC+Mn7pgbrDIBV841NxqFxskcNFlCk7pUYG4Uy31
9U4U6sIuZnjBj7pw4hr0jWMTVTg4PczpwTd0PA2ZxqlcYM1nxqF9Zs3gSbZk
N2IVxqFeZxqFJ7xtxlTho7pgSIV8ou55z7VTd0PI2e4UahfZ17xqFJkXE4rl
D1lRd0PotrTLt1efkWMTZcX.HjxqFlSKqoCbLkWM7btHzm99BqCD4Mi7pYRc
xwx8jt7+peRlXNDmjoFUDuYjWMezBj0+jLwbDNISErfss7p4ifDcojWML2kS
xbJurvnsPjifO0GVPIFrfZ.+aE4UyEr5CIXNDjfdwXtoTVHO.rCbfSHuZXtz
GNPWervHrPIG4dpOTf0XPAhksi7p4hV8gCrEBNP2c+aKQB5hf8HPPe4UCKo9
PB56kEEVPo5H6SuNK34VnTXAhAGHvaE4UyEodcFvyGovPv+AvlVd0bQuYv9s
RxqFVn9v946eEEM1fEGYepGrebLX+R3VQd0bQpdv9EgtKOWZsMs7p4hd8HAX
W4UCK49P946dEkP+HvQ1m5A4WHZs7boV2JxqlKR0CxuZHH+pxlVd0bQuND52
DxqFV5RSkeJ+qnD5Gzbj6otr8e0TL3+Jx1Qd0bQqtr8eUHDbfEbiKuZtHXWB
AzUd0vJ1GVPeurvHvPYGYepCw.VoXvAlosh7p4hTcHFvJGB9ubZSKuZtnWOX
+7kWMrJ8g8y2+JLBKD6H2S8IFvbL3+jzlQd07Qq9DCXIDbfbYaKuZ9HXWpEP
W4UCq09vB56kEFgEBbj6o9vB1hAKn5BsYjWMWzpKrfsTHXAIYiKuZtHXWpFP
W4UCaPeXA88xhBKXt5H6ScHS3VLtUH1UibiHuZtHUGxDtEhaDRAwMs7p4hd8
HFPe4USiIsOre99WQSd0vlza4USyqIu8jWM8sbckWsoZJQwUd0vVY8jWsotQ
++Ml7pov7K0IMVj+FVd0fzoouZXaYu99sTat5q1h1HdlV5SJN59zyQgtdxrA
1hQl3MMPfMh9p4hTOGE5JJyFoPjEtFj8lVe0bQumiB8BnuZo9jC9D9WQQigX
GceZ8I+nTLRAuoqguQzWMWjpGjegHE7l5Hrk0WMWzqCjeSnuZo9jA9DtWwQh
gNV2m5A2WLZGCsTYqnuZtHUO39BQuXPcK1z5qlK50CtuIzWsTeZECS3eEDxO
J6n3SuN42BnWMofzJFZzlQe0bAqWm+aI.qXzIFZoss9p4BfyfBbszWMnSMhA
WerfnuZH4n2S8nFbHHHMhAWc9Hn5qlKZ0iZvgfXzLFJkMt9p4hf8oFbbCCD5
T6Xv2KKJ5qF.N59z5WCNDDj1wPtrUzWMWjZ8qAG8mcH3+x7lVe0bQudzLZlP
e0fN0OF78uV8aj7BnTHrrjm6MJlFQy4rnSAKTVC6c736XbPTpVqSCiBVwEWo
PVBklCRzZnzbXQWarXSJz.azoAlJpX0BPQmcntEkjU5Zsz3JlqjRykm+DBHk
WzIDkLNfnEJGaGK.VG6wpkJNnjCzX+aByGJOhsjRyAoXoWF5TH0ECpIzZhYY
hyi8CKpZ6NGhRIS5+SixuEAklCR0kcRlxpVfhjJ1lYYkMvnGFmGWifor5zY6
eD7qJM21To4f2xNzsZJMGlYPSc.ryrHmXS7broaTdf00zgl0Ml0DaxwQm4.X
QUnDLaEKZMmQPXKfbFvmjZuroGGhFhtl.rFhSu0YN.fslNyYMuBM.YM3YcpT
qPZDx1eIcvaHoQ9mzed1eVyWFUmypjxSQq0..WVtcopQQajopkow6yIYJt89
p0ZvTasR5jWopZcxIzxcvR8qXWl+Xo0ZvaYiLNWsVSGrJCjPZl+pGkUVLi2H
kBXpTrojm5LBpRIKv4MkXqAfDqfGKjElnlLhUuxpWV1pKYqUtNTrEzJp6WUC
orRQQs0.XgyRwe.vTFdfxsjhXftvtkrdPjaM.J8XD.SChcAmPSksxnU8+Wd8
VCfEN8APFPaFtFKCkz46i8APWauu5sF.sdp2ZSjDIljAKdNkvkQMS8F920xs
lNdDgM0WCLxtu5hnA9oIZhxXt8fTzPIzQfbSiwfpBMtAi+8qbqAXDZydJXoT
FXqpQ2owRQRqdPt0vA6ZxOV5nMIqCP+pbqMO4VSom6wV6OkW13E4hLwBP+H8
qzJ+clbqAHECVPMj+VxtmMISAXzXT43xBdwzaMMT1PvB5Fj4VhELXBtl9n5C
KnuWVTXAuXJtllaPLXAQc1RyZCrjNvqSDsleQTYAuXJtFfkPvB5tGfaIVvfo
3Z.V6CKnuWVTXAuPJtlNoMFbf1t3PRBQpQYqzon4wA92MJtFPoPv+AJSmX9Q
ETG.pGZ3Jyf+6WUbMW1OB5C6mu+0LX+9aYEWCHLFreowyduRByPkxGJ7tPx9
cgTbMfhvc7Gys1P1tnjTondGEj3ME6WrTbM88oOje9tWQIzuKjhqoqEFBxub
0rgLjaIhnF.OcMWCH42ERw0.JGBxupZokDmKVc6jvjT1TjeASw0TJmtv9Mg+
UTB86ho3Z.UiA+WIO.0hhCoJj0kkqA9nfuXJtFPsPvAVnA63K0okZzKnYsar
s+KXJtFvo9vB56kEkX.uPJtl9fhAGXlFLEPoAIDT3QlKG3e2n3ZZRagf+KmF
pXUwGRvVw51IapX.Clhqoib8g8y2+JHreWNEWSm+FC9OAFHMF8jUilsllvmD
1X.ubJtFvRH3.45.p4AmvQUHvtglaqX.ilhqAbtOrf9dYQgE7ho3Z.WhAKHo
FAzvFgETp5Jz03xBdwTbMfqgfEjrK9HIVmQfIKep5FiELXJtFvs9vB56kEEV
vKjhqARLtXHYzNhtFkQ0UJWJ.D0SC4Ro3ZfDhKERFsCUTotRkrcueakM0ogD
MEWCj9bkPlv+Z0Y+NQEWCDp2JtllWiZbMFrBDOQHol9Tcgl.I4ZfvqqjqMU2
4QWSpXR1WxZ+dZlT18aOFRtlNcd8jbsItX+5JSCPBSbFSBzRUq2YtgkbMkc3
kZmlU8xuEIWCqaAIWKeZJtFHK6U3uBoAcDGAS6QpMDKzT8IDV9aXEWSCzKDQ
gpq.Lnu7.nIEoj9MMZtOOJzhcGvvDQ+cqfqordQHHzJkFZrYDJAEaxH1KNNF
ViMMqDWseUu0lidqARex.eBuqmiA0hGuVvlUkh+cjfqA4XjAtNkaPCfgIwjY
WT3ZMnbeWH8VCxgHAb0IYv15ZhGEjUMv75Vh6KVxsFj6S52S3bEEpuKjdqA4
XzOFpoplZWoH1oiw55SV.2gj56BI2ZZJIgf5KICpqjlnMgZVSZ1RvVh5KXps
Fj6SeXXBuqnv8cwjaMc46XbK7Zzf0+NabMknT5ksggII+96J0VCxwnKLzrV0
qcWVsxWwtGmyh+6WEaMewVCxcpGL35gMCBv+1Vs0fbPZBCtZcQLq+lKmZqAk
XzHFJkAERTqmZJcmFI2FqQzDM0VCJcpUL36kEj5u4Ro1ZPIHshgbcPMacYH6
pIUfZNpMhlKkZqAkPzJFJYYvZlUXg0rRpJhsoZDMQSs0fRm5EC99Wq9sQdpC
9FGRIo9KEf9D3EQ0gTM2J1KioteiZwDlGvTQ+6ahUWiKVGwrTGrMjtp+xD8h
T9KJ.hW8r0S8AIr9+vnrKnYaLndJUlH6YquzeMq3MLzaU4wmt9Yax9uNvx8Y
E1vb4dRkEUw694w.v1PMEII.0PQZ11cXyJOupbZB69gOte+6OAKNuNVbEGZe
1uVSKdrraluAWVGClWACdpB15cWcy9cjLJ9Noc3tzfu.IjcG.pqx.PRWy6H+
7wxUZMzChAoP3uzbylfVcTcBRZfUlHBR5Rd1pfHINzp3QkR1qyjl6x44qAQL
nqboipMAraVzn1vI+Lih7x29KMC5hJlXt1djYSg025iKyJt9F+1gkkV8Aitx
3ZAxhsOqw9+4LtlT5NVc0eNU4DbxJWk99WLinJrs2RSPIKUyPH4znj6xNMKH
MTFieMkyVaZdbywRR6y3jyMM8Bqzyr6x6SRSWHXny7RNyzenHvTzYoCleX4n
y4NX8aFR5Ls9iFqBK8TiF6uY27kJsD1Nw6ww39a9ZJFHnqRLPGD48JR5uMJs
xi2UChW+pqWIcGx1EFNo7tnTy4waNGpoLzF21NDDqk4MaAS8q4bYCQiK9cth
+5KAlPz53bARhRCVtTEtTZMpkMCYEkd1Ilo+O8OcBSyOUF+Wayg8mgmObajN
OQ.bpvul+cmIAsE8tyjTOGiaEKUQm9lZiBfXiVQi8TnvVzzMs8z+fwZZHahq
.lWai8DPVDVVikKOgr+xr33.r3xZqIkW17XG+MkUZbQgv.rKZbHPUPyhBCVt
rRxJXkmZZoLuohj5i2b8i6frFRI3CjjqklOwHpd5xB+5q3z3QAT8krv4wHKJ
kUIVxwQ.D0Qf7oLBHqzHPcr23+ErUq+.vnD2ymx..uVC.OMseMsYSw2+wSHc
xoj4d5q0p.lqAKRdLmgz3FAw0EL0gYXYPOUWYP31.S10ZmasDUysrs2WGX33
m+8kDpu4tqd+g74l6PR5TCr70MatvG1juQCUpqygEgxv7WrJsrKV8rE1FsMI
Bo8mvUIsecR63g52JZNgpcWY6v1psEHu+YUwRYMpGVPqYnZ4lNJE45qwf.pe
kEvGVLsG+a2kGeYx7mk88mt8c+z928G2+9WjHO+hD4EKGd5or2SG9R1GIPIy
1dWqKKIiJ.7T6XsiWU6bDLbu24l0vPXBHS+rZ16G7yuZL.iaI03xHe1K4aqp
o9ge7EO3m9w+zHAO9jew.y3GQoWLl9EuGtKBHCsujXDozyP3y.4hxWL6P2aK
bnsl5IdjOjUKWqF2w+G6j4ar4kdiT3T4y3+WoLUv4afKa4+7r0g4E.+9JqtM
e6aYyyjTWub1lspIkn95iWIlmr10Isrua1l5IGrxqNWkpomr1Oy2TyEYfyfT
DIUgwRf7rs82c2G9vdKF4uv3u912c+9qdX+twtAzt+7Os+1cWe6tOre2Gu6g
qGIue5uh64w9pKEklHykzWidtjOld9oTTd5HeHZYmCL+s9skfEeRvgyw.eJT
0K7NG0RK61eJPdbiiLoXer6RUstQlzj7keajpsENv75gzoWAa8qtgB1IUN2c
Sn1Vq8S5YqOIGa8xpr0By2ss1j0xscQVa5rcaqMdYWGVWDBpUwzggDmT5Wau
wT6qn+6bhREkZdrhxu39vK6t5SZ5cMQWINCVlFDkac0vOADeYIpAStgz.RRM
DSlf6NdU+0khU6lZV.IPgLwC8hi30ktZigAMMdxZQfZVjzXW0dMr7WgCehft
83vqq0FB6N6uQ1dTPIMN0LCby5ZQmIg9jy92go42TGq0k0EPs5Awtqb4TAJk
BwJn2NraamYSbbgL3RaYm5qgSerAmqAxfWTe8p5U+k075xfuSkf06tY+UytG
JWmpkVkd8vtpY5vlBe7sfJcwXpW1hg64M+PiobnxflinjqHX2cz0e43I1gXd
rLiIK8cqMXnugisVX4osHVygsjPx9P5.iY9TtFjfEa4u7F8hMGENdaigi2n1
u3Sln7vp49bA9XNeXt5gr3OdD4K1.32TYu8zOC7r9g7i2e06ud+gc6.94ev+
gqu4lwe3+9mM4u4v6qy2w3uCmok7E+39YaZDte929p+UVrWfqt8GOb.BXI8y
yJew2vmd7tOeXy+65i2e2Gu69etaROHKnyZ8jZIz07Bu0r.26nGFqjfSwhyK
7l0llzhEbMx321ExYGkbt7MqfnCLpJ2GtchJ+ptLjoQgZzworkUHuN6HOOeH
dYSNBAbPyEHWDElKXCsf+q5zYqtYzk3J5BKsTNBoElW1XiQAFveoz6SM6Z+2
XE600Skr0y2kjjZqUZgmVcyTm5RRtHkMiIBoeYXk0TaApgl8+6u+p28e9ZAY
gkiCxhZvwWvqC7r7obAupScAKSuRrOycfR8gfO6Rb84uhugAre7pqu8MMd41
5GvQVZfSuV8E7gOcyiWaSIuc+M+oqt+5qdZMduwyY0Vbt1zMkON9HR+RjB9D
0yN5yeQTPN84yPhyeFL8hQlKlvnTm5NetD5hh6vvyArDUcPQGHue+WdEE8ED
EMkGMc3O6VikoLNKEQAdwesVqlWXEQAVCEQ4tOt+1c+iyeAyxBejAdyl7Vu7
LCM7SeX+se5427qUP6gmd8t5C+fl6w61+c2piHe2i28cOd+md2eb3Oe0exbG
91Qehqu+8O7cWe628vGuWe8+tGd2cOdyU29dmuoGzut8sjnW7E+o8W8m9qe2
8JK7K93+70299u6we59OoOz86eX7qM6J1pVdCpDLMGwh5qGgiF9dgRUhRBIJ
pYa7sarcGnnW76Y5M2c2Gme.NKctKk4Ma8bukoKyNCh0SemAgCT4OuaKgZmA
wkMXcM57C6LHElcFDGKiGVrZNWeYjCGqTlGXhqicPRNMhBgdiAoTO6rWUg+0
MF7W2XvEeiAKm1FCRK6khDzPH57or0Ns8EjV1yoIws0xfWh8EjjUYeASH8z9
BxcaeAo4CwKZ3SsD3rsfTKfaKHsnMmulBjGuqf1pWgbWAo55sqf1Ea+3MEj5
0lBpqJcb7UJoyxrofTaw1TPuwI.SRe2SPrV0PiK0VMSHZh+cahlj0ZrmfbJ7
6IXJ0Fqa0BzDtf5BWzQMEwfrofLrdaJn2r0mW59uA1SvViThY5WZjTswqLVD
1SvTD1SPdQql1VtMjrcXZTiTnlX23wlUEO1pkZrCJgjvU7W2SPGjft.6IXSi
b+3sDzKttHrkf7xtA1IwYCr8BsakZ7bmTRLHuFIwvLlFZUfaEcgPlPpNJ0G0
Cg0E.ylVEyVmbO.XlaRx5enpwigxrm5NAedlsleNNHJ0bpkPQSSeTHqhjYuJ
UvCAZVBMPiAWc0M4rnLNIO2RQwrWkMn.Um6AxtoCIpng5CRSFQ6RTL6UgRSQ
4CQsKskXZ8xbbNScqo+JGmS8PP3grNugEcCSKGbDiTYdCBM.LVRntTg03Uks
3g4j5i3BfsesHu+0yxYsNKGqlmOkCyIwKK2TdsJ44IMXRNoCyIsnovqosG4Z
7dptUv4EyPAJGNJGrek38raoLk5hV6DZTfakJ7lVTW4LwaoB7lv06nbTa8RV
e2pK1fjJspl4CWsemVv56FwE6nb7Fm5e4c6Nb0sixAgneTNtnTHOHGLsdGji
2nPzKt6eb+s6u2ruuZUcm.6TExZnH4FUKnln3uVU2+Rt3KZzAtyhVgh5dJq+
28M+KW8NkL4tG9oc+W+s+ye++8G1e+Ce++59au85e5SO9vc298+18O7Ge7tO
98JEx02c+t+59qt+6ObRNH78+1+k+a+1gau6g+5su66e3tOc66e36u8pG+z8
6+9em5D848p1Wzey3YGq8hda6zjzFXTiKQCFo0zwUXLVamlI0glaX9WOtrmQ
B7hTB8YM4n9TA8mXulF6Yml1cZa1oKfEitN8IK4Eu9wGoIHTDLqVOCTKWLI7
NmcZCZ00HgYco22ue1687IW7HyIAB2o.PIMJFHZ.oMSivoCg0uNsb1WzHY+p
j13B2I3PrMZSxgMMfVmsGnM6sGXY6KXXgFWYuVw0qGr9982bRsEnxI2Bvd8i
IpxiaVZ6PGfVVm5xtN6IoK60cR4lJCsTikroYb0BYTTgBSwS95HNCL8fl30N
bHDxJ1hm1oIoLWF3b5jO7OMD6xPIwrtzhFDtlDt0lQSUqSrmsPvIMy1lo.cR
ZHWrCLiRYrpqEQmwICtPPqPKOxpwTBV+KirrPzYz3Xkk0V.bdoL5EOLCrb3D
rexsMHlYZ4MyJb37ek3Xlba4MymNPeLPSZ4kecUMv3i8Tqo.MElyKuQeHnoR
gCjYt7rvEPyxS3LaGMXsjIam3JrziUas9T+C6+29jscEyN0u7IuxKVrRC7Km
+leR3rdpPPSqWV8qhZvV641CLwHn5gzSQec1QuMwgtXJaoF9bQi2RPjrcd11
JCZfp5qTiIAKHBkedmPSmhj8rXmPEJk1vmMC8yeYVxKT6IHMTvxlHSNOvqSh
LekJNY9qetn8TPlZsAqnyfTCfr0btsxNqRCVWHmrVtRksJYcsjxm602286fc
3NZGuS1k2U1U201Aoc.rCPMIjc.q7fSs+KpCgyfzTK3V9Z6A0Y8++5ix.nCn
PAz7pjhROksSLzZv+nQ4HnlQUgJ1l6h7JUjOycFVlgkcFlzFximFVkGU3rwc
qVDSRtSsjlroniKxBnAMu3PGlhkExJKKvYk9pIMfGKFE6tRRP03RDyou9kan
tyIY7Gt6tGe3w8e7gu7nGt+weX+UO9hO8CW8tW9ee86t+N8C1+6+g86+HLwW
6gG0eVu3qc69G+C2b8e4Ee1GUz92+3mt2NxzW7E929z06e76zggOd88iCCu3
qp32W9W3Oe+0O97mM6UVxmy0H5jDwsY4oglL9TgBqKghRKiMSyYE6RU+UA3K
24ol4kkQG0ja4VtTyPVHha3XQ3mkgVlqMkgOYk6Kc1mn5ochQ4eIInEOlvkn
de8FJsCzmobFwbJIfNpNRaS4AohrtvncamohUp1QHvxWGBJmb1ILfv.UzH.X
AAJq9WVgoq9RCZLC5vnjSUiCmV3cF7rkujLunGcNC0z.yj5dIJeh0BRr56Qm
g.PUiaRg4lPU6plRkAcPQGcrrQLOv7ZrdNbRcfkL2V33FSCDm0EzSkTs0vmW
TW+DM.GqTdEczBrSsZAquuW0LkSequ0HTx1DW6tB23lFlFZWL+wcXQc7gQsb
fxlb0rj0d2YVlbY4MH1y3YV3aSDLW13HXjTed0sPCxcrdAWmJfaURXPy3NXI
LTzHR0rP0kMTljr99MVHnVyGQCVtpPfckbRV2FcsRX3eDOkfXDZYoW7seqcv
qS6rRoFDDSbdsNu+Od069i6R6lO+pv8X.vzkRMrtJy5JrnINzJAKegSXTV3s
jfalvL2.MCMcQ2jXWHDeS+LSX7DifUx8bWMmXSCPMxKM4Uc0IM5zho12swUj
xikBDYpEtc7sde1BLFMUAiqowaq.nYaoodouyJp8s6JV6esV3JVK1Jni4Zqu
9EFJUVCYRCstTEuOa9oeJk2NnL6zOmAX0Lk8pLJ5SJCUlGqIbtje6i.qklyO
AXgr9JHkrl5bsVnFQq.XU6.X8ZkS0TXkFoxad.X4V34Gu5w2H.ZW9dMsTqP.
kjImdiAGLnQokXQSZUS9GF2Mjxob2gspBQYaFUIZfJY1CV6x09cJbSpmhcud
Tf5xVIa9CX4A0JiWCjtPAlSwfBzMjkfRA5BV8fBLCgfBzGq1PTft.XGn.URm
lGrh8gBzG2hBEHoohjpUoY2zWAGOB69PARwfBTsUzjCmwl3Ap4rzhKEnKX0E
JPNDTf9X0FhBzE.6QTfsjKEnzGJPebKJTff0JCghFnL.5Ckg7rn.Kiaf9W9Q
mfOUNFDfrLTZY6F5LVyd0wSKZFDfSY+qG8mKP85zemMPUBA4mON85jeKHNcN
Tetf2Ln9xBcVTe1g.6Ap09P84iZyf56yr60i5KICPiyMTeEElpVChnGTesXP
8QfBCJ1n4hf5hj3XyOLjTet.05S8URgf5yGm1LTetfWOR7ExdXJzGlOePKFA
8oOtAqompOtRiXjqTWH9JXLH9.YfSUzpXXjspgghIwmON0AdOJD7d9vzFg2y
G65PDebs5Ew2qcWhWJdOePKFQ7YM6oZRrt4NUPRm9zoc66szJVVChuDLjR.z
DoYhPAZkHVL2sOWnpG60WIGBtOejZyrWetvWWh4ipdfZoOje9nVPB5yH5zgm
jFINWDQXoOA8UCA2GVyV69HgPpTTznw4fFzmKN0gf9ZQf3aBXZqDzmK10CdO
r3w6U6hVEOEnEDdONOnggWrIDIh0kjfNE0WEhAyWAGrNDJoPCwPCsKGaPi5y
Eq5QXeULDre9P01IrOW7qK04mZ+NnJ0G9OeXKH7eDLXW.A.MU7Tc2a8pFWpb
L3+rdLIjAMZ7j99aMmvvx+4hUcg+SBA+mOTsc3+bwutTjejWcNWy8g+yG1BB
+2Wz.EZo9bXG0XbOOrFxiZfZp.5pRYckxDGz7dcwo0Ou2ZHthGS.Sak7dcwt
dD2WM4ho849cLAns57dpQ8Cyu8VmaoE7hWOO0iUWIX.DzJ56J0poDIS0+fVm
1zznvEr+9cvT2pZuzEZSTa.7b5xKe6rNRS21pTFGryHr1TdXvzicq3vVkNj3
IJGK4FtdxwxTMZIqu99k8jgvpOKuT+SbUmEkOdPAWMnHBZE8OLVhYHNKVmH4
y+UAVXwYIsFhyxIJFg41xdO7UGHaoPtkQVGwrtYxj84ikraBMU0OoO5gp0Zi
XMo.ow3mG7otnATy11mtzA0zhQp2r0EBK5hgVaSKmoL8hCb9zs+UqDCcwomC
9TpfjSHhKONEhztm.ldN3yS276aEF5hcOG7YoIPoo1Tc4qs5D6ho8Io6I.sm
C9bd185w6wV.fVmHS3Ts.H2GduXjzMmKCjnq9j002ZbJygk2yCm5.uWHR5dB
XZyv64gccf2SoY8fz9jy8DXVTn8v7.SblRflMIWntP6URwngJvZH0VCL055f
HB4Rbo87voUm1qjBQuTXBXZyP64gc8f1qHdBuUpOMRgI.snv6kZCUqgEyHzJ
VR1yKO2y9fKKoXzHEXJOj.auNoBVTmnBOOpuKPiTvGrdc1uE.rBQiTXBr504
+hRiTvG.mAE34ebKD4Aq8oQJLAtMCJvtzHEZpCT0Z0iMK5TzDS6tTmMkTLZk
BLlFrB9ebFotJEUCb6zxEr5Pg1niMgfBzGq1P8RFW.rOUZiaTf8ogJLAtEkd
ISMMPE8Q0TudTWCsO2q3RJFMTANkGz0jX6.caD1xQ8dEOAPs50ZSABQCUXBb
ZyzPEbAudzKYRn3Ap8oiJLAps52r3IEt54KAZvh1p0oVKMnPu.4p9KBxVU1u
80.MPBkjFPsDNnSpzIWPSmwoNA0vpAZLjW1oXhMEyDMlJpKZ0zkttfZfljTd
H8oWvDYzyBH+pFncNZflt9Ufz.MpJvPUjJa0NegKzgBAKvZfFCKpVMQ0BNfl
tDgf0pPjVChfFnoLeaMMPizfLGrKhJpzSkbNa4qsg0.MMiuSVfnzw.bPCMBo
D2TmqwthvlPCzXbQ0xSpT.cxPwDU4FpVstR9VRDzXDW1k0qv.YTp0BzRlBTS
SsrdWUAMFoSeRto+zLaQ2qrmJsIMVi4AWEzT189qBZSDN2lWEzzHACVJCY6D
JDkFQW8QSXPFoahqJnw3Bm1fu8GWUPiwROF.hnJnw3xFBaynLxMyNy0lfsQN
lKuJnwXqmpf1DaaPHUAMEoFr1QchU2xjsEec5HKYJDErltPHNP5H.TojZqkz
b6968+HK8wpNbhkLEghVaJnZybhk932ESBzXpKks1TvVLNvxrFDRUeNZRBZJ
+r9uB8h+ihA+ma3Jwj+yEq5B+GGB9OenZ6v+4heWL8OiIoO7e9vVP3+L2mJ1
znUoj9j02mdw+kiA+mjFZlpjzZUpUBbEq4CUcg9qDB5OWjZ6v94BeWLoOioZ
eX+bQsfP9AVYzH05X6otUkRWJVMlZwf5yNPrBjzojLnqJ2JAsX07woUuV0Tt
kPv64CSajZUyG6tXxdFyPen87AsXHBF5ienkMznw4ba7bN6.sGiwf1CwAaWX
.cY3VIqAhBwj1yEl5.qGEBVOeTZiv54BcWJAOiYtObd9PVLh0SJM8QVfRpJY
toSL39P5IwfzKkG.aMXIkTff3n1CP8woNv5kCAqmOLsUX8bwtKlfmwboO7d9
fVPh0KKCRMkpVKwSeUotc.GbHT8GRWcYnTDJAh5+HsRbOfCerpG6vGGAk+YJ
nZyrEe932ESyyXoKZ+yTvVPh6SfAS+MArkfVRPtzk39DHFrekxfN1CXIIbqv
4QKMhw84hSqebeBFBlOeXZqD2mK1cwz7L8mde387Asfv6Q5O9ZxNvItHIc1Q
mT5VV3Xv7koATMXlzUjkhcYOBabetXUOh6SjPv94CUam39bwuKllmwRtO7e9
vVP3+PbPeERP0549GzSgtv9UhA6GWsVA.RTx1VBLkBqNe6gTcg6qFBtOefZ6
v84fdWL0NSej8g4yGzBBymWqRnCY7liwE5nRzPyDgtrNkrBPyL+PlwqKNs9Y
7lCwk4XBXZqjwqK1cwT6LN2mqxwDfVvT6LNS8VsyzXgggFZZvg9uYseoQ8OO
3pcFm40UsylpAJEV0Niyx5o1YS0Rk9aL0NSv5KjsLVrsC7Mn1Ys1VPsypmjX
mw4E851ashzgrtfIwTBJjjrwZ+14AKquXmU37PVPcYwZsYKjKPOT+BNGCQ+I
UvASHCMb.qfFU.FS0uvGmVc0ufygPzel.l1HpegO1cwD6LN2GU+YBPKFpegN
DMznwNNFTSXpwXW38JwP0eRJJbP28xVMumzUiBJumKNs97dkPn5OS.SaEdOW
r6RI1YboOh9yDXVPn8ze75xPERGjxXSeY5DsWLT7mDSCZD3ZZclxNm0vuiZ3
dt3TGn8BgX+LALsUn8bwtKlXmoeZe387Asfv6kRChIj6InkqREoVez5LkaIF
Lenxw0PMTbhaMrjCqTm4CUcPoy3RHj4GejZyHzY9v2ESmy3ReD4GeTKFxblB
CC5jFpVqMQz2pTuprvRLD4mD.CEqAZSIVmRRsTbaYftXUOptlZHz4mIfpsSS
iwE+tXZbltzden+7gsfz0XJllqSTRzGJCoZqO2j3ZPZeBsx.onhtdbRGw0vy
CZ803iSqe80TiQ+SvGl1JcMFWr6hIvYbsScPAePaCHvYDurp1glXxfF9aF0j
HkVUyPgFE3LYHa0cdtAIQWhhJaJANiXLVpU.2nARm1YEOPJU4bqMV1RsgTST
u+ZAPrTaX.D3LhokcJFqSwzErL0TDwlZ04wRm.G.rgf94MSjoGc1VeANCf7f
lrWQYbI0YuJVsAU0z.02NPSRGLe7wxL9W02r4rLLwbjz2LV8dxXqM1MvpsRq
LNYy7o9Z36kSdyHVVTuMNSCIlaikMZlJIYrPkxkgJPftxViUZG08qy5aFcNM
9kKj9lQISBn0.MYQzTxJowjdTZCkAIUZIQ4MnR0TG0sf9lQS0mVRe0w.bPy1
yh9wjqKdrTJILoQMnC.Va1WihZrbSik7lQ7xp4JZjRC55zsRgHrXke73kvKa
BmX5fBwVPcs6Zmj2rlbJpaFwskcMcMugpcajLccCSIk0YbMcXPmgnQynSDzA
Kw3dNyxg7zT2LRdCR3mnNzZbuEM9WtogjzFqobEmGzDsIHq7Ipyd0z7r.IuY
zaoyWb1xalevbYjFfVJ2z4BZDU5XYaaIuYjDrDFDw1mmFnjJYjykB2Fk2rhF
zrIshZNpXFfZIHxaFIKbRC91eFGrhRTWJUYbKYvjbvXHuYjvKbh4V8HoFrHU
JaBIo4nAEdndPlZs0hyUip5RKuYjrrgvJTdfrMJpzxiKtRSZ5cUdyHI2U4Mi
ZfylF.VGO1JPshxyZUtwSqKcQk2L0EUiGPiGLMlSeaTEbKp2ZVCUnV4hsB5X
KodgO+KJH8A.01FFW5iMo2qVE5PWPo91GAVq8q2GqPcVUiSERmtHpMrBEUCE
iNAvDPEiu8Afttm893GyGmHRWj2LpS8BfIfMuDv5+YUpuB5rGS2MrcyrUItW
7eAoa.3GuVL4+bwpdv+EiNBvDP01g+yE+5.+mu7lQcpm.LArED9OPFDojJPM
UShjJPu3+hg7NZ6KRQz0kjllVNxbABK+mKV0E9uPHuiS.UaG9OW7qGw+4JvY
TtOx63DvVP3+Rv.XG5VUGNZV+ommE+2YVCTTNFh6nfV8zHXBEvpfb9oZUq9V
s+Ui6yEmdctuyFmBgxNNAL85LegnV07wtYv6sNJbFk6ivNNAnMCduNbAsfpF
TJi5yrgTUeovRe38hgxNJ.NzHqRKTjnYkYRIl7d93z5y6UBgxNNALsQ387wt
djuqmHmQk9HriSfYwHbOnjFzAHrX02Ql47gV+4pS6Ei6iuUQKCMS4oRihLOm
fbPo8bwoNP6EhDcm.l1JzdtXWGB2yWjyn9be7mBzBR3dRdvRDuYmMs9mrT5z
17Ei6iOwU04wTbOq+QlgTUB6w75iU8Xa9BwExeJnZyrMe93WWB6ySjyn9bi7
mB1BRbe5rGLiInN5lKUo0m39hQ5tbtNvV4cZNOf59HsfF2mKNs9w8EhKi+Tv
zVItOWrqG7dthbF0mqh+TfVP38vxPUYkIwN5YcxiEpYWh6KHWFe1pwVIAbqn
+BQT3vF2mKV0i39hwExeBnZ6D2mK90kx6ySjynNck7m.1BB+GPC1cgGszxSU
FvdUdKUIF7eTaPxLYZvNBlNafgk+yEq5B+WND7e9P01g+yE+5R484I0YTszG
9OeXKF7eMM1T6F8VS.U0YQTtKo8Vqwf7SGvyUc..s6BZVgpfdbGtvTGx5sEB
dOeTZij0qKz0if9b04Lp0ojd8wrUmz6zz4LpgcWmyXAGgI01stTiN9TlpgAE
GcNiZzJqyYLotyZJIpmbEfTJI1DFT8dxp2aSPkKVrAufnyYTiWQcNahNqTkF
aRAaWkMSA1WJsYEzZiOuAoM64LZ+aIoMiZK60sNilH6pK5TQMy6D1FaojqQu
7XVwZRTYfwbU+e4b059xe9dMpen9lUyV6odYihoEiSXNmogLhIFRPqkkma46
02p8uVAa5iSOGsIXBQojOnl.KKNEhSWdBX54nMOcyuqQa5icOGtIIhvRhs5q
tKRaF05yYKOAn8b3lyytWMdOzh3yZ1Xr014QtW7dw3rk0Q4glohoVO9.st2W
T48bwoUm2iSg3rkm.l1J7dtXWG387k1rTexxdBLKHzdoh9HwJi.ZvBW5BsGm
hwAKmIdPM.MOVz1uCqWgGTZOWbpCzdg3Pkm.l1JzdtXWOn87k1rTeNR4I.sX
v6gVy8pXaYCX8h5ZUfYw6c9JFSJFGorZ2CRNmgDgkTohBLOluKPGSvEqdctu
E.qBwQJOAT85reQoiI3heyf+akD2rTeNR4IfsYv+0iNlPIOPiKFYMzF0Wuzo
RJjSw3TkyI08AakTKgpcxsRXKoPerpCkTCmBwQKOATsc5XLt3WeJoFu3+fTe
3+7gsfzwXxvPC04E.SVDpbetBw5yKFcNA21abD6XLt3zpWTMLfgnyI3CSakN
FiK10iNFiu5lATeZcB9fVmuBwu1Lb7b5gKmNiSVWIPWDjHSz2ZEndPR9JBOv
XJKUEc.cRR8PzvoC+9ERk2f1x1L4gbYfxZZPUE1I.qr4cVZz.xBSrZ4TQmEj
2Tp7FLYgPbgDsgTsLnyjLYVxNeQ0i6PQaU0rOExDrSqftXKlvKtJuAs7xNEi
vAM.SkpEgj5JIiUlUVJCnY5jf4ThIqKtzCUdSYWFz4yoRKmZhXsQ+ucmEPXR
m1yMQ+mVMk+UcdatAj.uk5tX0z4MrUgA1th3IplqJw8Hatlb0q.vWNgdCZKq
fTkx7.g1EkPCXtUrdkjMBTTNcrw5J5J+Ck3buE5Mn01ZB8F1n1Ptv0bisji0
UBGkmSM90TAsKjQoRHnKqrIz4MLc5hfkoj.ybHHV57FlVTAwEapCjxi.5Z0M
MATTfwCMRMPM3Xr.paV0jdoVmz4MvBQJcBCGzBG4nUxTlNRlpJdCznal6x5B
zQgdCSxoKzafF4QUJESm2z3dSGDyvpI20kRsl0o2M0A.pgRn2v2xg.c1B8le
7bJUvPVWYoQkltJqFT.ssD5Mbxyd4Bkyf0tKzTEZ1cdt1pk7Xw4nYnMPVfLl
bS1XqOHFDgdCSKaXLSX+RyXgqf0te07VS0vHzaXp0iA.RYSal2mnzTE6e4xK
za3qsw8mpoqqpRZHG5+ulTplhIy79ldWE5M7b1j72vlgCnIZ01c3IqqyxVad
1D5MM.eVCrvhrWTJfDWu7B8lcj1CPQWQnInNYUyyxj9WMTHklRiXxdM0rOFy
EYYOKPDhQE6A19nx19EZajnHGDEZohu8QfUqUH3CVZ9vVz65R5MKKxFvq.XE
hx1Cz3TOx9GO9h27.Pe6EB9.HXW2wbsoy+pHWzUK6jVugPepbuI7wrbQlucu
dTf45wqX2GJvXT5d.mGRbRe2qshNeQM33RA5BVcgBLD0tme3kaIJPW.rCTf9
x8FB8o38lvGKJTfhb7NozGJvXT8d11izHRIRfLVv1X2IOpTftfUWn.CQ464u
seaIJPW.rGQA5p3aH1m52aBernPAxz.1LsnyzlDolAbVTfmYgggXLJfuw8to
HFHAIaeJGq9+YP.18dAsOP85zemMPEhJ3CT2fir94P9EilAsO3MCpu0Qz2Pr
Okv2DdWyf5qGp.BQIaZAqonqKPnAlmn9P8wwf5KUFpfN5mQoA4LwbTo9bApN
P8Igf5KwGa8aHpOWvqGI95o6aHl6CymuyUTB5CxCiU7QEpnP3nV70CluRHX9
RMZnVy5DxpjJHbnqDFRlOWfpCLe0Hv7kZois9MDymK30gf97k9MDacg5aBuq
nDzmtZpIKI5K.oPSJC85feoTLH+pv.gUrXaCciZEIvG7qKX0is7ifPP.VpGa
+aps7yE.6Rvedp+FRXen.88wBRzeVejnkq0ZBJIJoiU0tD8GECY+MkaCIlsU
koJOpMWAM5OefZ8i9iBgt+lxxwV+1I5OevqGTetB.GR8Q3emv6JJTek1.C56
HQUz1UxVuNvWJF8k4jjGrRUsIIwJzRhh6A95CVcI5uPzblSBcr8ukh9yG.6R
Y+4oAbH0m9y7D9XQgBLmGJJ2rcG5ghtVQtzKJvXzhlSLMTkCxORqA1zl3RA5
BV8fBjCQeZNwois+MEEnK.1kx9ySF3PtO8p4I7whBEntvZ0JFcvBS0teZ4tj
.LGiq9glMxPcb2VQwTklTMG0DfcAp0OAXNDW6ijIgteo0ugR.1E75QzethAG
x84ReLg20pS8cZhAGxR2ECtjfCVi6VZfTxi269o5qPwQL3PNuxhA2DsaIcBC
VSkVJKlhBlFa5aQPL3zYuqnXvMwE1OICViztjsF.bqM1CL2vRCGKR0RMtVYR
C7Pm0a2Kh2fxvA7e6oLbJEvhd+rQPFjBpzKYgQz5YfS07O5gxvAbV+4WKi8x
ZSpT0rC9rvOKEpXs9nw8McYipIF4eij0TDH0TU+b6dQhu37WNc6e0B+zGndN
7S8ssUZ5B3KddBRHx8VeqO15+kvOOcyuuge5CdOG9YMIbyjbTpShCmR+zkvO
mv654vOmmcudTebSIckjxEUobNSPoKTeRLx7FQqClmF6d4P0zsfRTo9bApNP
8EhLuQjN152PTetfWGn9b0GN8MoOLe9NWQg4SWMUC9jzzZ3jHXwZNN8f4KFm
6LBjlVWMYM.npZKUFhJymKP0AluPblyp+wwV+Fh4yE75AymqDwgReNw4I7th
B0GBCMSmlsYOLCXsMKpuy+PLkXjwKzZCIqdnjrNagRL1lG42E3Dm8AqWm967
AqbHx5EZxw1+bH.ixIN6CfyfBbcTINL2m7dmvGaFTf83Dm4TcnRZH4htvTqo
O2dU2g4fztAqpaE.XJWq5+gUHDgsna7AqdTzM4XztAUT4H6eKUzM9.XeJ5Fu
n.ycpcC56iEk6bbKOfZnN5KVVCXEys9znYxAoYCVnARnjtNEoySzUpBailwE
nV+htIGiFMXIcr0ugtywtfWOZzLtZEGl6TaFz26py243WUMDK8rebirRtjrz
UHqDrJbF00CpI7YUg6LzFtIlgdycW89w2q4NfHeyRazBRlk8jYJ0UoUymmao
LvKp.3otzi8O9Z8.FddMS9oLNY1FGtrFWZDrVWiimqwQKZMnfRIOHG4KV0T0
gbVWMopQsUfTc0LbZ1UeyxhphURMWRCGmsxlsrJvj.zkEwmsdXrrbT9q5rB1
8rR6PeKTTvpGBckASFJvwZnFGjRSJZ3Bn9RZ6P1bUDsYtxVo1+VczZIqbhlW
I.ZDdjoPXJzYgQoQWNfURA3RQC5RiiB1FxJWYJBtulrxog8OT0wLVmTWSkxg
qhoIqbjcsd.w5t+ZD+0NoqbvrKlx7xFBhZoVKIArp6WmNX80VaeVshdufER8
xDKoCgfytHtOsXqymykq4MDmIUpCrIVMl3cp9SkphyUDzoIJMmB8TImPjVqP
seX+M6RGKqX9CWtiWS08i+5CJe6x8me8AXLoKYnoW1PLqS2JirNUMOursMIU
oxslL1PvPdUz54qt8828gcyN7gLcpBp0qsWDSLGyaoTosNRp07KZatVV1vHL
iooqrjzDB.i.dTWXavfFQQwFSPSVUjxlRqu0ItgR29Phzkv3VJUglljRkxOq
02fsVmtlcqP.Wv.n02bcQkrNzTknVSmIUUyjn7SkJqvCZrgMxZcSp6ELpy4q
uVeKPVCTpZaKUslZBaJ1pI02RpAVvzohH3H77qR88r1boVJRR8M1RGiufxr+
J.7kSpu41xlQLkZC5xDRQzEtnTIc3H5KZJIlZ+0pDURBwkNK02bC2bR8sRSo
KOpIwZ57aCsKbuwaaWYiwh.KYeRsZIjrERJiazomTlNdMnzzkrc6h0jwHKRH
8OKCLijl6EwfYfTzD6atsra4hlR5fNZTHVCJBKpEd3hhWFrttnlYdtVra8Vs
Wh8MVOEw9laxxF5HPCpAqIglU50ZqfSsr9YRybZZ8sNVe5ywIMZeH0RfNMWW
kvBSwNtJVoLLUsQxE06WCKlCkVeqAM1es9dh34TF0AgJMVWtQYErCqbao02b
KZ4LnYGXtRYtTkLNNJi5Xu.sRIyVU2fIFChReysENqAOqWpCHSkBVs8GTihK
L57sjRqu4SpejtnSqAfNCnUFCg6Bqx2RZgidsnFjZcJIrx5p7KkIM8tpx2Rp
ya7o6NFX2wdR+EWQ6l2lI.u7p7sxEY6OuFOrlrAHIQdRfejrFW.WTaHmXa8f
EtPN03piPQmoQBwGO..bo81GAVqxNyGqrpNqnqkLtc5UTSObEvpHzsimBpr5
N6sN.z0JOyG+rWKMJfDWs7QEaYwtnv25yuG0c1TnVNcBl8pw9g7WtRce39Bw
MN0MPkXx74fScg2KBW3TeXZ6v54fcqOmmujdqwR0ENOWDKFLd1kWVmrTgbJq
t21ERqSbdsXv4U4iG.BJqmKV0CdOHEBdOenZ6v74hecHdOWs7V.nObe9nVHX
+TFYcNgck23ZpUAJMO1uy7d6H.FCtuR5XyeNbe89BV4iSuNy2YiSTH387goW
m2KD2uJer60Y8VGY7V+z9v54iYuNqWG5tHRkFRJ6JWHQWZGyUtOrdRLX8jxw
leHY8bwoNv5kCAqmOLsUX8bwtNjmqmBdKPoOjd9PVLB0KqiSMl0IDMaeGaH1
GRuZLH8X3XyOjjdt3TGH8ZgfzyGl1JjdtX25Gpmu3cKXpOrd9XVLB0yJofbK
UvVA.RIj60QZfPL38v5wC.wb687wpdr8dHFBtOenZyr8d93WOB4yS2tEj5C6
mOpEiX9X04gohN.UfbqkK8IQWjiA2GvGa9gLlOWbZ8i4CkPv64CSakX9bwtN
v54JY2Bl6CqmOlECVOLOzfJw4FxYpkpPuh4qDCdOk43nAffFymKV0kX9pgf6
yGp1Nw74he8nD97Tqagv9v94iZwf8CnA.JsRoHThHJ0K1uVL1oOrVNd.Hnre
tXUOX+ZgX29l.p1Nret3WOJlOOg5Nm5C62DnVHX+3V05j8JWjPPF.Im6QFu4
TLx3EK3wleDy30GmV8LdyoXv64CSajLd8wtNDymqFcm6TY7MAls1rdmlDcme
KEr04IQ2H1ZCYBKMcYIVYYxbap1ATbjn67TkA0RIQ2S0kjx3f0fKK.VQnVzD
YBhDcmAY8jn6otm8IZfohHlaBWp4wa.51Uhta4wI3yTSt2BpvMjNIY3NC4ks
sbjggRV4NzvVxJmZCVq9xwrhtrf4AoP4w87Dz2K3yOOExTiAMOjwJcbQiZAh
w9JZwmcj4+KQWd51+ZEcoON8bzkrTy553oEOIfXTBMSfROGb4Ia8cM1Rej64
XK0.AaIM0Sp1IE3Ni8I3xIvrmCtbd18pw5A3ftBMoKZA4ptdMU6BqGFjFhPM
er4GRVOWbZ0Y8vXzKD7QoMBqmKx0AVOWw2NicpQH3CYwfzKq4uJZtsJmblkw
YE8gzKFg5ogber4GQROebZ8I8BwoGOAJsMH87QtdP54p61Yr0GVOeLKHrdUc
pSqH.Ix3+KSyh06rOPxLkhAumjOd.XN7dWftgfKV8pLeK.TAgf6yGodUtunz
KDbQuYv9sNRtctSUNyDn1LX+5QufIWGnBZ2U4h5xaJzSeJclLEjTcUumiF.B
ZufwEq5PoyjoXjuqOTsc5ELt3WWJcFWg3oSo75iZwnSXIVIcqiQDZOPrU5So
yPwn2+wX8XyOhkNiONs9kNCEhd+2DvzFozY7wtNzKXbUZ6L0mt+2DXVeufvu
pVxeNkv2oy2.xA8DnlogJCMJYRMBX5E+YK+eKfzrgokUReSrZLIdrFtxVT9i
606VWZ1jojflKjLK.VAnfBgfoOOkBnivqkzrMkN9b8sue+eY2c+gc2d20Ore
tdeSIDheEcNw55AiJxQpvVuyGdZhSeknmOmxEvAGgMgVXQNztk0ySqjyWV0T
.VVUjHkjgpIjBh0DYw1XaT7Ro7dfFWbiwBmpnUP7sZ8WUduyQ48DLPBuGz.3
X3M1Bu2h5rolHLPMMRYroVcSWWgWIc2aJ62dUeX2+e+e++6+Oe6te227ub06
Tv6tG9oc+W+s+ye++8G1e+Ce++59au85e5SO9vc298+18O7Ge7tO9857fqu6
9c+U0846e3i2qVEBe+u8e4+1uc316d3ud6699Gt6S299G99Or+4O32oyFNnt
vGdAt45a2OVQwed0.+Zi9kkUurODZXKAC1QimkhzrBfL0rksSCBmZMAPcpGB
MEBxcRurwzVSzCgJYBQeqwPVpr92abYCzKhnsfnGlmRjJ9ZAJUJUMGCrkq5r
FxBzdan4g4Dsn9U9yEzLPGTjUSV1N.hRKa0dWez7PJcJZdXdgkkK+bxt7hdn
PoSWzC0THGRf4p1RDYGs9VPzCk2xYod1hdnajzadMOTnXkLNlThkRJAEv19d
Mk7hDZUOTnkUhYmv9CrtGRROF.BoxGR4kM+AMvI0jqZ.SJ6KIbKHJe34bJFu
gc40eC4BoxGhZVD5xmRsnbRDSjj6TiCIGiZ+JQ4iG.hY0O3iU8nwgjCQweMA
TsYp9Ae76ho7g49T7WSfZgn5GPNe7Z08g8KF09kevJwj8yEq5B6WHp8qIfps
C6mK9cwz.wbep8qIPsXv9QVcowkZUmYjDB5UWROGih+JkxGY+Ak6yCo5B0WH
J+KefZ6v74gdWLAPL2mx+xGyhAuGXUlViQvzKsBkEnKMH8bHz9Un0viM+HVy
q93z52fzKQP2WmBl1H07pO1cwz+vRW5nGSgYgPTbzG+PC4jn93MrIIpOrdEL
Frdk5wleHY8bwoNv5Qgf0yGl1JrdtX2kR+CKbeH87grPDpGTaVUEYMSNqiin
yb5CmmDCNuLcj0GQJOWTpCLd4Pv34BRaDBOWj6hI8gkReH7bQrPDjGTjgVgv
Zxx4Fa4dcdFkZLX731Q1eL2ROWjpGaoWoEBVOWfZyrkdtn2ESzCqo9v64hYw
HNur0K8JMq53EMJTzZG4cHPuJDCZOJer4GxH8bwo0OTuJFBROeXZqDqmK1cw
z7vJ0GROeLKFrdbcnZ0Vil7csRPsxcJbuJGCdODOd.Hnw64hU8HfupDBtOen
Z6DwmK9cwz7vZtOre9nVLX+HYHgHT4Zsnui0RuJc4ZIFreo1wC.Ak8yEq5B6
WMDre9P01g8yE+tTZdnFGVeX+7QsXv9goA65rljbVpIqwB2kLdag3Za.0Fer
4GxLdcwo0Oi2VDtxFSASakLdcwtKklGJstbgMlByhklGJMt2ZdnFLLLHTtYh
AoFLrth.GeMOTZk0UyCmp8iEVMOTlpm+uHZdn+0r+u8j7vsplGNUSY4G2+3t
Q+qc+g6u6C6T63mdb2gYBytc1UvSuMsjs5jFyjM2v5faisBRqjZ.FZL1.ktM
kw142p6lv64l6t58edqP5UsxS8NiOithozNLFnK0apSjRwZ8HS93djoTWCBD
0q48yeom52b5Kt7pjnO28mxN.+Y1mDVJU9LAKaeRnACJPq4xo.tXUMa9Rpxm
DWG3RQotM0sWJoLzCoeR3PTdrHnitGY9QT5m7wo0V5mDNBUG6Tnz1P5m7QtK
lJeJbWJO1ovrXH8SDwCrfBz.AzT8RuXWjVQVOIFrdo5wleHY8bwo0m0KGBVO
eTZiv54hbWJU9TSJsOjd9PVPH8zTnXr.L0Zl1KmfVeH8pw3Zu23iM+PR54hS
qOoWHDz8IPoMBomKxcwT4SQR84Nu6iYAg0KwC5aTAS4Zov4J1GQ9TDHFzdk5
Q1eL03SWjZ8k3SQvPP74hSaEE9zE6tXB7oHTeH9bwrXnumXKMfkLQEIUaUka
tWs4HgiAwWlOd.Hn84HWrpGkIlHgf6yGp1Nc5HW76Roumhj6C6mOpEidcTIO
ng+1JIcEBPeWx0tTlXRIFbeR5XyOjc8CWbZ8KSLoFBdOeXZqz0ObwtKk9dJR
qOrd9X1ZeM3mpzn9IkK3DDdvI1P.50ND+4bV9ZBJNky.VGpoTBPRcqqHYR3D
edkyvrXVLRDH0RIRydRI83CMAjAAnTURp+RwZy7GzSpweoQvqjhjED+we1Oa
4e5128S6e2eb+6egDNgMVizOqDqRsZNm5C+m+OIAaieDkdgRO8RAc50.OfOi
3p8dmaYM+El.pfhc69rwqe900rlmDhpWn5TuItuuPuq3WLJHiiLOIxUOM.Ye
j7xAvWW8qbm+UHZTUAPzpCsp5I1lOjub9o+S+SyVHI4SUWNdcGSR8A03hsLU
Q8mepprSEaQV8W0hvUHmDid5LkmhIK4FEYbkbNzyG90+N9Lu7VVm6jZfhZkj
lPNZmxb9Y2bVrOkJ+rxaPxqsryKj.t9bay.optd5bzVIm00wxISkXp3.qK1o
yZU7BdRhh9LiXAKBvwxVzTqlO7vt+urZWb2O7wOLegx6MT9ePSFPEgXcxoo4
eEPlvjO2x+aVoKjKjtndVZrxGZwS0FUePcYpbBsWuF2DcsqWpV0yVFVBxAfg
4CTgUMfDMPEvDSWoXE2cUTF.VG6y4V6rkj64MjyvggbpotoYcXGVxg7nzyY4
CC4kpXWk91XHxWrg7T6ogbtTzE8aUZAGx4TPZ9a3SC4JcBo+FbAGwEnMzrQb
cYWTpfEFn6.dkGiRNoArJsVZbEfi+rS.LhQCZop7plEf1lMf15akIPi4N.rb
gjoCJ2Otn2ru.EMYoKA7wpjK+kK8a0FOTzfyQIqwnqglYxdKsf4L85Wyn7Bp
bXyp9NOZTPJJ7Wfp5hPlXtZGw8x5z9ZiB34bRFm9jAt7jvHZ8dkzYaqSLs++
3+X1Yh7ZJkzICzZrdG7nab07oY1DRdM52wDQxXKyYJAqWhHo4a6Kp7yZq71X
KEEhzPc0k.L2b.F.MRkDoIcJoBY6avZkA17M7E85NHYd.Zcvvm5Nx8wat9Q8
mprCy9fuywGCxodS4l4Vj4NZ3tbXorFW3mSYWHxcYWHhoyOurRbeIW07ZaDR
TKaanuEwWO89OAKutnVtlA+fFBtv5zaE0Kfow5A0xaKpkKpk2rM3PXMzUplM
QUOlL9RZYwbusgMnVNrr94nso5WP+7Svxwkc1N2F3dLa+qtLeZ2DIy4tFOsN
qw6OKvcQdYUVj+4PdrQiSX3fWogiLwCpAqK6K.I3gSGqiw7bX3.Z5vAcBQ.J
qzvgqmxZLbL0d8+g8+y69w6t5Fq9cze71jy1bSHFlpxh9Za0uXEGklyWUmAj
0EFFOKTgIqYqfI0zAJqoFc3hxLzTTpjsnA0wFqkh7l23+4UywVD4iM0dMaTM
uaMr707Xh+7CbMMdJmvKNjXXpSgc9mRLAK6oDmewg0xOclvogW7oi58zReLw
vggI3oyod7XhkWbNw1GgesWCmoijnqJYgmPsRCfTwTnus2oDSK8dyXdgioh0
rypnj4zHIzJbJwKfwScw3iYxoDungtwMd.0vUflP5BSPQpQMEMRVVKOgiq5p
K4pSt00caPXs77hZ4jZ4ku7z2iYhJzxtULtr+A0xW1shgMcH+R5meBV9xtUL
XwoVShcxobZkR+vcVP2SNUR6zfbm+nArRiFTNMvl8VxBXMnNFmJYrz5MZTRF
i77GMvUZzv0OYMFMlJ0zOdyU293N5o7Sa5SerKXM6rSY5zyNkf7XD8EMFOpw
BYsnvnjcJpub1SD0fO0zaz7SlL6T3oRkHYmqjHVhZG+YW9rSg5ulc5WyGTcx
zIYYh0YXsZCkxjYmNSD+hjfFzV5DzzYcCrkflNyKIrzjvlcJl5hwGyrSwk8H
UfpUrbZ7JhFwRRm5CgM6TbYORkjUY6V1oI03yXMYk+TPsbZIs7lZ3ht7ltNT
qvjU3WQMQEbY2JFWx+fZ4K6VwnI7LHWR27SvxWzshokaCodLY+0qOpS3vBwx
JkBh67fteXgV+Ad9iE0UZrPWne.MqkHMdJztvoWhyQNsqcBCFqTgy45l3NVb
lkI8WO0T7oTSYqkUqOkWTCAuxwxkN8bSSo1X77E0YfKOU8TQI2zpsQRZ9xVF
hPgK01F+fSgxulZ5WwEzJNcauxp5uzfTDq5A1dGaJrzWt1xyYg0P04qY89yn
lWJv8v1iYZovhlbl5Bb3LSSMQy9Ai6AGBKZ53kwTypJoutlToTHlCqgur2ei
Vd7.SIJIjtHS0tGawL+jzhdrg5RCimWptPutZBC03Z2K5AEW0DQJWPO7SvtW
z7QUZ7wy.Z0mn+0S5.Ns7QSqTs75MKn+ktZMYUy67GLVoyGziJn+CF6l+03p
sRYl65irFIl+0SFE9453Uic25pRHM6iJcJww6qjMZARiwvaRcWppo9kktjL5
TCB+0q9yulLVlR0Afr9RnTyYgHST5HENP6RUWsy9swEqbjoQvSyNBY03NfnG
dim6.5qczySjWm+F+LNRqu9.YBQolck9FlyG2aTl90dImsYpW2qLTqIDNXmJ
EpYBXcEWv54s9izsgl9qZSyDUpnfm5.MrXCzswqkGnonTsafdMYcjK+w4odo
Wvg46u6ladssbg3wQYhaMFPNQ4IFksqfk5WlrdtlZQH.m3v7TJM8aXXV4JNL
LmU1B61vXscF2g4oeqeCiy+gatSe0es4s0ryHJyGOh94Ja5O2fBu+CWM9CJO
2Q0xaW7VmeulZVh5ZBqNnRsdLpblZ557vAS8PQcQyrfHPswc5bEwAIJ3fZUC
YcwxVMYJFaKkwKHLTgCvPszzn8K5ROxpBCTTfgpb.FpMtV0fswCsirkGF9Zc
fMb1cbM9jidiJxPQCdOkEon+ehTPjsK8flei0eLkllaGdnRxfwE6zuRojq1V
3+10d0SRbqyRu01ZMD9wbZRrB1IMPVKnVtYWfScDCp1VZh1ZBZZ1OUZU5BTZ
X9MIu7o37+4r0jzkt4L3ONnI7Y2HNa1ulyGHicnsUIs++K+W1Qy13K8v3Mme
Yz3IkPjrrd0OjVMmeX1NI0S14W40FJ14lBbSiFSZrDPm+V+E1dZrarfMamcD
cUu7Vv4ur3kWn63PLc9KPOL9d67mlqSxaPo0a4CjaXkpZ7bAzyuP81y2Zg61
XRxB0ExrIw5a.G+k97acGFBpeuzAaumt8e3tqe3wOc+9YGzOc5t9kVZLnetH
Y8eRs.FyO0cmeuywH9N+zhW7FdCCwz4mjNX6WBm+YGzOd5A8mEZLn+BZEiWF
Dq3bhl2O18f9y1YB8ksfwMf6+RGzu+3PP8+gdX7WBBfYG3Od5GXq.GZ0Arcl
JDBfUwlgi.n6K+yYYLxO1ZNkE.GOxm3S.fK85+9iCwj..kdX7cOq+e2si672
g+.+c296t8m4E9k+U3W9WG+V9Ke2U+kqe32c6e8o+7ee7OcUWG.lqW3oe4Dv
FOTRJShxkZ5tS0NpHqxOj13uRFmiXxcgIFOirKkBBl9pX5klrvc07Oc6Gu5c
+wcou1+L66pwTavrLiBGZE+iudyaHYMW4bQphcIYPqypas4mifCccuUo7r1e
yjkrmWYKdhkl0Hs929pc5EcQqCCBZL8oJY5Pu5aOTH0aVm3QslsQeJO2pLF7
ueyte7969zG2gytNkO0BWbViAGMQ.yGpcqTFnFZ8kW6NYTWq4AocvbqQO4D0
ivmmGLq4CHwCTSDImwZIwIq4N1TClq1cpiAt.0rMcfOqghoJh22cycOre1Gq
0RVFutVtoxFMzVYujxnPi2nbprJl9ceb+syueOsnldQFRXlwZsToTgrK.kuo
KcTQU.tiBphKQXt5DN6pHwHOr+9qu5lcu2T1NLM+0cei8b2Sen.x3SiEfxCn
KSZeXaUZwSOoxPmfvirzZLj+H.TNd1fjOqQfqd7w6+z0O+5a+Wie7M2c2Gm8
QA0V16qjjN5NK1XPW.v5oF18VSRRp8KhL3RSC9w62eycWoNB69GgSPCFV3NI
BNlCSCbL7hOG33OuwjId3vOjmGFF+4ae9KGLd3tOc+6d9s44Zue2unWYue+C
Od8s+bwZ8+3ytjV6lZD+jdToW6QYWSly+QgoY8rrUzN+GFLGCaTiR+ruq6t+
86G86ft8vgie3oy7gmmwCukZGd3m4yRlyyBVjoOOOuXF10Y+rd9mxqXW0kXL
LAywsfyU+YKPOd5I6dQ44nblyUSoYgovBLNWmEU2R8rlkO3n2yhCoGd3vqZn
KwxGyyP+xg9EZ1SsIyFQuPObMI475fw77HMxKxrYrmOrzrlQIKwr2ZsiOKdF
dk+Ltt.OqW2tvE4YMK7xp63y+YUlyxxUZQvqxrlarLrnkYwhVnE4YIyYdXgW
i.im4COmWEF6wGN9pO7kY1C2QDkl0yZQ71K3rdVv5L6Y9ObX4m8jl0T215X4
oKoeSpi9MyZgqIrSb8e1iJe2m8c8gqe+Gu65ae7gm2tHpLz9xaFpUfDRJkXo
TzuRAp1QIVn++YtqDrjbabrWk4.zO8H1Iu+WrAPQ5xtxfRDpBHkUOdZ2NS3h
5CRhMhk16C1zIjV+goTVy0pBmrnuOibRgS5AvIZUgy8vf981uYJbxO.NaUct
s6v76suqTnTdBTB2iHPc7bpy0TFdZkHtUsmyzAUeNGET4AWqTl50KwQAMkod
ZINQlhE1OvOA59W68ot5whzFhsAeu4.LWxsx7F78or5SnINWHjzCrllehEGJ
hGS5Xa78oETJdL9.vD6EAylRa1266zOg5wTv7aQw+OFllva5jp.ZIH0m.je6
QopxF.Nkb9+4ByGtVTF+8X7Vb1L0h+5g1uC1L9frYH0ZUhpaNUbRo6I9AWXw
qOxITlzYnS1sf7jKNcKmk2W7kQNgJ4rLYO28FJko7bItCQxycGkRIhGqYsnm
6sLhxeNwkf6Il+6K9xKAM94.58jLC+5d7irilRftMJYsx7J6mZ.s6o8tkk+V
GBapkkp0euWg+D1OSo73bTCJEi1zY0Q6ca.cNTJ0fR+On91HlIpBOFMpqC8Y
7qNELcl9sD6hbKNVDOF676Si7mvu5TvDK5ByvEK71Xs6IhsNkyiS9VTnf4R2
gRLGAyX5iJkX9HlgqxXIVDfoTJi2imzWXwq2sqToa8u9D+r0BFOmgyoRtZpl
SpPFmpvRRO9dpbolGk7lEPla6RMNlBotDTRda2a4dmtdIqE+bIHVCev05Aka
jvaMkuG2RyntiO+QykXDBD1fE8EjttOWGzdSznpIYzbySbIHLhDJCEG84zVt
EnY3p2Tr9xHVASxT0sNJh5G0XAsAuW+8HrwNqkatZSzcv8kQvuS66Hab6HiG
s+bng3X7Z9fQZ2EjILPYgV+9gFX+4WEbO+fMx8FjcS.B2dIkydUfucnQLmDZ
xlBPCGn+USXOJETx2S1b2bhsKGJCarO5LmPa4QrHimIib6Z9G3VyFCVQw.dv
Rz6GFQe8wDlCW26PShI.yLZKOLEYfFMxIGgZCeCJlH7N.bgv19rERk1F0Zh0
hLYx04sOiQmP6OgDRmGCI22jswn6dC6+ubCio8wJDH8YX6cZ+IzoBnz9ywFq
c4OEa38iMnMxgMHJgdh59WdSFrEw+KJw5s9vuR033Q08qWyI8cnYeDz1qa7G
x6MKU48r5xsq++e0j1jFftTHNFtFc4vK2Sn8SASOi6FXIopm0SJU7TkYXj8l
AiyOMsEgMy0PQthW.h10kDs1uC9se7mukxfa8NBIaxE+aALoHCGys373fHP0
ehEGF2QFlkawoi1ykGXwgwAHWehE+6WVKxDirqNR2gi0IWcTv6nHNRt5.bva
+7gZRyDBOqh3Zmp9xJIezsTwuyJI9cFjw5fe8E8gqUJXUwJkJpj54oZKyfa9
h4+svwDBGZO7FSnvGZ2jW26ZpKcIZNcbjrsuZJsc2Pmn68vyo8iQVOESTKgK
ZoVqdIqkVwNVTfWc2dd+SGXWbazZfeaW.CSVStigetIqsThnZkH4HUDx+0Wz
GtVTE6XrrIQmtLdnY29Ww9scA1LCQz2JWui8uz9wHCSwEqQRUNghXIqUqhcL
cO7ac+xlysA0hdt7O9NVFfUh9xL9sWv5niTZvJ4sI0T8ome8E8gqkUwYPS25
gXinrOFbS+K3Lnlpw7nkztQzTsBme8E8gqEWwN1.1Di53nohpjR+MrikJgfw
Z1wvGKaezTQWTGs6vcvjKt768lmuebgPXCGfAcDjnYsqggbtAYal5ljQMYOG
Gsn98onHpPkL0+orpX6pzhXLecZc25CeM6889qpa2+Xy31vhzfCTUvCns5nI
jiEx8aoSJpohvp1KQwSpHr9qunObspPwCJ3Vj0m8FFQ2Fn1KQSl61g41thzv
7yKw6HenXr2n8iQVJEO8RDikpf.+0WzGtVUn3Ai44.n.ZPy5R7l++3tDpoRx
UsWhxfdJEOcpj0BpXGqiasFLHWtOwQ9mf+ErikxCiRRyM0xIAtUxZ0qXGana
thRWlXy7MMdv5OdfxTKk79RdoPMU2j6WeQe3ZIEriQPjH+.LDYzbyaPC94ui
kJB6ZIsFEMUGbSK4gCzTMQsk6XHuMhMJe+BY.5+EDLZMUD80RB7hlKFOVI5L
SI5nDabzJdnBhaa89XDMQBzMJEGxO+YiTMUIsHtXtMrRzNqU7PEjHt+spukI
Jatejf8WvNVJMKZIVJlpENokzgCUshGpH1E1mLQbq41TGsPvedMlZJMKZIVJ
p4j0Wh1YshGpfh5z7s327SuikJ3XUvCkTZVjRrIURIqWJwN.ojGPnAasX9oh
jaQUW41eAmMjbGNp4zgjNJse9ZkqPWKQuhjRVuThTp8+T.XYtybZ1WyB02Z9
ooeWLk6g1FfwffrSTDqPdedWOMiTlPaMPCW9rz14PSfM2DmQJnMM0MtMnQKg
FeNzTYaf8b6ZSywgaCZ7RnctrQwFacWXcJnMMY.tMnIKeSq94PaPQp8maWa5
aLeaPSWBsS6QHr1rIV6b.zl9Xr2EzVGIy9efcbjAaQ+PjQnyRT7Fwa1M+8Yl
PaMaZKQ146YtDgM09tPDJphUW.CODquOHj2qs0ouiwDZKAY3RjgKRcRee.jQ
JjMMd+2ExnkQPZbNxPcSdSc8AHaZbwuKjwKQltHU7bMviVp8r4wO9tP1Rw9F
cNxDdieSYcjg.t.PZLTTGMSgnGDSMJTrCwn3kX2geBuHsyk4NO9s2kL2kZSr
1hj7quQuYCvbtfKfdWxzvsdChHi.xEo8.N1znZdWbLaYjMN2VW1khQuYZwCy
wlFUw6hi0Wxw3EoDltguqW+Y4XSip2cwwFK4XKrL2k5BVNmNla0zsY95ZWtW
XYNFMD9jNcLM3M2AzxkJasaIOrtvhWdSE40hCKy.MofwOnasvUxByCMk1cZk
ci25Mq4JMv8XzY1l+KnAzHoa1qtc2AEIx6z9w.KUo5rH7PPerM3neVHfvPie
IfxhAYDzTzsrDFVzmHNL5PuQa0U9YRjtHZQWAoGDrnDHkeDjxe.REW9VLLiD
7jXG8FsUWA2YQJTFROHTRIPJ9.HcUnktBROHxRIPJ7HHUJCoGDnoDHs8DHs2
qRhz7.OkRhDnOBTkxf5zHQkCpxi.UrLnNMzT4f5SnnYUvptBTmFqpbPkdDnp
UA04AuJGTeDUMKBm0Uf5AQcJCTeDcMVYV+dT3hx.0GQYiVl4uGEmmDPc7HHk
KCoGDelDHs+HHEJCoGDulDH0dBjJkYpzQguIAR+TKkxk5lkjlnbtJzsjDQI0
3C6WeQe3Zk5IcoRR0KFdPdXKyylHmZDBhF6FQEyxWWCjafbGCcPt9W+psIQi
FbPtKO69FMuDWmPaIHacmG2NGY.LCYZaiXCE..ycDfi4WHLBg1hiO++W86xf
NlSaIHa04d47QvXDqA2Do2PFau9ZatO.lxtN2+2+m6Cfs04QykPOPeSihYky
LZKAYxx40v3bjozLjM6zHM.2SGWHOFd4DiRSqeimF4kyRfE2yDZ1oQemz8IU
8qSiAGY6QzIjcKG15z9mu4abtNlwbZKAYqRu.DfyQFiS2yfsnA4BDw9wtHm5
7kJ5Yj.1aNoNx7MtFLm1RP1X87q+T2VPpIyfl.ahe0QZsQ2FnuEESqogKtww
PHEw8WystvlSaIPqurw8QKjgLjomGs8oki0Ps26NnhN9Yzzwm.s2osDnYK6J
f5hCjcd5tF8Nz3NpS20naAZ35VN3B4i98mI6ZltQMtaw31UUDi6ZnakwVC76
Z8.FDqTeNseJznbEDV67qZcy2zZMJZk3CzEl6aOcWc.GCJ4XPx1TW52+zC70
9nyN9GQC9evWjTbScIn9FqqYbHn8+NvZg2or73amh4ArdOOZJkxQJpjJdK0r
G8WeQe3ZooZGImeYKxJCmyaJgNo.9pq8GI+dmXWobK5HKTuEGenAEYhFIwuh
h2GnOm1OFYobRiJoxDRMoS0RlUkZpIc5xcLWaa2bCibm0YWgr7UUk966BTX9
XxcL5yMejx49YIU7VpoK5u9h9v0pUwNlpavvDfDY+uT5+tKDugQrORq2v9Eo
eJtRMP8TrjpPCSI+EKIzDnUw9U77RtZUR6JKn6MINYSvU3ldCyo8iQVFo8xP
Nq+c4t7atSHsVHlWEYngqI9wwsVWBeM6tPE2.fvpBvsgBRPZ4FKjBn2xbKVw
T5cvRz6jaLShkn2AKQuyP2bapI.af1Yxn9eA2JRoK.pQNVppetjFFrB4jOWR
W0.pPlYjCrtmEZ2.28j34u3e9SGPFQIeebS91498pThsn4B5tY0Fu56UcXiF
1v8bZ3WJDKdsjirM6cZ+Xj8biHSM0HxTA5VZak4lOms1Yp8NH9vTW2bOxQBI
SiD+JFKgtO5v9vmChgoLx9lmbQZ2ibcLTvbY7.a8XBMi9A8Mb3pQ8ef+K1Ow
+NgkqQKE6i+V9VUkl7TyKAsURwrmaxBTx7RPabABLibXhYTFt4Tf+g85clDF
1L2pK+GGu3r12Eqbf.y2o8iQFkxHyymIacl2bEXCFToywT+6UxsL1bfhlLnN
BcT2mhkMdi38YdnSbKr3dNs0med4v54oh30vJz2DLp00tKIAktkGqxifUoNr
R3F4XT3lyBIn24zXkeDrh0gUNLL1Fn6tjDQf8qjrHCVom.qmmMhWCqxXiMB6
wiNF0xfjFp3i.UsNnZzFD4Aiq0O5rznkFpvi.UpNn16aL5WUYWqkakR6q7AO
CVaOBVaUgUr0vsXXshtRWAbOVjzXc7DP87zQ7ZPED2aD2vR++JttB8rHs+HH
kqCoTay2YACB+Fil4cZ4R1i.UnNnxxl4JVcSHZL4VCJosh3QLXR50AUsEyoz
g0H2DhF20zGfeD6kDoNn5JaLyovjV7TtzHMTeDykjOybIHF1wJG0tfqIsuA9
UUe6saVDtm4jVdkykCo7nLjB68ghnRHGiHtXMXjEpOhwRrVGTAaaHn3xeEG1
Z6a9wcBTeDikXpNnRv1vr9H5yH6AyCxB0O0VILk24kjbAol+Yc37YrreMOZQ
itSQCmwp+SCnuySBcZydMHqm7a+XrzJ.KuRWJFH20Gk.a754OdZrjIQZp3Mi
jbC5rRF.dRpAc1x.pMT+Nr4WMingQD0fe77hPRMV0jil9w3ir32xvON4hSm+
d0ySY84Aiu4ZaYFMz8BvEG6hxuFoSCaeSkeywBfel.2Ki7O2+me8KWMwbdBo
tm0yeOeAb6d1Ztful60h3lVQ1QUXw6TVs8eIg4JUZSgYjWmQZQzMMJJYUwip
JgIjVs624.52uflBnwO30mu+OStaJZebbkk7NsUGSkbPkUtlityKGiLGc4G.
lzf9CfohaTjozL1ckdtgvQsYvNL8sLWsXGh+UhXL7NkU61cNXhm61xQ2P6an
KoWiKitA5PHcmZCayMRifg63xPil05TRqNTJ4.pqYp+mfTvMswQUD1De6xBY
QhI5laAJ6GR88O2hmnv.lPZ85wRgT5aQ8rLiPRs53PuG6uRs5f7GowIpq2wq
Fxia9p+eNpZHlPZ0QAUF4d2X7OvXO2bgMWdqKJhbQQC4kOjts6a69f6VPNBk
KBdUZEay8SqKQy9YD4rzNsPays62sTrEY.J906V9Ns0eQISwBK69.cBKrSaQ
GWLEKD5aAjh9zkzbT8Uqw4BzNmEh7VDDDxYfj+kzrCYgEvwvkbL8bNlqBPwd
tCcQqLT7qQ9UxvghwWUH+EncNGy86jhtrn+Ciy+sNdibLZIGiNMYpZ.rIpj6
LVzlz7+.0v6ogzvudXwKP6bNlDFnnC0ExEYgEy1Mxw3kbryS+rF5HqkTvlDI
IBSCKDh61e7UjHu.sy4XtirtGqDJLC8HeL5OofMY4K7ZmyBog6wdRAaFt4W+
7+YvuY2H+P0UocNKzFtPP2sM2iU++1FHciG5zkbL9bNlPaQ+wJEGycJkHyFT
KxlQ4eZGQWf14brAuwfal8nGN.yHNtQNlsjiAmywTGYPJAaQHg1Lli9xNvQw
3R5UocFGKJc5MvsQQ03DoRT6NUd1W9t08y4XtCDXNK1b0FbTY+tiVtkUtnal
5Wk14bLD15tgtjKcvDNT09jB1FKYgx4rvgrA4rXygpK.BL2P9t0aQZNHWk14
rPRcAaQ8H6R17SbhcmB1f0YKwo9I31GMh9qPtScLu4N1KcVh9uq.c9pzNmkI
QepHFkBB25Tjni2IKaoeAx3bVFRQbqxcJSFthifAyf42oFJeUZmyxzHMIcgg
9mQ7uufvcxxV5XfnmyxH+CqkT1lhtbPTQWim6dsCQ8pzNmk4VbXpPcLNSZt6
q5i5M5RWEjycUvc9aavIEtY5l5qajxktwEH2gqR6bdnaCRKFBAZCcsCZC524
wtk9JHm6q.n7VOmYatfPbSF9AUsEcJGtQWjzoLrP7p++zUHDO.ZSdU8q2FCa
omA74dF.VeqmzpM+WtMhVQOMhaeFpxUocNKyEuNX+lLycSbSco6zN2kyjL2E
uyYYCLaf1b1vvOO5fiX2mb2TLwtJsyYYt30wdi1VoQLg.f6TA557C57Dz7fP
SJ9c0V75vjaobjTl6OFVyciHhWaDymtyY9xLh7zhif2XF6Ns692q1tsa8Nt2
udP+Jbzn1r8hXaBs0GE8TAAYQuu4fffLms358FVz21bubcegrlbQZmyBCu82
Ckdz6yiBXFdRN3RusnySvkC71ZFWw41zV2OsDOfm++6WT4KR6TNn6eJ+EGrK
QAuL1SI2GiEt1ptkOY1Tq5DYqsiNK96e+eVa5q+QDm7O5JHe8ORe0vD5jakg
aRRzoX2+k0yKxkHfsRRNDH8S674qUlD067AyxAUT375pLJPBZDCM1npbiGz+
VJqRoORURZUTs1eufEO6cj970xJX+ZZUQNOs8lueUeV60Sk7UctDdXpj0oqk
rVbA6WSqrye38qbk7ITBOLkr2NUxZAEreMs5Tm2f2Rse8482sWuW15h9qhDY
+6E+5Yuf2muV8B1ulUgs+rWurToxqURtnZoD8ZknpzjB1tlUkv+r2trTsLEq
DK1rbkuaIZJMrfsqoU57O79UJKr0RrXyxI4sDMk53i2uNnZs+Y2uzTVXqkXw
llRzqVhlRUKX+ZVIm+yp8RSYfsVhAaZJQuZIZJUpfsqo0M+O79UJCr0RLXSy
I5sDUkZqf8qoE++O69kjx.aoDK1jThdkRTUJVA6WS6fA+rpujTVXKkXxljR1
qThpRgKX+ZZaX3G99UtFzPIlrIoj8JknqTfB1ul0KIl1XsOe2pp9psHorutj
QYlv4D7VhhR9yirwAsCi4sH1TaWedGh8UxErd+pDC1RMl690WzGtVRA6WS6o
G+v6WoLvlKwfMNWlCThhRFKX+ZZiI4mc+hx0E.Jw.fTyCEojI4gjZBkHTImM
RMgRjRlFJBUPCf8.uJononvrqAXnbOlg21gM.1IzVBxfOqcutmhvQRz8cj01
5tYHp6SVSI9UZZN+gMmPaIHC+rl6J1j9Vyju0nKN.YSeBv6BYzm0JWca5ihh
E0THa5ikcWHi+rF2pa8q3Jna41yl8rR2EvjOqMshswXyuz78srtt4BC2iBrL
LZrWlBQo.ZZzMQ.JxUC3W818rzdTkgMs8u9FokvvzOqYu5lpQ920a2dmwDbn
DskiXPri9lNq9g7KR6QEF17lH68vwrOqkwh.Zaw3A7GjicXqn8d3X8Oqyyhv
q7hB9Q4Xy6ns2CCa7YMvVDDYSouKE6.fcPCr8dPF7gMrV+.6XSdyZgiJqp4M
r1aBZvm0fZc60nM4MyENpVdl2fZuIngeVCoEccZa7a1KbTI1LugzdSPi9rFP
69jMk5IuqMuAzdSHi+rFNK51Y6Fam6p1QMb1aBZxm0fYQjFanl6p1QMX1aBZ
5m0PY2mD5u22.O.ZGzPYqGZohteIC8QI0P5T9130rp7i+BKd4ia3WK9JsPX6
qZb9CWqL8KMiuGlbpVqkbKSpQI07K8WmC9v0J0iNWx7lUJY9k5lhrwtcLrNP
VoQq+WPmvM27KEKIhoolim+5K5CWqJFFXDxtmfj4tHEyOwgZ7O9T3VRM0PEr
l6X4TMURLtSMyPWtiQ1lDsZMbXPDMIt+WvcrTu9LTxqOiIzxwk71yolEp+B6
e3ZUQmHmjnAGyL2BWCAYf3O+s4TS4UoldGep4tpTxT4UxMITWsiop6zu6lqQ
h6dAp8+Bj+lazmBkX2AjRVOTh1Y.qXGq21Bui.l6sH7gM5ufcrT9czJI8Dgb
x5KQ6bp435Bm86QGHIhOGL1rAfL5aw7.5Q+5hAruM7agHpsleNar2+3lPa4d
RjyUwawCpTyr1kWD7C2tVFoCZS6ZqeTdH9jFhzR4xTqDCvaoTr0JwnmlVvNF
25uMJn+o2uR4vTIi1dokahdWhgHoFNTK2uh9iAFC9mVXMWSj+FtgkxgoVIh+
aP5Yv9muVsTC19RTglSC58H5OCJgyOWZi1KMpMMlmJ+tFUxBcmf6eZWLSUsQ
3bZKWiZFiEpX6KgCnPERr3TiBKtj4t1q+TV9HlJc9a0KJr4lBv+WULye7A1E
MArZZzE8FQm74ddWku.FtDXm+l5hIaB+65NO.XtS3cCCGCr8b6eH2Ivnk.67
WTW5i8AzQBf49pBCycYyUDE+kR2IvV89kJd96oqwr1BaY1w53VLzEhb.0+cz
WMdv6BXxRfc9QQErMRRcGaDYUXORiLP6bz3wtSfoKGwUmqWwOp9l8oyAl6D5
Vy78qHn0fSOv2HvVOLButc2SSJY+Wgw7gShoWEn.rOgxtgjR9qMrkoa8J6Sw
Mj9cQGSSBzCFQh0mEnuv0xzUYgjCNZnZVarFWyevq6BWKyUE77zvgk1F7Mky
GfqoOKzcgqkYpBddN33.xEyIY1ul93I2EtVllJKjGxVLx9918qooxnegZqG4
EWL8fTfardQZmKmc9CJbWxYWl6Kv4Q5ycqXa7MM9y4A9Vrq+v.bmZ+2uiqqP
6A7qogy+t3WKSEY37bERZheRny+f7qoAS+t3WKSD4E04lrGUX4m770zXtdW7
qk4gbagyfDsYTJWKlXgzsYl55.JsvUP11TKkiESCf2c.rTYuyQSaP3wV7xS5
sWK9xzAnaEjzabpwpXGjkA0kb6ODWHPTvJ69Z5WnmLM2mGIgIz9w.KSjwWE5
GnGw8SL29IGjl6d9tcE8XDyonES.QmFLxT8iB9y6zV8jdMKT4+DnpxLnNObP
uSa0yt2rPE9ifp80mO.tYDJeV.hdm1pmv1IgJ1+iN.qyf57PF8NsUOaWyB0+
nCvzXFTOHHRi0PEdBntvIpq.0iBqTBn1dBn19Ln9eEKMORSSncxtp9.XcUrm
tDVmF8obX8IT2H+YBllh04QjJGV4GAqRcXcZTpxgU5QvJVGVmF4pbX8Iz3HP
c2WOJpSYvJ7HXUqCqGDwnLXs8HXkpCqGDsmDXc7HPsUGTOHPMIfZ+IfZyJCp
yicSBfZOBP45.5AQyIAT+TKlRk3xCtj7.IUN2VR55yiVlf2LfRVqTmVtm.jk
XooEYwra65dNW0baYiQN3PvAPvvOE9+9+ncq95QZbHB2i2trOm1xcbI0DFpn
iK8jonz4yNZoAaQeOwsLdDCGb53dwC5FWtwp0L+mGSnL2KhqR6zgPHu2Nujd
qKcgcs92z7I+KV15je57YEs.xVaLZoXYbai7yh.6XKF3cBeUZmyx7eZ2cT0D
29Wr2rAemrr0oU04yJZ+J2VizbmxDdyjH5BlalO2.huJsyYYwTGrqhz69dWn
dCtSV15D157YEsvnayDm6TlaJ4dXZ5i3Yr8+NcUZmyx58sF31iGCHYm4Rnbm
rr0oB1hKlhtMfjxxFwkXpqpamlEIENcUZmxxjF5mz86t9eQw+YbqmxVljYHb
NKKR1CNorrX7R5xyaZz2OLUE4pzNmkA5VyMYjhmfWE9qugahksV674bLhszb
LrC68IK2T334xQjvqR6TNFF83M2La1OS18awldqR+WFH1yE9SJrY8bB+ifS5
bB+LfCqQyMf6xzNkiQ90R0cTw+ezcC.uU4XXpdTwI7KK5TW4j7SswlqYiU18
cyYAVWuJsy4W9cRNF51HpgAFX+FYXTpVDwILL2sUMoMrj61pKiwcQs4tnKlJ
5UocNCiZtMrgeuw.BtG8Z1ajiwoZQCvI4fKsIIMgkHcy4.tCSf+ahdPFbUZm
ywX+aP5.zoVzBH+Jy6tINljpwEbBGCbONSZAaDw1H1kc2Jd2Hc2GH8pzNmi4
+T+ea28fn8849uZ2oXLMUiC3DNFAabRCXcF6Vfot3Bl7qciAbUZmywrXXs6F
7iZjqAwva+F4XVpB2+DNF6HKo8qjMbyPhobs0aQgNpvUocNGq6JUo8mBEbqQ
7+Ltyak8TEN+IbLY3tLmTN1v+ot8nt.G2nI2jTRtJsy4XiwVDvW+H190xFcm
V7ORUg0mvwLxkgjzS7lKuAn3cmbKM6CFjqR6b2JA1+FjvkxniQBiwcZcAjqH
mOgk0cnkzdrHp3M0c.zE7Yf6RyIBxNf14rLbDIkueuja9owv0pahkoYhpnc9
8R1k66laywC..NGC+m7o.11iTC0in03hYb.ybzLgI2Pd2fj3BjcURE1YMQxC
1Y0+qVX4EGUiyd0KzGpajwvsLYNs0vvVYyuc90RNhzhaookig0iRV.UWfSjP
wHbQRmyvbS2b48Rz6vM2r+FMtSFFsjgc5kRNhyx9DBHCCihmezc6y4utQ8xW
wsHOoyYXtkaRy7+UbsjXzyTr6jgwKYX34LrviNPycBy8X2kJGQgtObOl653p
zNmko8MHbNJtSZ90yuhZ8MwxVFOww3bVl6SGIIkhY5FEcsZWaBSMWqV+pzNm
k0wv1M2v0FEEjEdurrkwSbnmyxXNZrA4NkYgwawiL21SjGiGWk14rrQjbPX2
uZ17uBWewcxxVFOw94Gx7CBo4XnfQSQmiBiw8uhGekGEWf1obLT5ta6Rzd3Y
2OI2BkaUT1RN14mwbE+t8X4D8ips4VAXtkatgoXDk4qR6bNlaFse6Er3wK8+
0GeYO18vwV9hkc5bNFO1baCycFq6mGCC3hGicD9.1uJsy4XtUziQa3l.GQdx
0SH2HGa4CV1amywTZCRZ.aDGZ2mfVzPgZNBsulQMWf1obrHl0zvcCU43OM2W
26zhrkuWoYmywL2WjjVvFQh1kg2HLLAv8Y7q2E4BzNmiE13LZtErtAYlaGGd
mR9WZdgwmywFsvVyTmwbSQibNHTp4VAL.RwqR6bNVT9yCKzP3JpZt1x6zMok
VWXKbqrwaij1vRby0R.g4at2wcvYzWk14bLV25V7HR9ERHrGqeibrUgTT0E9
Ut6.XN4XjHtMVTTKcweCQDuJsy4XtGBggaj3GvP2io6z3h9RF149UFgpwRZN
lekYaHT2MU.iXpg7kocNCyjMmuBPvai+13NMtXrjictiktWcaVRywnNu0LsE
I7bjertYCWk14bLWTpQsFtWORQxlcqAuXoM+x4176eJaZR6wbUdQCjywTTfc
tKP7EIctORMKd1IQL25B2ezlbWLrb8kerjLhLWalrjDCtmJIjOuzH78KWWha
7LLT+fa2UlrWRyDuo8NzEoSgk0fuiQwjibDiaMNlvftcg1Az9wHKUe42FkvE
S0.F6kjxyoZo7iaIkmSM01WTheVzKAiGGjboEcn02ygklMoMN61j3964dkoi
HoXnna.Mm1xS.ZaTw0hHjHhnQDu7yztvcaxQcLzGj7ZgS6md7wRc9wpnkZxV
pdHgURmtzzJ1wnnelhRLYAcqUdkOZ+ZSvwSXgHSIji8uz9w.K0DIvJo0jZoz
xYkj2+FUwFlfailuY35bFjZ6sM4e7crT5TsVIbwbZ4JwtDqUwNVDcRTFQXJg
lu2wzeA6XI.lMZ2wrGK0ZyGLy0nGXoOXhqwOvROek62+JS84Ks7.KsMeoG2+
Ri18zJoRs1GbLSu+kFNnmraOvR2ZGr3eX08qoreUqXjIwZJKJ0RbzRsJTA0w
MxMwmb67hjvA9avnAMi8qcjNGYbLB3iZ6gZfE3a2iCy1hNBpPn3dPLvd++4z
BMWQrxtsS7vhRbDmS6GirTFvVxnZj0TFvpz8boK0him99Sj6X6VScW8338aI
BrWMjRYKpicgZRqyBuOTFFrsE0.XGage0t6zWk1XbPCXqqRDr3dGiQ779QCY
O+ci+TfnX.lSa4FCkiE9slwWU5ozT1sqkX2tlxtcsD610Bram5Ls4hAstB8Q
GhAm8tfPaicYhQraGQiXBnSDZ9FseLxRE1sawrcMkkEvsLuWXIk5coD06RJ0
6RIp2EqhSpRz5RZQoL0k3sTZvOeX2jTGUkRTBJoT3JkDhOgqXGyvMyhF0du0
hNAPu+WvNVp3SIkDSeIklGojX7kZDEtbGq6lIFcOxHSqGQsX1+4ewmbiAKtj
vMK+fQLJ0z06v1B8Cr32T3pjetvUI+XgqR94BWk7yEtJ4GLbUxOW3pjetvUI
+jgqhyYOaIddk6AzWn+azfML56rfnla76qLs6G1hkjOkPIudWNWu3R7.Imz2
U6Xf37ajBWBFhRbi+4sXgSY8LWRT.3T1yxk3ABiUriQvlpQyvpMFlF4e4eA6
XordlJwaXNm8rk3ARpNUNedWiU4wVaOPg8MWjN3tGvwjCPGuJ0j1lDEiezRf
h5LezmS6mBEZTwgOViRRtMhFmRORBe5m+vGkR0EUhi8TJkITINSQZE6XZais
Ho2G8XHLs2Uf9g0HSozbQk3WOkRWBUh1ehpXCKF9tu8JC+z6XozbQkXIJkSW
RIZ+oJdCfw.15.nQqNRhArH7WPBMl5n337d1ugRTgzwrAwrtozqDMcumFAiF
GsAJXvQW4DAk2jgDkLBzGXTMyyosbGSSgT8VB4DlRmJVh1.rm9YT+70JWJPU
hFULkVNrDAzu5vnqmfZmmNAtf4st6l6+Hg1uxeXQaLuF.lPaMPCWBs14PiwM
S+WkOmBso4w8sAMZY1bZmCMQ2rFlaWaVBOeaHiWhL9bj4tPn6MvnDHaZlAea
PSVBM3bn0csJVxqZSSg1aCZ5RUN8yg1n+a14cJzllZV2EzVqLE9CrfMFr6nR
bTspriCHZC7Gj+DSnslMskwRseNx5iMV9WYH+CxbW3YIxYLq2XkvHaOl+d6S
nsDjsTvuHKpUMZiFsVJjM8couKjsTtufmiLv1n+i15SQ1z2u8tP1R497XQY4
D2RjT6YyiL+cgrkh84EohJKaX6s6Ycw0coQuvCioClFSrPpO3PudLVJc2MBu
U3KR6TYtGDY76Rl6RsI7hTb08qD32t+NiK3FtMBIStYa9U2gik8wl2Un8.N1
zHSeWbLaIG6bScaFEihb4GkiMMbp2EGquLlJ1hDD21ZuqW+Y4XyBm4cwvFKY
XKLLu01FRJWNNvnoay500wWagg4.u0Go743f3icGPKWllS2R9QbgEu9ICOxY
rt8eZP0e3ZgedIY3mIZatgVcnaQ6jGf84K2ObIYfoh774QGJZB2aQ+9sa7.E
sAecQNFXVCnonqzABobmDcn2ns7jXJITa0A0ChVTBnxOATOO5QqfpDGR6jF9
keTzidizxyDvjHkqCoGDMoDPEeDnB0A0ChtTBnBOATOOZSWCpGDsoDPs8HPE
pRpzAgeJkToOcxvmCqRuNrNMfT4v5ins47PTcMrNMDU4v5int47fVcMrNMnU
4v5inv47vXcIrNOLV4v5inw47.acMrdP.nxf0GQkCS0g0CBcTFr9H5b3VcX8
ff9j.qim.pjUGTmGslDHs+HHkqCoGD9lDP0dDnVmMSGENmDP8SMYJUxZYmmM
tvqYK9zrw0OfBajH8ne.Lhw04tua2P13B4xbnRRhYHWmHojLr4aSMti1iJIi
n.8AwUxLG57dBdSkM+tF86cwhiybHs4G07Sb.2MizqR6zdCaCvsNYwi9E8q9
ws0gqgjYjz4ME7nsVRv2a7GGwFFaJ4hjvA5PQPytJsyYYnECBA2v5vXLibgb
2IKiR0ZHOgkMnMTnbmxvXDqSJZZevl4FlbUZmyxXLldutCIZW4XLjeqmx3TM
mQ3jWpw8gZzxcJSfMW8liKzMYUh.ecUZmyxb+y69pG15KcwbGWuSVljp6HdB
KCC.mTVlZa6CFOnES5Bm.4pzNmkYzlB7n6d4ahak43VuXpo5lWmvxHYRSL5.
1f0h1Jj6EniB23iSl5HyIcNCq22HRG8FYiHR725YrbMxmq1zml+HiQj7hNuu
0hAbTWDVtJsy3XNst9mw9bsA08o4EemGwR0PYNgigtuksTh92iG39nGy08wh
KSuyWk14bLk2PyOE1c6Wcqlg6Zv1.4R7M8bNFM1FbK2YLyOOB8djxTTS6s93
pzNmiY8WYc8HbmRae4m7MwwnTMTkS3XBEiHkbmwhXSxwTQkPWVSXw4UocNGy
MwALz+KWHAq9sz6jiwoJA+S3XtJsdNaXibvzMxBfvUTUDn0uLsS4XiXB9pw7
sIdvN+WdmWJkTU.+ILrNDVZl5HVDlzt+CGQdCIg+4zUocNCysuQcuonlyyZQ
WCztQNllpBzOgiMhgJSNwXth+svJS20K26XmI2nqR6bNFYaZvnb8qp6.hH24
YLKUEfehWRgCf4recOjs.XiF5pyDjv9kocNGSBCbZtTuPfpqWEuSwX8Tks7I
bLj1jjliE04n4VAPH1ZM+tVCuJsy4Xptwpg9wKLRXY+v1MxwFopa3S3X9MBI
o4XCywf3lLnC2TfVDB0qR6bNVOFjwt6ZphP28t5NuTB4pa2S3XtCybRywh.Q
y9GuebnoQfFD7pzNmiMb00sXPrGQt.c11c4jTt5Eulf.mK34kTy8PlmjfV77
gxHR3Fm6G0DsFoJ3qNWft4++gaEc2zG2kL+7gqowMDh6naATCc4tz3.Z+Xjk
pdwa0Dd6LoYosnrdfsQTHUXaLFt2tgyqBz2HWpAPPDXJ2ii8NNh01Pp42Yba
cjduq1TRKOmeRAyUU+vTXJuCyPUyDTJqQIb+nDqZyzMB6Obu7CyQhVtdLSIs
IhVu.gLZDgIbXM2bsn+v6JVmH3vEnkVHCCebBV2RU88sRdWqVp2PqURWEnIU
riglaxrpPibQVwzUD9u6BMp25l6LbB0B+KseLxRk6+sRZYIsTFMzJogk0vJ1
wXxcjt2j3giboP8u5YT+9tfs2dTxsiYed6QokxDkQIMrrVNiFJwLuLKkfk.q
TMXlSSaJzkZtE4r3fbcvcie4+qq5Zys8Fhr7ynXfm5GM7SCC+PPWHW6UuMDN
5o9yn8CwEMFUbneOf4wfkP4vYltx+3G5cWAScnuh15h6hZp0pWxZoUriYj62
H5dJ2fg6Vs8WwNVFUlc37jD286YCG8g+Aal8Oyz7n9o5HIr6mr+ScOcCjcfA
QuS6GirLxOnR5i7zfRcVDuil3axEWNO7tL6aKTmEr2AvkqH5d4zZwPi0EcpB
ywiTFguvERtEIGc7n3reJ706nbEZcut2h93k4GrchQh2OZP3DGxmPa0tHljE
dTONG9vEOkkB8ZDj0R7hl53TAYfpjeYUIw5b79q59S8fDr4tQGksiqlzMAHl
nQr.aXPjhtM9f6vHOkzpqTsW.EV1XUGmCz3o4yATWot+CUazateuiA92ENw6
nawm7zz4M2viNM4+zV7e5gnfdWhG4.4811CawzvR6i1.v4zVc9dmDpM35PMZ
YHQ26c+87e6eDbXF+c2R.XyMaP7yVtxwH2J0+42dO6qKOT0ZW+xSRv1DZ+IE
byb.WQfa9d+XvBsG.s7P9SDUL+TLRaQZg5lxIQ39s+pND+8Ih3GH8e9XQLiT
Q9I1RQrHo+GzrAdJfhKM7T9i1QsM+jHYraDWS8Sk5+6HS8mPa0MTfb6ozenF
8YPU5vVT44t0vnfs3wfeRntZWE49endUT1MRoqf69STitTLhIaF5+StDILFr
p3bZqthaxsqhM5OT366PE18BOdchV22+.Z2+yGCpq1UgEEB3Qaqf6lEGmFGt
3VJlRn+u+OWEB6mfMkgHwcG.g84zVuCUY1VguGWkj6qywp67HGEHWSi2l2us
BOIVWtuR53OYecOaEDws3soleLs4+TgF3r802osdeUysuh+QFKMGqba195Sg
0k6q3nW09p+aG+o6q3SruhBW09J2U6OceEeh8UP9iLClcyijVTtqMfcq3iJX
B699Zj7Y9tJ4Nwf39CTMg1pqc2j6q.7GYI7TrZ995nqMafpKxE38B.4IvZO0
ypLJIfu8JxM.l3Mn6N.h9oog5+yuBxuyrciOMWD.wn+KOK2.di1OFYodVktT
BWL0ypzsRVqJxM.NlzRp6R2nEIsGIH9imMGTO0CczwR3hoBTdmKYspH2.bm1
1jHSlFt6pl6Yiv+ErikJh+1nDtXK0NVMRESsgcpMDtSX5V68WrGCssfAMFIW
YBEZUHVP2WtwXuMmz8+crCn8SAlUwK1ylqYzsrwM1gP2RGbu5E9Yy2TxRsmY
k7h8VJUKVIOzkUwK1yteV9YQtYB61m15R+mW3gka7.SkvESoZwJwX.ipHmta
51d6lu41fZtwm+EjS2jkRbuVh3dKk3dqDw8VpTA677yJTOOUZu672fQdDNC+
p45E0KGynEEuAhgMu84z9o3RqPZuaZ3FSQGerY5.DFse9ShZJo8ZIR60TR60
Rj1qUHsO32QBvzPIdDmAOj+B1wRIsWKQZulRZeIs6ne8E+Y6XQ1k3Wz5xPAW
4JavOu9YMkaKZqDtXNMKk35m1pXGS0MrOhG7Wcg+lwi+B1wR.L6VRcO8i0fB
h1mFeYZafQGfD8ebyYV5wZPem1xikt944x8QHkau95iFSiFU1433b49cZK+0
fjTFOHkDSKIk5boDCKEqhq+cbK57THi9Ivtu+7WfCURJiGjRT6IR5b08yWqb
I.cIlNH4xA4RTDs+mx5NkHrnAXCiM0cO720DwDtE8eSZvtCoCEoH8dlWovSn
sFnsLCsZK520j6TE8ckryg17zw+1f1xVCTaQ6slsM1zT6ZyKjuaCZK6gOsEc
yZ0UNu2nNR.soU71sAskcam1hlWsE0+bt6ZyqqoaCZ5x2KbUypdLwf1Cf1zB
.5tf15X9O9CLUmBL.feOz56cbp38TN3kLmPaMaZKQltvJj1lKMW+NxhjpgFt
Skcv7CfwO8fW7aBskfrkYxYeUKsv2GTFRgrouL1cgLZIxNWDh15avapqO.YS
enk6BY7xfsZKp27nxqF41yl9fD2ExjkHiWTVfhKY6s6Ycay+7s1dVQqfsK.o
KgdchQmZqo60J3UH8.C2l9b.2kD2k5RrEmwcYMi2r.XJSPwVHVJF.jCMh70q
4HbdROfeMMn02E+xVF3pE2bhTv5MyJdT90zPFeW7q9R90h6itbQ6cM5OI+ZZ
.auK90XI+57YVgeCXyvbNaL2Zoayr00tZuXrd0jXb3jxr04As4NfVpnYKsao
JbuvhC0GzPHUss.ELcUob4AWEApgGWo2B7gqUpQPxhXBQQtq15tAwDg6MU+W
mziADgEyNZH7ooKmESn2ns7pCJGTWDinq.0ihQTBnxOBT0OBpQiIvMRO5CjG
Fyn2ns7B4NITo5f5AwPJATwGAps5f5AwTJATgG.pqhwzkf5AwXJATaOAT60I
VZdPmRIV5CGspYwpVGVmFFpbXUdDrR0g0oAlJGV4GAqs5v5zPUkCqOgBmUAu
5RXcZvqxgU7QvJWm0gGD3oLX8QT4X0cF9nfFkAqOhNGstyvGEvmDXc7HPsvi
vGDqlDPs+HPEpCpGD7lDP0dBntHXNWBpGDLmDP8SsYJWN5VSz.REjiQMQ4fS
E6lRReHldPbkLMaNeVNohtABt2.fZsN+Jo8NNMab8HMzFtmZTu2nqRquZp6O
6PXoEsh7wtOQsXH9Ap0FXCPv+HlSaMrLLU6I+DVl01hVHWFVl1ZtmsBa8Qzo
rBmftJsyYYPOFtXtkZw6S5tLa3cxxnT8G7SXYcdyOJj5TlBxVLLVcwlptOpQ
wqR6bVFQaMQT+uTWih80H+6l3Xbp9y8IbL+9yvsOMEGiwsHINF9AAmkDczqq
R6bNl6Tq5VG6mFAx+HjuFNQ2DKSR0frgSxXPvUylSTlJ1VOdJ9fRsgDiWk14
rLkcwdtWEznyFABX2IKSS0dkOgkgxVmRJJSaa8AGEClesaL5mcubNsyYY1Hx
BIm052JYQ3akkkq+S.mT.iXZNVD.rl1TSitoRLXG0qR6TNFysMvhoslzhZr1
Yb24grT8+gS3XftYVNY+QXzfNNnnQyXX+UyI4RzNmiIhK6mh49p6uu6.7cps
DS09ENggQsMCxI5mUWb9.GZKFZZvPgKR5b1kN1SP4g3JHvPIwcxunTM+fS3W
LuoIMgcOddt8tPLkBQnMjKR5b9Um1FN+0h1GjBL+0Lo6d3WbpJz+D9kz2jj1
utOeFPm4xhKtogmc9ZJoy4Wi9jlY9cwujT0G+I7KCCiLy4hTyw.arycktNbj
xWk1obLAv89tUzC1Ir4JXuSWjzT0m9IbrdLiFyIAyuZsYt0ttdezu8Pu5L2W
h14bLru4B5gHymH2Za8VMd0RUG0mXt+tueIcCOhwo6obquOXu3d6xzNmiwgo
ML5NfJhzF.emFh0SUGymvw.2R6jFhEkpla3lKxAL+Dg9U.FtBsy4XtmAtu2j
aBxvbejt2akiT0Q7IbL+FAlzPL2M4MEiNV1vM.PrdqeUZmywbYotZRWFlL5C
QG25YLHWsWdBKyuRfIMEKhbKJcy85yMCfsAzuJsyYYcWcs6xt5Vr0iWbktKA
YYbOpjpQLSTRglcZT0ck.9oN0Ms24psHSIiIbn45chTgR5fRs993GvsGaSl9
K+TbjJkDKoQBjo19KYndkZqohhiFG4NDbZdKn9dpaO.aMxsIws8rCwFM89of
tKVGcirB63ciGvwbB+TPkpsVADd5aiEY1laMvvui6pVZcWJwd0w6BMF9J3pl
hnc1axNFbq4I2BZqEfq2Zu5W1uS6GCMIQv.5vo4OjuArYeetjNbz1EdfXTy3
M2HWGWp6hvvU+F1sDcpzlNkvOFTYD5YeqahVTxmkaw8CKmdEv2oGg3M.hp4w
sdoiXjXjCCh1yL.5Q+1pyvLLUGEBKo+zfo5nP+5K5CWqLuVIbduR2Uz6VWKT
j0eQS2E1G+VQs.tQQeTF7avtgQZjGJ9uG1FCdO3Kw35U18OdBseLx3TbQrDt
HkZs3RVKrhcrgr46AXyEBEaeNGext.nP5cLm1OFYolizkzRQvTcTne8E8YqU
p1xxpcLr4lJ62r5QsO4JzeML690tPCiWXH1LVui8uz9wHqmZGSKgKlRBrzKY
szJ1wP2NkQuw6CLS2JdF9KXGSRwEKQpXpFyBVRysAkJjJ5b4sN0h5tyupIxq
GP9mdGKkTQtDohRJohRMRESAqJVItD4ubeScWxcWW7+OcH6yGie3yFoRguNX
KjSMa.LEo0BDdUpg1lF9BXyMa3MR+XbkRVeqD8JoxMP7aY0WQkiTxEucdj7Z
CaycUN1ch5twc621KXZcSMC5QaLa+cg2M+FcZM2kygeP1ONC6YuvUnMZ5sQ0
L4t5xB2Cy+BZ619yhDEMQezMcu1elPa4dlw4zUK2hatbJmL3RTmlJAL+0WzG
tVUnNswsHggTgbEKsX9l8RLHu4BYzAy9uHdqV7DQluQ6GirTpSoRTmxoTmxk
nNM0AwJhv8u3Me1YCg1HYH8nQXNhWfv94c.M0Pj.oRT6jpCl+qunObspvcF2
5fX3JESkRIJXLZOyr9o2wRYEIUy49bWwJwclbOizpcLquIQB6DgMfGMQ5+7g
kivTbwVIbwbx5KQ6bpmwZ4NVLKeQ2t5vFpAEOdyeA2wR8XV5oQxmM2Gi3EMA
2GA+yqEOzaj8ju8XVBQa39iizT0bnYyo7SQUp2nCwRBxNlRuRIuw4u9h+rvC
2bsyckkQWbWCi5W4mWxAlxWoZBYPtnSfkXIPJS2VtiAVTEcPzTXh2VjX9ufc
rTdrfkX+KlRuRMV.iPE6X3XqaBEuXt6esYxeAOZFlxikRZaRXNWLsymXaBOa
F9visH6Ha5PTma0hHuiCW4Zjekbj3.QrHHZNseJvfbpVJwDGnWwQQNphdJR0
J0s8HpqkedyNfTNHAkXtMjR0RM4FDTxKtKhetuoVuOLvzdi+KXGKkCRPIlaC
oTs.kXL.TxKtqtbF2WrX9ZqQ1pL9KvzdHkCRPMRpZoC08muVYdVD8zbvLZKA
nf5H1Hng6zhqsfUcyZ9+YOgUwWcns3cdcKjU22DgbomXzDRemxxe6fbfjuNH
MZaDe5wLLEFtSWuRfWIR1pI+xxineNfQ+Q.quOrgrlQM++0dIV7JKxl7KK+0
BRALR+CNVRucrb.vrSkzxSkxSfw1evlWaxoxux+4I+xpaRh4.FN9i.16mJcY
7+CvVdpje.fAPMW2rwH8sM6AvUaTi7wN0SKdTeBbcPZ8Nt+015iRTrp9Uf+H
8peVKfBaobeqUhOvsTQFrUhiGu5iIvxbdbQmiz2UhRd+2en+nTf5coofaYdy
bOI2SWDaDzFYJhFsXPcfyosFngKg1hFEIL1KN8TPafaTTPcRTCUCRY4NgFsD
ZK5odDsWG4YfF56vbLgKnHUmX.6i6DZ7xzMaQKzis8R9NEz7c3AFc4DfahfV
CtSnIKg1pNWJrWc1ofl+SM+7HxidzwfHvtSnoKg1hCjlLIegN.ZraKTLuULW
Nh0aTueiPac9ZPKBMCNAYti6QexCim9hEW8b3RO0nHp8zHBdYL+bFrNm1Z1z
Vhr1hPXn6Ex86HChMGVi9cv.rXOqE0jlaTTG86Z6Am1lSaIHaojexVT.Ks8B
tNExTba.t8GD4aQgHE4FQ1RA+DunPO38BiNEx1uS168nhfoVLMftQjsTtOAK
KHBD3b6Y6COFWSBf6hJzaEYKE6i8kEN.7lx5HGM691cKZ+CM11KubGYtChgD
QwETvXLsZuHsGXeSyuQ.s3816tZJm36TcxRsIe68XeOw8081VXFNFL5taEin
mj3xlZzq7X5Jzd.GC3ngIvt1pQK9R9UGj7N3X1RN14iaDW+7dWK7Gkigtubi
nC+F8sYoqvc5dPeYXtOe7.3dOr20B+Q4XQuxULazcKyE1+y3N85XrjisnjIh
tLUKmSGysZ51Lec8adbtQd3.1aUdoflFIWeKLyCcJijd8lfVthI3dxD9Kr3k
OCqds3vxWniKXFVgsTkUeIArgtR087gqUpWB77XCgiNtECvHAZiQys2B90X8
gYKl5O7Pc4a3IwF5MZK+EJRBUpNndPrhR.U9QfZ6Sfpnfqyp04SBcz2Is7Gm
IGPOORRWBnGDHo0.EeDfRkAzCBqzZfBOBPq6n6AAYZMPaOAP45DGMOnSoDG8
gytprXsUGVmFFpbX8QTyPVcXcZfoxg0GQOy4gp5ZXcZnpxg0GQUy4Au5ZXcZ
vqxg0GQay4gy5ZlJcPXmxf0GQgy4Ah5ZX8f.FkAqOhNmyCgz0v5Ag5IAVGOA
TOO1OWCpGDilDPs+HPsPe3NHnMIfp8HPsPe3NHHNIf5GZyT5l3W8gUIwRKz7
UFu+UFluxz8uxGLH14aek4w7UVt+U9fSX2+gaVmux18uxGb1te6q7q1mITsS
0vHWpsUQA4AiTo1FakrVVp3+1KYszGDWRtT167oTS7Ver05+d+A5nT1q6ZzL
w00EEYBSP+pzJ1VCvXxct22fzXPF3+p9VmwlJQzUF.2k4zVCKCS0KNOgko7F
29dKU5.1PusM1iwfaGTWYdbQRmxv7agwbiOlBpPjjriNemLLJUqv7DF19YAI
2YrAuApvJPhepzTjtJsyYYtCURKFB.CYuLqd473cwx3T8hxSXYCLhMUpyXHX
wrtfhYNI.D8uCqlzzNmkgc+.oKaXDCBIB9m4zxMwxjTMCR3jjOV2PLmjLGMa
MyuqAjNZcsy8qR6bVFCa9GfEWI69Qd.tSNllpYLdBGCaS5MbGwEnsXhHNrXb
PDIMlbUZmywjWCwJg8uCVFP+NYY45PZWsa5MmKzn8dhjD81UgrgX8qR6TNVi
0Mx55nKtVBWP18J7OUGJCNo2S32dF4j82hIam1aCcLhrR2kseUZmywjvS7tF
yoknX9w1cZeAlpuccBGKZ4lTNQ+9+nSaLsMMf3g+yuLsy4XQVF+V6P3t3XTp
9l0IbLxUTkzFV2Dfs3e1u23p0XpOtHoy4Wc25BhbiJ.K53D1WC1s6gewo5ZU
voYg9HoArtBOW6mKYxZ.ntLcRuJsy4XC00TtmhGw.3pOj6zdLIU+UBNM616I
sfsMhZXykLIhipv5T9pzN2KolKEKdtcWKQz6q9ZzxdOLLMU6MBNMo4sj1uBw
L9KJ1unAfESpiFbUZmyvPZKj5YfLhwesv2IGyR0dgfSSFeKo4qgJBE8+a2wZ
+PSqaiqR6bNVziyUljtvtqRFeqJJ6o5BNvoI4ulzXLfrMNLFkXUT2LKCtJsy
4XbLsEciXEdLPHbR3F4XiTcgF3zhGPRZLFHs3AP7yMMNFecMoeUZmyw1qasg
gLtO85as67VIjqMv.mV8.RRqwbaPi3Q3t9w1vOdZDbUZmyxLYyOL1cGDHhYF
G2j0EYhJqhmmANtieawS45tzngoicLjg3FdtAwi7ph4FG4Z6hc8dLAignVu8
eBqcc2LyIzVdvt6YCweq5GNCRUXq8yqIMWxWb242p9cme6122rHO0T2PA.wc
los4mPTRE+fnQi8t296j9oGcRUIx8EIHgDgBfFwKo51aKG7o5xA8i9vvor6N
p31N5tzcefBW1vRVLlx1FjeOvu1J6CIT5.P00WfpO7a5ttOLZzZ2DnV9nwmt
Q4hwN33W2MMU7K+9E4Fr2lF16rqslKJHZjdgPtIDVx9zJHcpbdWJs+Q0htam
1aQkC95C0hg0p3Xvb+YBaMslq8ZBh9FgUfnUG7rSSirnKvYinRA8ifw26fNX
SxOpsOLIwQT5JN9M811kVIgvN0jOP6ah6rj+cNb8BJ022l7aXLwtpFHllR7d
CcDH+xzaX5MJq.SqjkqmJKG5vl6aPvpiQdYzuOOXehnw9jeMFZT88A6sba6S
qbXVO08O2y+YB8PWsNGZ1cg1NreAzVLdX9NldixRjPrdpHed+D098CexqgG9
lDGoH1wfq7JJauYin2IDVBlVI1iNUrG5hy9sCezWXhieTjW+9+29bKxB6Umf
ouQXEXZ4DR+7zm0M29cI4S1klNCcuosoQlgk7w6RtYxSDkOYWJF04vLLcCaS
qe8oS2lZ5Li8lrOMs8veS6SVlYN1w6SvXlcQxlKDzMD2+ZEKFG5Nl5xtYDeC
SuQ4mhobyCT3zbuWi2aJ5pVtcoil3aVNpzdeauUlMbWzbW5MlBai56ubAO7s
TazwHTSWfR111GZyJFNZ6pxBJcU9P7Cio5bLY0169WuS4GyoR01tKYhvBolu
nUsVYVpJ5SxPtIYZISM0Wq0xb557ZWTI3c4+tw6ac2LDWOOpwKEF4ht52qYd
MkeLpR08mEbgY+c21q8Yp1XDOeWDrnovJMokfqk6V7ooYuevQSiqttwOAtRM
bFNuLD2GiYwLznABD89gdDxu43x0WrEUDsaPM6rKPGi6CWq2uNsxd.qgowku
e4FgJtVQvv3MTo6AWolVMRMRnnT7vycMLT9wt0dn65mqsKBe2Q7PK5mFIH8i
wUJKJX8OPFEGyOzFNXUfgeTPe0Dz2C0oeLxOY.bjmeSI8iwUJ8+7oIhfetsM
Y+xh7GygH15wU28bBycfbq+FrdixOFUsz6VmDzkohcluaMEV2vtUFXUw03bi
G3UG2mpO5G83N2ybOty0vD6PBggtyE0sXKQVuWyhUw8qop+9QuekZFKuTZ3T
UR+fRCYMi2P8wszGhRt5NqucKk2ie3H2YUsjKFJ8nqFjxZKUpa0RfMplUK8U
wybxXlZkgqWOZobjg8lHCg+e4I8iwkjS1oTyIDIoDsZTKHo1znwoQxV5b7NH
8XtisWHUi8ZnRazFKj3FzKMLp4XeaS5to9Qyjh48z1soQX1lQ6misbxQjZN8
myG6e8Q8oqFUx9lvaFoDYbjzcQj2mrWzzQ58Mm1OGa47bSpwJrr9Sg0rZsR1
2TycWlhnTqPeLF1+YqPFcrqMaOpzK119Oz94PKEejKgMlzamZpeUNowy7nlU
yJ4HRG2hJBy7iIwasSc7ufyHIsQnpSIIORVihalKYeaHaQXmDCFZLWOG8+F1
2xYjbQdgy4T.v0n3lgJ12FM2GxtqPs0nw+XryO+9VN0M89BYI7FMLMJG4NO.
BeEAHcCaNV03aVw8lIyQlT7NseN1R.MpnP0PibmQpwnbJm9lu008Jpq0jd4a
mVOkh6HvFGkVbb9Nxf9HWP.xcZyjA2iTsxOwumH0h00sXt8NFJ0GM22sqRKz
cwlTarmeV9U5QjdVQpZuYiFzhdzK5dlHyIs9PnjiG9qimUGBEJmdVpF8rTN8
rTM5YoRzypi8wjk0IavQFafurgNRgDWoKGo3mr2TUOTb82I8yQVNsrTMZYob
ZYoZzxRknk0.bSi5bhiTRx0RNn+BbXkx83UXMZGvb99P0n4C6krughuW.B5Z
+iFEIiveA6aXtPsg0DxFLmLYrFeVQoj8Mx15Zjbyt+il1Dg+KHvdXtPsg0Dx
FLmTYrFcoHVx9Vjs5RLjVbujbCuso6aO98sbgZCpINJXRox0nMM411oO9A1C
embKIhpfKpovnvplmEMYIsDbs78pNeZeCDIYwE4J3wm.Wvnj6YJ5VQBtFsXV
QLbm1+a3gOfbNvA0XONjSKJTi0OPIOXkY5lFkCu5tVFsQf8F42O99VN+nfZr
HGxoEslbtlgRdvJqO1vVyUpYhff1o+JtukySJnFKxgjZQqw5GnlGrx2KhFlN
4RHiYltB+MX+eNWROOyWIcrM.0sQVEHhmPz6VNP2l11q1NgFQOMrYwqjeK51
Roy97DwvMeJKt7iZasHsvXUh1QBLtmpNfa477sUiGTsjgAsFqUa07pe.69qE
8oIli2+ZuSf8SeMqkyw2VMNP0xoDsUiwOsZdzObr4RFGQSYCPjU5uh8sbN91
pQQSCy+pCErZ4Th1pQkcKoRzhDkj6M+JasVqpgOWUSLcu0Qz4hhhjELoeXAS
4GysDj9w3Jy.T3VZ5Nojmn2xqF857GrN6TOeBDJinB7a9Vw+8U2b+i2Fp41.
0atvJY75kvoQPJy8dy3NX6sS1Iz9oanegMbM13ywl4Rp4u8hhGfMA1T+7HIl
LPEc5uUrQqw1hTXXfaC2y3LXSY+dn0ijdhhNx.cqPiWmKHmlACpqlzslURss
46vXTCsXz.sQWa78tsIqwlbN1bK06Rtqai8VNAHwv9bDcOw685ltFa34Xi32
e.+4Xaz7eJDiIw3.oqnXuaSeaXKwCvIKLUGl.Mh1ho3D.t7BgAKlEUhNzM2j
vVOlvvstKnzlSaQaaqgFtvbVYyvuKHw+bQYD4ETyc.tsKyPbid2zlyFPWqwX
ntd84zVCzVK+GGmCsn0FoBjAZXDJamYDsxNWkAt2v3uMnsV7OpKRFbx2JvT6
Z6WK2UCFVCDwR7Ng1Zw+HsLeoEtkZWissfL21Ry8jpo2KzVK8+7tXoz6vF+l
Ratu4RFrFAH3ercQ1gVzBq76ZQdkpcWU+EIcpbW23lsnES06gFyVzLitS4tq
Uo.1xDrkeyTfobgtXat12VL4V5rEMW0qR6bVlQahFM54ngCvpwxcxxr0rrEF
8FMz62rv3YYYc0OPxw8ut0bcKia8TVeMKagsznKv3Ms6OKKaz2zNNvnRtiVb
mA2IKarNXJmmkwtz0MDy49wTamtMyXSDmnys0a3tBBZN2ObamFC277VyhosB
G8vf6AaIyrK6lB1vEV95qw4WK+5HW0+xf3Oc0zKkG9e5pkrvGOWQoxfeRLLa
UYhPTZ68i+Xht4mN8ihtKjtyw5ggw4cRqdlelGpbYPcZTcx.U9gfJ7IPUbKd
wAXb+nn7LgzpGesow54Q84ZXcZXeRAV7g.qTGXmFGnTfEdHvhkA14AFJEXaO
CXIoJoSyiTTFoSf9PXEqBqyCcTJr9PZcVDKoqf0owRJEVeH0NKBtzUv5zfKk
BqOjZmEQa5JXcZzlRg0GRqyhvOcErNMPQov5CozAJyf34Q3IEVeHcNPYVDOO
zLYv53gfJTFTmFRkLPs+LPs0KyOmYgXICRsGBokYyz7HtjAperISISdLolnB
jr5RG0rZ4JbjAUypAOJ1ZIySlSmet6gU2Fx3aUH+AIJisotdyQL0dh4IPStH
sLL116L9t.aJFptQfKD2W2MPMHl0dHnM4kDs2osHlFlqAYAmFXcidqsBLmQv
sMJ7PfHvhpPJlsNWh1CXZZayPYujH8+zhxZ8VYZTttSEbZn0i1xVJllPabL9
D.GIs8QG0Eo8.llaRTOzj.7v8Een68sg6iow45WSvoAsWidgaFllYaXzqkhV
OH2bXZWj1CXZ89FAHBCM3vf+GxsxzjbMKI3zb+S3jxzF9UtgekCFwEN+TpbQ
Zmyzbp1hc2leEE0nYZduxzzbcpH3zjJbRqR4.FAs4+CjK5A6C18bTuHsGvz.
JR9u8ABiXQ.wr6jokragb41KyT9fN169.Zfig+eqmHRaNsy4YVS1bKDcGBh4
kpp6sEo66bVtV0wIrLq4lzlSKPL2a3ga3r+86lQDCC2KR6ArLv8fx7iXig0b
mJLVuSdFlqMYbBOKZ8URNk.F5xnbu9Y2ih8LxQtJsGvyHHlqQQ.OIv+SoY7c
xynbsnhS3YtWxXR6ZcS41BCoPMZc1wPxBtHsGvyX2ZCWUaTA4CLBs1sJNiy0
dHfSxFzH.u4DmYQaT2kkylHRLO7vqR6A7LWjZ7gxtEJCMZrQ2JOSx0ZFNgmg
92aRqZcEaatiQ9YG27gNR1kn7.9k1cw+6OEbLd0ajdq1Yn4Jw9S3WQR8kzfV
+Zyl4RmhPbEOmOLrKR6A7rNtojqtDijqAEgtUclVtxaGNsCI2RZOanqXvJ4G
d5ineWbRZ9Lm1C3YQKAsuOT6byx7uiwsdNqmqzxgSy11QRSybAMa81vO6z7+
6dmU6hzdfiSMayXKFxwL4+RTuUclibkILbZZ7NRZalyChXa5BmbI5cFb+KtH
sGvy1KjD1+KyhrBP360C8jEoKbZp71SZbVGY2HUv8ezUAFcqUztHsGvzHYy+
Ws4WLctV7lu3cwzRErX5OnCCMcBtPFuOOKGVWhd+Z6fgozGiqd+A6spTOWc1
2qIl68j48mUypkqmus3Ic2mz0uOLGhrys463Cv278888pkfhLC+8hQ+cZ+br
k6sR5XMbxbcqlNWypgkruE8hp267xeauvEPkdeyo8ywVxlrRMuLSOYsuC0rZ
411Nc1SwtSWuW76yl8T9os8YI7uKPt7YO0u1J9vSilLYDU7OGv5wTm1+0wQz
UGF+WZ+bnkaGyJo2fPVNUMVMJ1LsjssHN6uO1H9o22xoXyp4wjsbpZrZLQvn
J123FMarQ7SuukSwl0pgSlTUSMlHXsR12.Y1Xi3mdeKsqFvIMFsYc6r4tZLU
y1M3pglS01h7.z2nlMNLL2Sa+mM3guW52GA6+cnQYuS6misjViTiqFZtiHpd
KinhrK+hG2AUd1aH1wMIFmOJw1n6GFCOc7ygyFQEWfVD5a9M8duGC3EXP6co
RRlzgKlPZ8EmPRd32ZTtkUHH4lxmTMylRRyoSul4tIoknSGiXX99HpfaaMIR
XQYzTrehlguS4miqb5y0Zzmma5k9qOpOc0JQeNhsYCnheXmv0BBc37Ny8Oan
Ckb5ykZz4kaNv9qOpOc0rRNORzrAuwO74wjScVolvqkaNvR0L+dobyA1k6a9
skICdie3fvlbpyJ0DdsbyAVpl42KkaNvtbeS5yF7F+z22xoaiqIRTRAdONum
R+y58HmTaSMV+v8RNOZsYCnheX4HIGwsbMVimanyR0LrfItjGgKdn1ICnhe5
8sb513ZrHO2PmkpYXASbIOBGNrYCnhe58sb51Jp7nR1.pqYXAmbZPd9ivMus
R+S9HbTMOBWTgYuO1M9gsFI46lVy3BkxFL6Zr8gp4U3PbxXb3mdaKocc0XNd
x60TMlHPkDvN+r1rw3vO89VNEa0LUYojy40ZlNoDkLnc0nXiRqp4jvnMM1XC
dqGkCHYX280er60XVR+XbgIU1TiHx8+XfDsduy6nLrQuOkAXruYH5Wrb2v33
0t1CONayZOcuSaQXCWisy6fLb2deJCb.1htpx68itaCazZrctPRo0deJCb.1
LbR+m61fFuN0AN+ggEfeeJCb.z7c3IsataCaxZrcdCgw8H88oLvbr4Zyl0c4
tMroqw14c.FgwI85j4XCvYMSt6BaId8qy217OoIcJWMJNBIJHXKrz.v8Wstw
ylx.uSaQaaqgFrvZKZxTFv+baQkvhV7x9FG2qbGwsYSYf2osFnsV9ubde5IT
X89TFXNzn1roLvcAs0h+EYQVJCSlx.yg190x2lx.2EzVK9Wvko756SYf4PSj
YSYf6BZqk9ym2pBo9XxTFHpkO2zC2OE0oPw3Eg8cs9jQGPdRmK2EcVyjlG2c
I2csJEVWlOluOkAlxEHUl0S2u.sGvx5vrdP2cwxr0rrEF85BoeeJC7rrL24p
Isxt6hk0WyxVj8nDLYJC7nrLpoyZId2EKarNxBmaitiq2mx.G.sY1NcelwlH
nImaqG6tB91TF3.nsO6ndqm2cGXKYZrbSizvqr72vTF3acJiCywxdISY.TtT
5M+oqVxJx6bEk.Ol0nFonj2gwvs6ysv0MPXbbbbdm15my.YAKUGXmGXmLfke
Hv19Dv5VK65cFPjPBGDom2Is9IMPRrtHxOWBqyC8SFvhODX4x.6AwBJCXgGB
rPcfcdvgx.11y.VkKS9z7vEkR9zGOqAxhVnNzNMBR4P6Co6YQLktDZmFTobn
8gT9rHLSWBsSiyTNz9PpeVD4oKg1ogdJGZeH8OKBF0kP673FkBsOjBHtNCjO
HjOoP6CoAhqyB4ChVSFzNdHv1pycf4wYICX6OCXIqNvNMxKYvp8PXsN6nNHT
LY.6GaFUxDZolDrESV7fEkkIPpHuf0jpxX6QwVtv7TxZsW9uYxVmy6IrQv8e
eVGbT55HyleAWgVA257n6+.ritW1Q0iHTzeSeeVG7NsEwzvb8OJ3zv6+9rN3
HFQe17K3JzNmoEM4z2m0A2FSix07lfSCv+6y5fCXDJLa9EbEZmyzb6vlLqCt
MlFmqICAm9zAuOqCNJ0kjYyufqP6bl1PmMqCtMlljqC+.mlAhuOqCNfQL5yl
eAWg1oLMtMlMqCtMlllq85.mlZiu2yMNJQxfYyufqP6blFByl0A2ESKYam3p
8ojCdB6FMa9EbEZmxyPflLqCtsyY455Cmvxr9jYcvArAPmM+BtBsyYYnNaVG
bW7LLWmI3DdVzWijbJAP+mNY9EbEZmyynwrYcvcwynbcE.3jJrRmLqCNfOvv
r4WvUncNOSfYy5f6hmw4pH+S3YXaxrN3.9PLgode9EbEZmyybQpSl0A2EOSx
U03mvyHdxrN3nbkSeeBFjmx47KSmMqCtK9klqZsOgeEoVXRCZQaLa9EbEZmy
yFsYy5f6hmY4pTZ3zFHbKo8rgthIyufqP6bm.h9X46y5f6hm0yUkxvo476Ho
oYTimM+BtBsy4YfLaVGbW7rQtZoENMYhGIsMi.a17K3JzNmmsWNKuMqCtM6Y
SVIqvoITbOowYDgyleAWg14LMllMqCtCll8jMyRHWQaC0DGbHYV5US4uCodP
C77mbMrRJ6LAvcTUen1ZAj68KfhhDdxvtWS8uCPI6atn1jyDfj6aUTQ5PtWK
oUyqkjgQ1KpET2R1fgKZ05UbFwE7mrS8u5HRgcf5VNE.sZZ+CsbJ.Z0ntoIk
rsEkKStN0+Stuk68ya07rqsbJ.Z0n3tgkruQsrcp+mbeKm5lQMMRiVRE.0n3
NyZUhbjewd9vSHLlcl.7bmPvQRkM5BoVir8N+iLdo9dmONRosAqoCwhCM2cM
4V5c9YW9yer.1FP1dmO2QHauy+.Zm1674XnI9iz67yxC+VO7rpRa.G4bpaf0
bdkxsZbMqVIZYMhy067ORBZ08NebjSCaeTCWrkaOCpY0RsXmpYXX8M2N1cEV
tIsCItLqMvcMQiXEhpuMz.eGKKkkfpkODMddWyugXRXA9Qus1SfqdIVoXQOJ
I2jN34Bs.laXhh8ZzjmaXh9qOpOc0zR12zV1IBvStukym3NUCmLm9yZFBrXt
gI5x8MCyNQ.dtPvh4FcoXuUCmLoNzZr6I2vDc49VWxNQ.dx6a4f1451TxswR
cK+c4ICWoUiNT2lvaL5JJbq5G8AJQk6eK51Roy97VzI3eWIwkCrsdLlBb2PE
MZHm38fqbCkUzJIBkXt4jJVyP6EyMmTWdOaXYmzAOn7wbSkUzpw2obyIUrl4
aKZ07zh.jcRG7j6a4zYa03+Tt4jJVy7sEsZdZQjyNoCdx8sb5rqoPrvjgMrr
0ZslsS63Afys2ZwbNv44h6ooB8C0r4hSkDj9w3RSpYqFKH0Rdxz37bx4pvCZ
SWtA5IVyLtD0bZ1zZrHQK4MS67H4bU3I21xoFslwbIlazj9qOpOc0JIXtcky
NWEdx8sbpQ0hDb0x+pMErZoVr10Cn6zI3VSssQLnCHAa6UT28LA2vbyKTTpw
5fbyKTrloS5q+XRTn1v4MNmVqkbrJv8X9T7DiUguv15BbBNuM4z.N4XUvgj9
LiUguv15BQpcdSwogVxwp.2GiGYrJ7EzVWuPsy6.N9mSxwp.O7c3GYrJ7E1V
WWOsy62MMQRNVE3guC+HiUguv15Zvocd6swMMN4XUvOGNdlwp.larJ+8279c
CI0jiUA1H3YFqBesssFZsEFa0RNVEh+1yLVEdAMbcf+sEYwNmbrJ3FOZOyXU
3Ezn0PiWjn28jiUA11uV9.iUgWPiWCMXY9ImarJvVGelwpvKnIqiZQeQ1voI
GqBNUx+cVI31mbMRmK20OG7LiUgWLLcMCSVl7r4FqBQcv8eah89+R3Eo8.qm
Z8mYrJ7hkYqYY3hrIkSNVEtOVFBOyXU3EKqu1m7EIfqzSNVEtOVFQOyXU3EK
arlksHudML4XUXtsS2mYrIhPy415M5ZxwpvtsSOxXU3Woh7pXKH2SBndkku9
wp.9slxwggUzpXrJfBeoLe+SWsjE034JJUSy1AJOJNNOvXUHMXw5.67.67.i
Ugrf87.8rBr+2FR+AQ54AFqBowpVFVOHzOOvXUHMXo5.67XA8.iUgzfsUGXm
GbnGXrJjDrqhVzUjOMObQOxXUHMZa0g1oQP5QFqBYQ6hXJcIzNMnROxXUHMZ
45P6z3L8HiUgznEpCsSC8ziLVExh1EAi5RncdbidhwpPZzVnAxyC4ySLVERi
1BsPddzZdfwpPVvpi5.673r7.iUgzfUqCrSi7xCLVERi05ri5fPw7.iUATxk
zSRMYyqjryMTTVlzRE4Eol7hNUR+AmVn5.HsI47pAt1ZcntLNRbGjia8snmA
+VZH8FkeLp3wStisWC5YxKnSazs6OiPxA3f6xBlc.NbDsy5+bbuSOy.b3KlF
lqYeAm9PBIGfCNiPxN.GNh14Lsg7LCvguXZTtteEb5SIjb.Nr+r2+mgxfhBe
UZmxzb+uelA3vWLMNWumBN8QJRN.GXWCZ1A3vQzNmogzyL.G9hoI4Z7Svo45
XxA3fCNI6.b3HZmyzH8YFfCewzzbccI3zjnL4vHvYD8+6PYPgldUZmyz39yL
.G9dGb4zdexUaVNGjRFTK6.b3HZmxyLt8HCvguNmkq0ibLKq2jrCvAGZz+cn
L3NFfWk14rLgdlA3vKdFlqQRbBOKZtV45QvQOx++LTFjQm4qR6bdlpOy.b3E
Oix0DGNgmQT1A3fisd1A3vQzNmmY8mY.N7hmw4ZfBmvyXK6.bfsdK6.b3HZm
yybQpOx.b3EOSxUL7mvyTH6.bvwFka.NLmx4tAznmY.N7heo4JB8S3WQRLlz
f1dS+uCkAykSdUZmyyhg98SL.GdwyrbE.NbZ2dN4.bXWWw+NTFvwPkqR6bdV
zdSehA3vKdVOWgCCmlcwIGfCN1f+6PYvkRYWk147rX.j+DCvgW7rQth1ENMs
kSN.GbrwYGfCGQ6bd1dgy7.CvgurmMYIyBml5xIGfCNpF+mgxPezg9UocNSy
ZOy.b36sA1atipx4BTrc5yK311FWCU02gbIsZidY5pwaRXdAEQ+HXPtS.9MD
20fAzotSohw7F9pzt+La9ONZDzCQafed.snDUn3Y2hxSui1WlK8Fs0mKdYYh
sBYhtdkvQADimL.rWuz3UncNSraavPPfiVMrR5WgRMASjeFlnZ0wD65l3qKB
hMrv8O6pz9OLFNhKfq2d3j5dt49wgj6EmedEfup+0uSY8YNXVNHWGGzMh1MI
zEp5936nSdkDRWg1obPf2rN1jXPqin+Y.IYg3CwBgxXgVT03to0gUAQ5W9k2
EWg1orPzM.Ghl4K3WiI0MNJIKDdFVnzK7Tne2L50bjDHST7xzNkExtO09eJt
b61fC6FFIYgsGhEV2oP0skmimQTngLPGy5Uocp9DkP22cW2syrh.NLLKq9jO
NmLSxE4BOH15t2dRzrnfvN7wqrr3JzNmKxtG8tjS26gVnXtejF7aHWOyxEk5
3hfr4twHnhCmKAs8gWvkncNWTrMybuJcam4dy0qHo4hOjwMLVGWLlF.jovPi
5YUAZbUZmyEcGgbmcb2kHWKtEO.WZt3CYfCMJjK5pQsnQ5fXiF9ucbUZmyEi
xbDP2VQ2aTFif8jlK9P13PZcbQh1F9sToGYCzdFnbUZmyEGRDYX1cSzY5M2f
w7ZWdHybnBcb1sGYfTe.i1dsbztLsy84qM1fXLXQl+IvteybZt3CYoCUnmyw
KvzvQ7.3QV87UkDbEZmyE8epJtS1tJHwo7U2SKEWb7LLQrPOmi4Et6fl06wL
pWT0tJsyYhDG4Li3VKZNC0O4lN7C8GhIVnyyZKRaC00Ermek8lcUZmyD4X71
wZOFUTtQO5H88Y6gXhE53hZaQXuirAZPQHbgqR6bln5lDEC.rtA6ld2Sqf9i
8aIcGD73T2cdWiedKDbZt6dCsPP5y6LhGLoW9YaMhXA6Wy6F9+r6WPEoP9zd
g7OYJj2RipyFeCyNZMXeKjH2VgX+y+g7+KOoeLtxLpd0yKisF31Lpg6HiwnE
oq3+KpbjnWk626F9dm6xbrW4tNu4pVzWoQHNvH3huSY8wVLEHoqCRWcQLfQb
kkgJC++0tBGP2jo+x5i3WFfctCYGr6wus6M.X1lGubyid.Lhi+fMOXxlWHPU
l9Kq+QCx.L.p4ToMFoOTxO.tZGLmF06ess9nDwYpeT4ORZ1GFZUHUwLAkzQk
gdJIL0rV4ltIfg0rZTtUiqY0RYsGzN+QELKl3NXj+8f6kB3Js1a05ssvoDx2
tbkO9g2CcI9cR+bjkydOcTCeLmcX0LkX.K0o+ZNOlapbr7DxnsEQlCzlFuuQ
Gze7SH4l.Hfp0vGsbqVMxrxMSNVsqEoJLYwPxn2b2Ta7usSzHECMLQrXWsq8
uz94PKYNMP0vHyI7ulQoBjalbrbaCrsdTeF3X3l8CcC9aXeKmplZlRFflT7e
QBIakru4bcczhQeFwBJbu+Wv9VtQkATyvq.xMpLfZltIPtd.+x8Ml1DLzL46
EJMDY72v9VRU.zBYIis8Tsb3Nr2GQxqEeuliXi5Ms2i3wNhDY8Hk7uS6misb
5.fZLRNWa+D9VqMoJ+wyt7voopO5lLs0fdCiV4i6FotWBKjoQK9o4RbHSHRF
19i+4zRQ0n4l1SwfgltHsHfareQn0FFRQwrM1SGWbxS4Lg1xiHUVlXytk1jK
jqSv.RMJ+x0IX90G0mtZkn7SIYa+orPlPW1neddWvHso5PTV5wo29oBQei1O
Ga4bqqD9XttACvEsZ4Tzx0nnkKQQqxcee2kODU7hzihX+G2Kxb02Bv0nJJah
2UiOqLWxtl9+2bWKM2FI2fuu+JboyaMEPi9ENtIGSxobLGRIqU6FkHK4RRdK
mTU9uG70TRwRr4PL6zCo8A4RjfpI.5AO5F3CDlUnr4aPPFIRNc90Z9RzJNly
qI5y5ebLorF4gn0rv1hlJSYJfDbdYDYdV0Z97zLny80YTeyIEQu+y4LBbFUW
eZ2Az26lU8R4p4JwoWlwDIhTGxdwZZJWKZEUWdlPEie12KJ97nIi4fgDedYj
wDafjFhVSMu0XFqpjsGVQwSb90Z9dndPmct3yKiLlXCjwbWNLZwEfUIwHa47
uqZ0NyZMedzBiIVUwoWlwDaPXL2uRHNUjDEPeZlwf637eCbAe19CiIV0fO6w
gwDaPXH2uRACiZTi0pDShp434+VwbB0pgwDqpupC80uTqc0Fx0qXJAKxXfRz
jERRzLUd9sP5qpMesbAW6p4zd7X7hFFx4KUxX.SjRIRJZ1BuNb9igL3NCiCd
ttUsLUPEoRpwRV90b+BREHKVdu7KFd8n5sJfFTY.wU+GM9.VMetX3w3Pi84h
gGiowc6x30hbJJQ.0R322rFgIAnUiEQLvLpZoLCBmrGsCh4BqEQSvfkBP+nK
l6.HOxVwbxZQZDf5plEDWZtC.IHaEuEWKFfnQLuMUWJtCAVGaEykVK5bno7j
T88H2gfQish4xqE2LzV6Y66QtCAvEaDysZ.sneisYYuMIVTJ0.CMkYCIcPfm
nCsCRwsVXl.8aF5C+2yal2AyrgokrfvHh.pCc.3fnCsig2BqE7GZSUaUyt3s
tfzvlwaxZgjAS3Ow64+9.7VWnSXy3s3ZAJATMiTI5Su0EPC1LdKsV3K.k7Gs
m2aTwFgRqqSSQEi0Hso2RVp.1yawRFyNhPqbkVBs8s+dH3KXir+lWKXEfxsS
2Kpf9xghFlTKuwXjobsXtNyKj1CDl3A.qfMRlUVKzD.f4rtWvFmXY1AflfMR
lUWKRDXVHmp66m+zJyN.RDrQxLcs.O.FFPkrurQND.ArUA0tZ.AnM4oUeoib
nF2eCXNmkYnrMUH2RV9wOH42s7GOrWJOhAIu2S9iFygry9FLbwAcRR9Nt1zf
VMeUyvrGgZCDFpIK5tB.BdAfMcWvJn+gnNdrJfI00tQdLGmOUOkZrcyuJ1WG
NxyDIRxLARo2VOyG5jFkIDsPAAuaQuKQcozFySQKJXKVXtTkhE1X6P9vnTv1
yDiIJksD9y8ocPBsfuFLbFgVRmvX50mPKMUaokqoTBMjBuTZ6KzTKXFLhuYK
qXja0t4i7lIzDe8b2LBs1NhjucZXvKXlEJTRDymcNIKk1tBspkmfjsbrIKSP
6CZ+M1TgVzWGuMiPCm+cTbsSqxVvdVFinBgDKK8mAi9kPaegVPPgmVzpfdSg
SgscmVxW6lwyb6I7DU8YSqFRSJZBCamoEXOJOtkRaegljav9XEFSLYFG4MUn
k80qWyHz3TmFa4PBBL+tUkxn9ViV9PokRaegVTaWdR1d9LxnEmBaoPyYqcrz
lA5.GqizJeaLmzvPLQyjtTZ6JyrjelBBvcZKAHbcDbZS2n4qUGNrLyLSOAua
tjYwnYZuXQVxkLQ16mWJs8kYIfqMUQpljkPHJaZ.GAeMZvLxLVmppOu.YLGW
TFyIMK1fL.XwkRaeYVNOUEyNlZOMioIXdS8BH9Jy+YjYh40xYfsVRHSTq1vw
8dwxLNN6SZeIVwB1v7WHXFgkiZoV1RIVzWwnyydQZEmQ0ZjMglLKjYMh8m4x
RosuLSYysYqXFXLsmzsUlk7UJ37rWPWwYPsnMDrnAvjRqDBgnkw6Rosel.jY
MSZ0HQVqUptoN.x9pCad168K6Lj1BgJ6TEyTcxRPzhDMrTZ6KxXzlHIT5gEL
VpSkMMPihupflm89DSNinsDnIy9rYPWCnU3Udgj1WhYgjTBQJitS13p315xr
5qBj4YukxjyPyJh8pwnTQmQZobknvRosuLKZY7BTBKlqb09WbScYp9p+Wd1a
+L5LzrRLMkBIE2taNDsnozkRaeYVqLbznfZqQ.L+JaZ3rNK+1YDZBCu791nA
7ZNXwOUx1ibsHOWJs8EZEB.abRSENiHiBalWS4ThXDtNB7WNLfUtV9tIgiLC
IBxjktFFehX5sXYvAijlybfGMkTJSALSdan7QNQH16nRwLYI61tk+NjtVFy4
oROqO6hkuTFatShhurHnry5I6aYp3gqn2g4fi5V1VzxO9aYyBZwCx0UFx0dQ
k7obwRmxEK5SKlGyp4xzI8NfBbbaYWvxuEaYcgariAn.HWfV4vVL5TtXCnUW
LGOgueZ0ExGxXRiAyOIenUIMFfHjbgej0wzNYjKDcbXKV7TtXNscU2FSmKY4
2.Sm4SWTvj35H9ms3mUptu8EhaWng8RgLY+OtXmt8b29DNDdxg8kgrS0WanS
io8oIesF9fVqjONSGypE7sZiwLsuFwcXqF4e0V+SzAek.0bOQihEXuz+59Dc
2R.aCdj12tiwrwm0So5xW6pRCxwJm8r63UQvZWMmlPBia0NNuEFSbk9J7UZL
38.4qJeoAY5uUrvG6RrMG4yNcjRZDinDsDmppJp4WFmzY.kqPLl3oPjiVdEs
Aeff92nCoCGUXI1SDO437S9oTklH6Krnc3MQ0mKeJKUQ6sQ0S0gxMh0NpZiy
tTaQyHN5RBMpQKyujhwgGliNXbXZICh7DIENA5P5vGDLN4Mh+8ya1Zf1SLYp
INmrc..TYbwakSxVRhbskrKuQIYBM1qTMO.BWTbmEe+vaQM86m2hALrEBZBM
4Szd+uq1RFy9dbyrOjijEvElSkIMfByAvH4DGSVXUBw.lpPOV1gzgOjtbp1B
AuVI2i2LijzTU3JlxBJgdDRNk71wzaxw1R9hoDzKuRJJUA8qb.CXSIKVFwZ0
15gQ7CtHjZWRG9r5xmdSb5cqKukp35OSDSVDvALaSjSIucL8VX9Fr+acAjwg
VjQ+4ZwDBtPBgDp0qDNZMaycvrszkzgOCv7o2Bj30L497lY+XhC.2XnpPYtg
OgmNd6nkDYLRtTbozTrVv7PvhrpTKIyKcpViS.uXhLq0nxRn1kzweVctTbrk
FsKMWl6wbRXxhWNQVtzEQrm33tjtQL2Q0bRVco4TdxrrmxFCw4BG.rllDMzQ
ysOoi+Pdcp4B9BMoOyAjUYOM2Ig47c.WC5DSH8jtZdNDz76PZxgIYWvpuAWM
fqCBSe2ANN1Um8t5mDV8cGwz3Ty9V8sYz0PjuNONQGAJ5SSJfzorHLUKxtQL
SMLga3kjZ61dIbMuQ.U3VD.oBg6moRTC9h6P6p0qN3rzXtTAGqjLn0gW75z9
.Wb4m+7uc8CO9Lwsk3hOc4+791tn5O190ata2uxse8gq+sadgdlZuzkOb0+3
lmt9pm9xCsuWW7UbYssO681Fx69xMO+Dgwc1Z93S+6au948Huvq2c4m18Q+C
+4e5O9mZeXvJ+xke41mdK6e68W8ut9m+6e7Wu59a2885u8AZhIfDg.OiLG4.
x9LuhVVcJZwjnsuCS+J7Rs2MVZTiMXS+eEzubusy8l+yyRVN7Mu02rTL719l
e7JUOc8We5MeorHipU6aAZv3Hhqn8Q185QljXjQiT81+JW+y2X5ueceFDYrR
wZCkzdl+rHuBU6oBVv4oz3uLptS.MWMbK7cr2KRYaSwO8wau9o6u6C+0Ku6w
O7WrE8Ke5hWo8xqt556dKuj9VgI.CKuuzq659wm2T+f8mtsEnoueVU+ISOey
KtHdYy4U2d4iOhuyO94K2si+hOd+Wu3G9u+v+CUfUR5B
-----------end_max5_patcher-----------
</code></pre>
```
</details>
  
## Sound Design
I knew I wanted the piece to give a lot of control over to the environment, which I ended up using three plants to represent. I thought of the instrument as having 4 voices: one for each plant, and one for me. The sounds that made up each of these voices were samples of either nature noises for the plants, such as a storm or wind, or human and “household” for my voice, like footsteps or a microwave.
  
The moisture levels of the plants control the speed of playback for their tracks, while the gyroscope reads the position of the hand to trigger changes in the sound.
  
## Performance
This video is documentation of performance with the instrument.
<div style="padding:56.25% 0 0 0;position:relative;"><iframe src="https://player.vimeo.com/video/559100666?badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" frameborder="0" allow="autoplay; fullscreen; picture-in-picture" allowfullscreen style="position:absolute;top:0;left:0;width:100%;height:100%;" title="Room Tone Demo"></iframe></div><script src="https://player.vimeo.com/api/player.js"></script>
 
