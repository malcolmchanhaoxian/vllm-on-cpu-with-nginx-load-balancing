# vllm-on-cpu-with-nginx-load-balancing-with-Google-Compute-Engine

<p align="center">
<img src = "https://github.com/user-attachments/assets/54e7cbc8-8b2d-4a05-9895-a509f2ed13b8" width = "600">
</p>

## Introduction

In this project, we will explore a few concepts while leveraging existing use cases shared previously. Here, we will invoke a vllm model serving engine on CPU but with a few modifications. First, let's review some of the common shortcomings when inferencing directly on a cloud-based VM cpu cores.
Refer to this [repo](https://github.com/malcolmchanhaoxian/VLLM-on-Intel-Extension-for-Pytorch-) for a general understanding of the vllm structure and how we performed testing on the performance.

## Hypothesis
#### Problem Statement
1. When running Intel Pytorch Extension on cpu, the innate nature of the framework will defer to inferencing on the physical cores rather than the logical cores. This presents a challenge since most cloud VMs are made up of vcpus and it is comprised of half physical cores and half logical cores. This problem is magnified when user provisions a large vcpu count and multiple sockets are introduced.
2. Inferencing on CPU can get tricky when the VM is NUMA-aware and performance can degrade when the inferencing operation is crossing multiple NUMA nodes

#### Solution
There are a few methods to tackle this - Intel introduced a method of distributed inferencing to spread the model shards across multiple sockets but this method does not directly address the concerns of concurrency and batch-inferencing. Here, we introduced multiple steps - 
1. Configure a cpu core-aware VM directly on Google Cloud via advanced configuration to expand physical cores
2. Introduce the thread binding feature in vllm
3. Perform load balancing with nginx across multiple NUMA nodes

#### Configuration
We are utilising a VM on Google Cloud. The specific SKU is c4-highmem-192 which features 192 vcpus. This SKU is selected to display the nature of VMs on cloud and the availability of multi-socket, multi-NUMA node.
Model utilised is microsoft/Phi-3.5-mini-instruct.

#### Step-by-step Instructions
1. Google Compute Engine VM Creation on gcloud
```
# Use the following codes. This is just a snippet of the critical details.

gcloud compute instances create <VM-NAME> \
    --project=malcolmchanapj-intel \
    --zone=us-central1-a \
    --machine-type=c4-highmem-192 \
    --threads-per-core=1 \
    --visible-core-count=96 
```

## Findings
To document and assess the performance of our baseline VM and the efficacy of the solution, we will be using Locust load-testing method. Locust will allow us to set a specific concurrent user count and the latency performance of the VM itself.

For this load testing, we set the maximum users count at 500. We can see that the Core-aware VM performed much better reaching a response time of 21,923 ms at the onset of reacihng 500 concurrent users. The baseline VM had a response time of 38,524 ms. The improved VM is also able to handle concurrency much better, consistently floating around 15 total requests/sec whereas the baseline VM were not able to break past 10 total requests/sec at any point in time.

Results at 500 users
| VM Configration | Total Requests per Sec | Response Time (ms) |
|-----------------|------------------------|--------------------|
| **CPU Core-aware VM** | 12.3                   | 21,923             |
| **Baseline VM** |  6.3                   | 38,523             |
   
#### Baseline VM
<p align="center">
<img src = "https://github.com/user-attachments/assets/62556964-4871-4f71-9207-e2fbc14c7e03" width = "900">
</p>

#### CPU Core-aware VM
<p align="center">
<img src = "https://github.com/user-attachments/assets/eb99b12a-0a4f-43af-8ac2-0b31387fb7cf" width = "900">
</p>
