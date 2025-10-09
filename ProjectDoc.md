# AGL Vehicle to Cloud Communications Project 
- [AGL Vehicle to Cloud Communications Project](#agl-vehicle-to-cloud-communications-project)
  - [Project Definition](#project-definition)
  - [Scope](#scope)
  - [General Architecture](#general-architecture)
  - [Use Cases](#use-cases)
  - [Project Timelines](#project-timelines)

## Project Definition
The intention of this project is to produce a prescriptive specification for the production, consumption and orchestration of messages between connected vehicle devices and the cloud using MQTT and Protocol Buffers, inclusive of recommendations for vehicle identity, security and system architecture.  

## Scope
Documentation of recommended basic practices for MQTT V5 based vehicle to cloud communication patterns. 
* Message format and orchestration for the most common vehicle telemetry and command based use patterns. [See Documentation Here](https://github.com/jamesenki/AGLV2C/blob/3dcf4fcee2a7bcef8e7f2af6473dfb90b810aa05/src/main/doc/%20v2c.md)
* Gradle Build configuration for building both documentation and Java (and other?) stubs from protocol buffer files.
* Example message implementations for reference. 
* A Stretch Goal of message testing and simulation
* Stretch goal of an exposed API through AGL for generating a salted hash identity for vehicle from the VIN number, using same keys as the used in creating the operational certificate. 

## General Architecture 
[Insert Drawing]


## Use Cases
* Vehicle and Vehicle Device Provisioning
* MQTT Communication Lifecycle Events, Monitoring and Best Practices
* Client Initiation and Connection
* General Messages
* Basic Telemetry
* Remote Commands
* OTA Orchestration and Content Downloads
* Remote Diagnostics
* Application Defined Messages [Key Value Pairs]
* Simulation and Testing

## Project Timelines

* 01/20/23 - Project Description and Details Updated in Confluence
* 01/31/23 -  Initial Repository in AGL hosted Repository
* 03/01/23 - Completed draft of main contents to a "ready for invitation to collaborate" state
* 03/01/23 - Ready for collaboration beyond working group

Calls for Help
* Need reviews of content in Repo as produced and help adjusting the diagrams as needed
* Help producing reference implementation in C, RUST, PYTHON etc. 
* Help creating a quality testing and simulation framework. 
* Help creating security tests to validate securiy of vehicle identity and vehicle to cloud communications scrips
* Need to create the vehicle identity API for AGL and understand the mechanism for translating back to VIN in the cloud system.
* Help aligning the message objects to VSS