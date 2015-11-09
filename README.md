[![Build Status](https://travis-ci.org/martinhansdk/Continuous-Meter-Reader.png)](https://travis-ci.org/martinhansdk/Continuous-Meter-Reader)

# Continuous Meter Reader

A little gadget that sits on your water or electricity meter and monitors it continuously.
The collected reading are presented on a mobile web page so that you can practice saving
ressources by answering questions like "How much water do I use when I take a shower?".

Currently Sensus water meters are supported. They look like this:

<img alt="Sensus water meter" src="pcb/Optical water meter reader/sensus_meter.jpg" width="50%">

I have designed a PCB which fits on top of the water meter which uses [GP2S700HCP reflective photointerrupter](http://www.sharp-world.com/products/device/lineup/data/pdf/datasheet/gp2s700hcp_e.pdf) sensors to read the turning of the half shiny half red disc. The makes one turn per liter. The PCB has six optical sensors, so it can sense quantities around 1/12 of a liter. The reflective part of the disc is not exactly a half circle, so the exact 12 quantities measured are different from each other. An automatic calibration method is provided which determines how much each quantity is. You can print a [PDF rendering of the PCB layout](pcb/Optical water meter reader/optical meter reader pcb board.pdf), make 5mm holes for the steering pins plus a small hole for the middle of the sensor circle in your printout and see if it will fit on your meter.

<img alt="Photo of the PCB" src="pcb/Optical water meter reader/optical meter reader pcb photo.png" width="50%">

The ambition is that the same or a similar PCB will be able to read the turning weel on an old style electricity meter. The PCB was designed in [Eagle pcb](http://www.cadsoftusa.com/) and the schematics are available [here](pcb/Optical water meter reader/optical meter reader.pdf).

It uses an [Arduino Nano](https://www.arduino.cc/en/Main/ArduinoBoardNano) as the CPU module and [NRF24L01+ Wireless Transceiver Module](http://www.icstation.com/1pcs-nrf24l0124ghz-wireless-transceiver-module-arduino-p-1388.html) for wireless connectivity. Another Arduino Nano with a second transceiver module allows a server PC to receive the measurements from the meters. Alternatively communication can happen directly over a USB cable.

The server application runs on a PC written in [Go](https://golang.org/) and stores the measurements in a [Postgres](http://www.postgresql.org) database. The server application exposes a REST API, a [socket.io](http://socket.io/) service for real time updates and also serves a web application written in [React](http://facebook.github.io/react/).

The communication protocol between the embedded code and the server is based on [Google's protocol buffers](https://developers.google.com/protocol-buffers/) with the Arduino implementation generated by [Nanopb](http://koti.kapsi.fi/jpa/nanopb/). The protocol definition file is [MeterReader.proto](MeterReader.proto).

## Build instructions

All build instructions assume Ubuntu 14.04 or later.

### Install prerequisites

    sudo apt-get install make clang protobuf-compiler arduino postgresql-9.4 git
    wget https://godeb.s3.amazonaws.com/godeb-amd64.tar.gz
    tar xzf godeb-amd64.tar.gz
    ./godeb install

### Getting the code

The project uses git submodules, so you have to use a recursive clone to get everything:

    git clone --recursive https://github.com/martinhansdk/Continuous-Meter-Reader

### Uploading the embedded code to the Arduinos

Attach the Arduino for the utility meter to a USB port and run

    cd src/Continuous-Meter-Reader
    make upload
    
Attach the Arduino for the server receiver station and run

    cd src/RadioStation
    make upload
    
### Building the server and node configuration tool

    cd go
    export GOPATH=$PWD
    go get -d
    go build MeterServer.go 
    go build ConfigNode.go 
    go build SampleSender.go 

### Configuring the Arduino for a meter

Each node has an id which is an integer between 1 and 127. We need to set this id and also choose the wireless or serial (USB) protocol. Configuration can only happen over USB.

To configure an Arduino that is attached to /dev/ttyUSB0 run something like the following:

    go/ConfigNode -serial=/dev/ttyUSB0 -wirelessproto -id=1
    

### Calibrating a meter sensor

The calibration procedure is as follows:

1. Attach the PCB to the water meter
2. Connect the Arduino to the PC using a USB cable
3. Turn on a tap so that a constant flow is running. Make sure this is the only place water is consumed in the house while the calibration is running
4. Run the command `go/ConfigNode -serial=/dev/ttyUSB0 -calibrate`
5. The command will exit when calibration is complete

