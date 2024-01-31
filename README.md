# SOLECTRUS Hosting Guide

There are different ways to install **SOLECTRUS**:

## A: You have a SENEC.Home?

If you have a SENEC.Home battery system, installing is simple. The toolchain is quiet stable and used by many users. Please choose one of the following guides:

1. [Local installation on Synology NAS](/guide/synology)

   You need a Synology NAS (Kernel v4+) with Docker installed (tested with Synology DS220+ with 10GB RAM)

2. [Local installation on Raspberry Pi](/guide/raspberry-pi)

   You need a Raspberry Pi running a 64bit OS with Docker installed (tested with Raspberry Pi 4 Model B Rev 1.4 with 8GB RAM)

3. [Distributed installation on Raspberry Pi + remote server](/guide/external-server)

   You need a Raspberry Pi at home **and** a remote server somewhere on the internet (tested with Hetzner Cloud)

   Server costs at Hetzner: €4,51 per month

## B: You have a PV system, but no SENEC?

If you do not have a SENEC battery system, you may still be able to use **SOLECTRUS**. There is a brand new [MQQT-collector](https://github.com/solectrus/mqtt-collector), so SOLECTRUS can be used with any PV device that supports MQTT. Please note that the MQTT-collector is in an experimental stage and I would appreciate your feedback.

There are two guides available, both targeting **experienced** users having a deeper understanding of MQTT:

1. [Installation for use with ioBroker](/guide/mqtt-iobroker/)

   You need:

   - Linux server (64bit, Kernel v4+) with Docker installed (Raspberry Pi, Synology NAS, virtual machine, etc.)
   - Running instance of [ioBroker](https://www.iobroker.net/), which already includes a MQTT broker

2. [Installation for use with evcc](/guide/mqtt-evcc/)

   You need:

   - Linux server (64bit, Kernel v4+) with Docker installed (Raspberry Pi, Synology NAS, virtual machine, etc.)
   - Running instance of [evcc](https://evcc.io/)
   - MQTT broker
