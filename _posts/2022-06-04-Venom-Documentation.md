---
layout: default
title:  "Venom Documentation and Usage"
date:   2022-06-02 12:47:20 -0500
permalink: /venom/
categories: jekyll update
--- 

## Introduction 
Nowadays enterprise systems and networks contain critical information.
Threat actors are coming up with new sophisticated attacks and techniques
that are very hard to detect and prevent. Organization are conducting Red
Team engagements to simulate such attacks, in order to implement
countermeasures. 

The objective of Venom is to provide Red Team operators with an
integrated framework. Including a Command and Control server which
will be handling all agents compromised in a target network. The operator
is provided with an interactive web interface that presents multiple
features. The features include a listener creator, an implant generator, a web
shell to control agents and a dashboard to manage different types of agents
(Linux - Windows). The communication channel between the server and
agents is designed to be as stealthy as possible.


## Run Venom
After Installing required packages in the Installation Guide, run Venom as by specifying a port as follows: 
```bash
python3 venom.py --port 1337
```
By Default, the web application will bind to localhost. 

Venom Generates it's default account password with each runtime. 

Logging through the login portal to access the Dashboard. 

![image](https://user-images.githubusercontent.com/54769522/172024407-48e538a4-f668-4e4a-97d7-d2387a559418.png)

## Creating a new listener 
Navigate through the side toolbar to listeners, and specify the required port to spawn the HTTP listener component. 

![image](https://user-images.githubusercontent.com/54769522/172024416-9d9ced70-5361-43ae-94ef-d7e7eb3468c1.png) 

## Generating an implant
Navigate through the side toolbar to implants, and specify the implant type, and listener ID to bind the implant program to, once run on victim device, it will callback to the binded listener, use the **Copy** functionality to easily copy the program into your clipboard.

![image](https://user-images.githubusercontent.com/54769522/172024436-286aa502-b368-4c40-813e-167d34857f1e.png)

## Interacting with agents 
Navigate through the side toolbar to venom, and specify the live agent ID, by pressing on Venom you can easily use the integrated webshell to assign tasks and wait for agent results to be written back, have a good pwn time! 

![image](https://user-images.githubusercontent.com/54769522/172024478-ae6e9c4b-6dc2-4941-ab1a-6517664a85c4.png)

