# Using ACM and AAP for Bare-Metal Cluster Installs
This repo contains an example/demo of how to leverage ACM and AAP together to do bare-metal cluster installs. This is useful when the bare-metal provisioning capabilities of ACM need some additional support in the form of task-oriented automation, handled by AAP.

This is also useful when ACM cannot properly leverage the BMC/redfish capabilities of the target devices, or the redfish implimentation is not complete.

This repo assumes you have a cluster with ACM and AAP already installed, and that you have Provisioning and an AgentServiceConfig set up.

## Components
This repo contains setup and cluster build sections.