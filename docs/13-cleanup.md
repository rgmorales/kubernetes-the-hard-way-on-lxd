# Cleaning Up

In this lab you will delete all the compute resources created during this tutorial. This single command will delete all containers, networks and profiles in your server. 

## Deleting all resources

Please be sure you want to delete everything berofe executing this script...

```
{
lxc stop --all 
lxc delete controller-0 controller-1 controller-2 worker-0 worker-1 worker-2 haproxy
lxc profile delete kube-profile
lxc network delete kube0
lxc network delete kube1
}
```

You can also go to the folder where you executed all the commands and delete all the configuration files, and downloaded files for the labs.

Have fun!
