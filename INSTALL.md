Presentation:

- Prerequisites:
  - Plaform: VM, RPI4/5, PC...
  - Linux OS: raspbian, debian, fedora, RHEL (or variants)... Needs to have native packages for kubernetes (and crio); Already installed (basic installation)
  - A hard-drive big enough for your data, (i recommend anything bigger than 100gb, e.g.: 2Tb, but for testing you can go lower, e.g.: 20gb, maybe even lower)  
  - internet connection
  - a simple DNS server in your network (not really needed but highly recommended)
  - node must have a vaild hostname and resolvable via dns (not entirely needed but highly recommended)
  - configure LVM in your operating system for dynamic volume provisioning (using topolvm)
  
For this example i choose to use a raspberry pi 5 as the hardware and Raspberry pi OS (debian) as operating system.

Steps:
- [First we need to install a basic kubernetes](install-bare-minimum-kubernetes.md)
- [Setup the storage provider](setup-topolvm.md)
- [Add an ingress (haproxy)](setup-ingress.md)
- [Install nextcloud](setup-nextcloud.md)
