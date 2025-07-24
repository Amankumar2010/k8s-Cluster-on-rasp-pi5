# k8s-Cluster-on-rasp-pi5
Tutorial on how to setup multi node kubernetes cluster on raspberry pi 5 from scratch 

# 🧵 On-Prem Kubernetes Cluster with Raspberry Pi

## 📦 Hardware
- 1 x Raspberry Pi 5 (Master) – 4GB min
- 1 x Raspberry Pi 5 (Worker) – 4GB min
- MicroSD cards/ Nvme drives , power supply, network switch optional. 

## 📶 Network Setup
- Static IPs
- Hostnames: `k8s-master`, `k8s-worker`
- SSH access, update & upgrade

Lets get started -- 

1. Prepare Both Raspberry Pis

a. Set Unique Hostnames
On each device, set clear hostnames:
<code>
sudo hostnamectl set-hostname k8s-master     # On the desktop/master node
sudo hostnamectl set-hostname k8s-worker     # On the server/worker node 
</code>



ASCII Diagram
              +-----------------------------+
              |     Your Home Network       |
              |     (192.168.1.0/24)        |
              +-----------------------------+
                           |
                           |
                    +-------------+
                    | Router / WiFi|
                    +-------------+
                           |
            +-----------------------------+
            |                             |
  +-------------------+        +-------------------+
  | Raspberry Pi 4 #1 |        | Raspberry Pi 4 #2 |
  |  (Master Node)    |        |  (Worker Node)    |
  |  192.168.1.10     |        |  192.168.1.11     |
  +-------------------+        +-------------------+
            |                             |
            +-------- Kubernetes Cluster --------+
                           |  
               +-----------------------------+
               |   Deployed Nginx Service    |
               |   Exposed via NodePort or   |
               |   Nginx Reverse Proxy       |
               +-----------------------------+
