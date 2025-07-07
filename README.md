# Image Mode Demo
Image Mode Demo scripts
Draft container files, index.html, and config files to get an Image Mode demo story going.

## Image Mode Commands
Build the demo base image for RHEL 9.5
```
cd ~/imagemode/imagemodedemo/base-rhel95

sudo podman build -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:latest -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.5 -f Containerfile.rhel95
```

