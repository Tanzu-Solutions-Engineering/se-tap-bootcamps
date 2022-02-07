# TAP Up and Running bootcamp

This repository contains material for the SE TAP Up and Running bootcamp.

## Prerequisites for attending the bootcamp
- Be very comfortable in working with Kubernetes and knows how Kubernetes work (if that's not the case, it's recommended to do a CKAD certification first)
- Completed the **TAP Overview** workshop at: http://via.vmware.com/tap-demo
- Watched James Urquart's whiteboard pitch video (current version: https://via.vmw.com/EdB3)
- An account for https://network.tanzu.vmware.com
- Has access to a public could and has provisioned:
  - A Kubernetes cluster (GKE, AKS, EKS)
    - Commands to provision a GKE, AKS, or EKS cluster can be found here: https://github.com/tsalm-pivotal/tap-install#provision-a-kubernetes-cluster
    - (WIP) Terraform scripts to bootstrap the infra for TAP: https://github.com/Tanzu-Solutions-Engineering/se-tap-bootcamp-automation
  - A Docker V2 API compatibel container registry (GCR, ACR, Harbor, DockerHub, ..)
    - It’s not recommended to use AWS ECR as container registry due to its lack of support for long-lived credentials! 
- The privileges for a (sub-)domain to create wildcard DNS entries for e.g. *.cnrs.example.com, *.learningcenter.example.com, *.example.com
  - **Will be provided by the instructor, if the SE doesn't have a private domain!**
