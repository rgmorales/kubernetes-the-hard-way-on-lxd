# Using Multipass to create private cloud environment on Mac.

After you install Multipass on your Mac/Windows
see instructions here https://multipass.run/

# Launching VM using multipass

multipass launch --name k8s-hardway-18 --mem 5G --disk 20G

multipass exec k8s-hardway-18 /bin/bash

You will get the following prompt

ubuntu@k8s-hardway-18:~\$

Next: [prerequisites local cloud setup](01-prerequisites.md)
