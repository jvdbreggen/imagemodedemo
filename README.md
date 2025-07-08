# Image Mode Demo

Image Mode Demo scripts
Draft container files, index.html, and config files to get an Image Mode demo story going.

## Image Mode Commands

### Build the demo base image for RHEL 9.5

Commands to build, test and deploy a RHEL 9.5 base image and create the qcow2 VM file.

```bash
cd ~/imagemode/imagemodedemo/base-rhel95

sudo podman build -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:latest -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.5 -f Containerfile.rhel95

podman run -it --name rhel-bootc-qcow -p 8080:80 quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.5

podman push quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.5

sudo podman run \
--rm \
-it \
--privileged \
--pull=newer \
--security-opt label=type:unconfined_t \
-v $(pwd)/output:/output \
-v /var/lib/containers/storage:/var/lib/containers/storage \
registry.redhat.io/rhel9/bootc-image-builder:9.5 \
--type qcow2 --tls-verify=false \
quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.5

sudo mv output/qcow2/disk.qcow2 /var/lib/libvirt/images/rhel95-demo-base.qcow2

```

### Build the demo base image for RHEL 9.6

Commands to build, test and deploy a RHEL 9.6 base image and create the qcow2 VM file.

```bash
cd ~/imagemode/imagemodedemo/base-rhel96

sudo podman build -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:latest -t quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.6 -f Containerfile.rhel95

podman run -it --name rhel-bootc-qcow -p 8080:80 quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.6

podman push quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.6
podman push quay.io/jvdbreggen/demo-bootc/rhel-demo-image:latest

sudo podman run \
--rm \
-it \
--privileged \
--pull=newer \
--security-opt label=type:unconfined_t \
-v $(pwd)/output:/output \
-v /var/lib/containers/storage:/var/lib/containers/storage \
registry.redhat.io/rhel9/bootc-image-builder:9.6 \
--type qcow2 --tls-verify=false \
quay.io/jvdbreggen/demo-bootc/rhel-demo-image:rhel9.6

sudo mv output/qcow2/disk.qcow2 /var/lib/libvirt/images/rhel96-demo-base.qcow2

```
