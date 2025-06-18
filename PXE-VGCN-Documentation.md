# Setting Up A PXE Virtual Galaxy Compute Node (VGCN)

This documentation outlines the automated image building process for setting up a **PXE-bootable Virtual Galaxy Compute Node (VGCN)**, used within the Galaxy Europe infrastructure. The system uses Jenkins, Packer, Ansible, and iPXE to provision compute nodes that boot over the network using images built via this process.


**Reference**: [Virtual Galaxy Compute Nodes](https://github.com/usegalaxy-eu/vgcn)
```
Galaxy Europe runs jobs on compute nodes belonging to the bwCloud/de.NBI-cloud. These compute nodes are known as "Virtual Galaxy Compute Nodes" (VGCN).

Virtual Galaxy Compute Nodes boot from VGCN images, which are built off this repository and made available as GitHub releases and on this static site.  
```
## Scope
This documentation describes:
- The workflow for building VGCN images using Jenkins and associated scripts
- The tools and repositories involved in the image building process
- The role of the PXE infrastructure in booting the VGCN
- The artifacts generated during the process and their purposes
## Jenkins
Jenkins acts as the orchestrator for the VGCN image build pipeline. It executes two primary scripts:

- `vgcn-pxe-pipeline`: This pipeline builds the actual VGCN image using Packer and Ansible.
  
- `pxe-config-playbook`: This builds the `config.tgz` archive, which includes runtime configuration files and SSH keys for the VGCN.

These scripts interact with internal and external repositories to gather configurations, fetch dependencies, and trigger provisioning.
    * vgcn-pxe-pipeline --> builds the images
    * pxe-config-playbook --> builds the config.tgz archive
## Repositories Used
- **`jenkins-scripts`** (private): Contains Jenkins pipeline scripts that control the image build and configuration packaging process.
- **[`vgcn`](https://github.com/usegalaxy-eu/vgcn)**: Contains the Packer configuration and Ansible provisioning roles used to build the image.
- **[`pxe-config-playbook`](https://github.com/usegalaxy-eu/pxe-config-playbook)**: Ansible playbook that generates the `config.tgz` archive, containing system-level configuration for the VGCN.
## Output Artifacts

- `config.tgz`: A compressed archive containing configuration files, including `authorized_keys`, hostname setup, and other cloud-init-style data.
- `VGCN image`: Built by Packer using a Kickstart file and provisioned via Ansible. This is the disk image that the PXE boot process ultimately mounts.

## Infrastructure Components

### `dnbd01` — PXE Head Node

This server hosts several essential services required to boot and run a VGCN:

#### TFTP Server
- Manually configured
- Provides:
  - PXE bootloader (e.g., iPXE binary)
  - iPXE script for boot instructions
  - The initial handoff from iPXE to PXE ROM happens here

#### HTTP Server
- Serves:
  - The Linux kernel and `initramfs` needed for iPXE
  - `slx.config`: [Explain what this file does — *currently missing*]
  - `config.tgz`: Contains SSH keys (`authorized_keys`) and node configuration files

#### DNBD (Distributed Network Block Device)
- Hosts the final built image, which the VGCN mounts during PXE boot


## Workflow of the image building process
~~~mermaid
sequenceDiagram
	participant Jenkins
	participant Webserver
	create participant VGCN
	Jenkins->>VGCN: fetches
	create participant jenkins-scripts
	Jenkins->>jenkins-scripts: fetches and executes
	activate jenkins-scripts
	jenkins-scripts->>VGCN: sets parameters and starts build.py
	activate VGCN
	create participant packer
	VGCN->>packer: starts packer
	activate packer
	create participant image
	packer->>image: builds image with kickstart
	activate image
	create participant Dracut-role
	packer->>Dracut-role: pulls
	Dracut-role-->>packer: ansible role
	packer->>image: provisions with ansible
	image-->>packer: finishes provisioning
	deactivate image
	packer-->>vgcn: finishes build
	deactivate packer
	VGCN->>Webserver: copies image

	deactivate VGCN
	deactivate jenkins-scripts
~~~

## Glossary

- **PXE (Preboot Execution Environment)**: Allows a machine to boot over the network.
- **initramfs**: Temporary root filesystem loaded into memory during the early boot process.
- **iPXE**: Advanced network boot firmware capable of booting via HTTP, TFTP, iSCSI, etc.
- **TFTP**: Lightweight file transfer protocol used for bootloaders.
- **Packer**: Tool for automating the creation of machine images.
- **Ansible**: Configuration management tool used to provision the image.
- **DNBD**: Distributed Network Block Device; provides the final image storage.



----

## 1. Configure a Jenkins Job To Run the pxe-config-playbook
1. 2 Jenkins Skripte: vgcn-pxe-pipeline, pxe-config-playbook
2. Welche Repos (jenkins-scripts (privat), vgcn (packer + ansible), pxe-config-playbook)
3. Wann kommen sie ins Spiel?
4. Welche Artefakte (Output vom Image build) gibt es und wo werden sie hinkopiert?
5. Wer benutzt die Artefakte? -> TFTP/HTTP/DNBD Server
   1. TFTP -> benutzt gar nichts (iPXE bin, wird nicht über Jenkins gebaut, sondern manuell)
   2. HTTP -> stellt config.tgz, slx.config, initram kernel zur Verfügung
   3. DNBD -> stellt image zur Verfügung
   4. alle 3 liegen auf dnbd01


Based on this graph:  


~~~mermaid
sequenceDiagram
	participant Jenkins
	participant Webserver
	create participant VGCN
	Jenkins->>VGCN: fetches
	create participant jenkins-scripts
	Jenkins->>jenkins-scripts: fetches and executes
	activate jenkins-scripts
	jenkins-scripts->>VGCN: sets parameters and starts build.py
	activate VGCN
	create participant packer
	VGCN->>packer: starts packer
	activate packer
	create participant image
	packer->>image: builds image with kickstart
	activate image
	create participant Dracut-role
	packer->>Dracut-role: pulls
	Dracut-role-->>packer: ansible role
	packer->>image: provisions with ansible
	image-->>packer: finishes provisioning
	deactivate image
	packer-->>vgcn: finishes build
	deactivate packer
	VGCN->>Webserver: copies image

	deactivate VGCN
	deactivate jenkins-scripts
~~~
  
## 2. Run The Jenkins Job
1. Jenkins fetches and executes the [pxe-config-playbook](https://github.com/usegalaxy-eu/pxe-config-playbook).
It creates the directory structure and the config files for the ```PXE VGCN```.
2. Additionally, it adds an ```authorized_keys``` file to be later able to ssh into the compute node.
3. Finally, the server structure is archived in a ```config.tgz``` archive, which looks like this:  
   
   ```
    ├── etc
    │   ├── auto.data
    │   ├── auto.master.d
    │   │   └── data.autofs
    │   ├── auto.usrlocal
    │   ├── condor
    │   │   └── config.d
    │   │       └── 99-cloud-init.conf
    │   └── telegraf
    │       └── telegraf.d
    │           └── output.conf
    └── home
        └── centos
            └── .ssh
                └── authorized_keys
   ```
4. The Jenkins build number is appended to the ```config.tgz``` suffix and is copied to the ```dnbd01``` server directory ```/images/netboot/http/netboot/config.tgz.r{build_number}``` .
## 3. Boot the Virtual Galaxy Compute Node (VGCN)
1. During the boot process the ```vgcn```  pulls ```config.tgz.r{build_number}``` from ```dnbd01```.  
   
2. The ```config.tgz.r{build_number}``` server structure is extracted to the ```vgcn``` root directory.
