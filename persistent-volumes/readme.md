# Peristent Volumes in Kubernetes

In this example we will create a persistent Volume, followed by a persistent volume
claim, then create an nginx pod that will mount the volume we created

As this is my home environment, I have an Ubuntu server sharing and NFS filesystm over running on ZFS

* Create the Persistent volume
```
