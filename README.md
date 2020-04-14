# Abstract

IBM Cloud Pak for Data is a fully-integrated data and AI platform that modernizes how businesses collect, organize and analyze data and infuse AI throughout their organizations.
This guide will provide most of the information one needs to stand up a working cloud pak for data 3.0 environment from scratch. 

  
### Table of Contents

- [Part 1 : Prerequisites](#prerequisites)
  - [Hardware Requirements](#hardware)
    - [Sizing the cluster](#obtaining-software)
  - [Software Requirements](#software)
	  - [Obtaining the software](#obtaining-software)
      -[Evaluation Licenses]
  - [Installation Checklist](#checklist)
	- [Automated Prereq Validation](#checklist)
  
- [Part 2:  Provisioning and Installation](#part-2-resource-provisioning)
  - [Bare Metal](#bare-metal)
  - [VMware vSphere](#vm-provisioning)
  - [Amazon AWS](#aws-provisioning)
  - [Microsoft Azure](#azure-provisioning)
  - [KVM](#kvm-provisioning)
  
- [Part 3: Serviceability ](#serviceability)
  - [System Health Check](#sys-health)
	- [Core OS](#coreOS-health)
	- [RHEL Operating System](#rhel-health)
  - [Diagnostics Collection](#diagnostics)
		- [Operating System Diagnostics](#os-diag)
		-  [CPD Diagnostics](#cpd-diag)
		
- [Part 4: Day 2 activities ](#day-two)
  - [Monitoring](#monitoring)
	- [Kibana](#kibana)
	- [Alert Manager](#alert)
	- [Openshift Console](#ocpconsole)

- [Part 5: Debugging ](#day-two)
	- [Installation](#installation)
  - [Performance](#perf)
    - [metric server](#metrics)
		- [Pod Scheduler and Load balancing](#scheduler)
		- [Disk Performance](#disk)
		- [Network Performance](#network)
   - [Common CLIs](#perf)
     - [Pod](#pods)
		 - [Nodes](#nodes)
		 - [Registry](#registry)
		 - [Router](#router)
      
