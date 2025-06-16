# Setting Up A PXE Virtual Galaxy Compute Node (VGCN) Using Jenkins

Ref. to [Virtual Galaxy Compute Nodes](https://github.com/usegalaxy-eu/vgcn)
```
Galaxy Europe runs jobs on compute nodes belonging to the bwCloud/de.NBI-cloud. These compute nodes are known as "Virtual Galaxy Compute Nodes" (VGCN).

Virtual Galaxy Compute Nodes boot from VGCN images, which are built off this repository and made available as GitHub releases and on this static site.  
```

Ref. to [pxe-config-playbook](https://github.com/usegalaxy-eu/pxe-config-playbook)

## 1. Configure a Jenkins Job To Run the pxe-config-playbook
1. Project env vars (e.g. ssh cred.)
2. git checkout
3. ansible play 
  
## 2. Running The Jenkins Job
1. Jenkins fetches and executes the [pxe-config-playbook](https://github.com/usegalaxy-eu/pxe-config-playbook).
It creates the directory structure and the config files for the ```PXE compute node```.
2. Additionally, it adds an ```authorized_keys``` file to be later able to ssh into the compute node.
3. Finally, the complete server structure is archived in a ```config.tgz``` archive and looks like  
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
4. The ```config.tgz``` tarball is receiving the Jenkins build number and copied to the ```dnbd01``` server directory ```/images/netboot/http/netboot/config.tgz.r{build_number}``` .
## 3. Booting the Virtual Galaxy Compute Node (VGCN)
1. During the boot process the ```vgcn```  pulls ```config.tgz.r{build_number}``` from ```dnbd01```.
2. The ```config.tgz.r{build_number}``` server structure is extracted to the ```vgcn``` root directory.
