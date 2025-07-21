# Image Mode Demo

Image Mode Demo scripts
Draft container files, index.html, and config files to get an Image Mode workshop story going.

## The workflow

```mermaid
graph TD;
    corp-rhel:rhel9.6-->homepage:v1;
    homepage:v1-->homepage:v2;
    homepage-create-->homepage:v2;
    homepage:v2-->homepage:v3;
    corp-rhel:rhel10.0-->homepage:v3;
    homepage:v3-->homepage:v4;
    homepage-update-->homepage:v4
    homepage:v4-->homepage:v5
```

The following diagram will be updated as I work through the workflow.

```mermaid
sequenceDiagram
   participant BaseRHEL96 as base-rhel96
   participant BaseRHEL100 as base-rhel100 
   participant HomepageV1 as homepagev1 
   participant HomepageV2 as homepagev2 
   participant HomepageV3 as homepagev3 
   participant HomepageV4 as homepagev4 
   participant HomepageV5 as homepagev5 
   participant HomepageCreate as homepage-create 
   participant HomepageUpdate as homepage-update

%% Begin sequence
Note over BaseRHEL96,HomepageV1: Initial deployment of homepage Virtual Machine from RHEL 9.6 base image
BaseRHEL96->>HomepageV1: Deploy homepage v1

Note over HomepageV1,HomepageV2: Upgrade homepage to v2
HomepageV1->>HomepageV2: Upgrade to v2

Note over HomepageCreate,HomepageV2: New homepage creation triggers v2 deployment
HomepageCreate->>HomepageV2: Create homepage v2

Note over HomepageV2,HomepageV3: Upgrade homepage to v3
HomepageV2->>HomepageV3: Upgrade to v3

Note over CorpRHEL100,HomepageV3: Deploy homepage v3 from RHEL 10.0 base image
CorpRHEL100->>HomepageV3: Deploy homepage v3

Note over HomepageV3,HomepageV4: Upgrade homepage to v4
HomepageV3->>HomepageV4: Upgrade to v4

Note over HomepageUpdate,HomepageV4: Update homepage triggers v4 deployment
HomepageUpdate->>HomepageV4: Update homepage v4

Note over HomepageV4,HomepageV5: Upgrade homepage to v5
HomepageV4->>HomepageV5: Upgrade to v5

```

## Set the environment
Setup of the terminal for building Image Mode images that we are going to push to the registry.
In this workshop we will be pushing to Red Hat Quay.

We recommend that you set two variables in the terminal you are using for the logins to the Red Hat Registry and Quay.io.

Using Quay we recommend that when you push the images to Quay that you make the repositories *public* by selecting the repository and using the Actions to set *Make Public*

```bash
QUAY_USER="your quay.io username not the email address"
REDHAT_USER="your Red Hat username, full email address may no longer work"
podman login -u $REDHAT_USER quay.io -p $REDHAT_PASSWORD && podman login -u $REDHAT_USER registry.redhat.io -p $REDHAT_PASSWORD
sudo mkdir -p /run/containers/0
sudo cp /run/user/1000/containers/auth.json /run/containers/0/auth.json #The user number 1000 may be different for your user
```

## Build the demo base image for RHEL
The first steps we will build our base (golden) image that we are going to use within the workshop. We will start with RHEL 9.6 and during the workshop update to RHEL 10.0.

We will name our base (golden) image `base-rhel:rhel9.6` and also tag it as our latest rhel base image as `base-rhel:latest`.
We will then deploy a new virtual machine named `homepage` as this will be our new homepage http server.

Commands to build the RHEL 9.6 base image.

Change to the folder where you have cloned this repo
```bash
cd $HOME/imagemodedemo/base-rhel96
```

```bash
sudo podman build -t quay.io/$QUAY_USER/base-rhel:latest -t quay.io/$QUAY_USER/base-rhel:rhel9.6 -f Containerfile.rhel95
```

If we want to test our image we can run it in a container.

You can log in with user `bootc-user` and password `redhat` and run `curl localhost` to test if the httpd service is running and you can see the base image welcome page. You can stop and exit the container with `sudo halt`.

```bash
podman run -it --rm --name base-rhel-96 -p 8080:80 quay.io/$QUAY_USER/base-rhel:rhel9.6
```

Next we are going to push our images to the Quay repository.

```bash
podman push quay.io/$QUAY_USER/base-rhel:rhel9.6
podman push quay.io/$QUAY_USER/base-rhel:latest
```

Now we are ready to create the virtual machine disk image that we are going to import into our new VM.

In some cases the podman command is unable to initially pull the image from the registry and returns an error that you have to pull the image from the registry before building the disk. Use 

```bash
sudo podman pull quay.io/$QUAY_USER/base-rhel:rhel9.6
```

```bash
sudo podman run \ 
--rm \
-it \
--privileged \
--pull=newer \
--security-opt label=type:unconfined_t \
-v $(pwd)/config.toml:/config.toml:ro \
-v $(pwd):/output \
-v /var/lib/containers/storage:/var/lib/containers/storage registry.redhat.io/rhel9/bootc-image-builder:9.6 \
--type qcow2 \
--tls-verify=false \
quay.io/$QUAY_USER/base-rhel:rhel9.6
```

We will copy the new disk image to the libvirt images pool.
> You can move the disk image if you don't plan to use it for another VM using the mv command.

```bash
sudo cp ./qcow2/disk.qcow2 /var/lib/libvirt/images/homepage.qcow2
```

Create the VM from the copied virtual machine image qcow2 file.
We will give it 4GB of RAM and set the boot option to UEFI.

```bash
sudo virt-install \
  --connect qemu:///system \
  --name homepage \
  --import \
  --boot uefi \
  --memory 4096 \
  --graphics none \
  --osinfo rhel9-unknown \
  --noautoconsole \
  --noreboot \
  --disk /var/lib/libvirt/images/homepage.qcow2
```

Start the VM.

```bash
sudo virsh start homepage
```

and login via ssh. You can use the following command that will get the IP address from virsh and log you in.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

You can run a `curl localhost` to check if the httpd service with our base image homepage is working. Exit the VM with `exit`, `logout` or Ctrl-d.

Finally for this section run the bootc status command to view the booted image registry source and the RHEL version.

```bash
sudo bootc status
```

>Booted image: quay.io/jvdbreggen/base-rhel:rhel9.6 \
>Digest: sha256:a48811e05........... \
>Version: 9.6 (2025-07-21 13:10:35.887718188 UTC)

## Next steps, now in commands and then build with text and fix the sequence diagram based on the flow

```bash

```