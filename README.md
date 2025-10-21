# Image Mode Demo

Image Mode Demo scripts, Containerfiles (Dockerfiles), webpages and workflows to demonstrate and understand how to workflow a "day in the life" Linux system administrator.
Draft container files, index.html, and config files to get an Image Mode workshop story going.

The core of this demonstration is how to create a "golden image" or a base image that can be reused in all RHEL deployments and become the standard operating environment. This is based on a tree, or inheritance, structure where the services, and more specifically the sytemd services are build on top of the base image. These services can be a web server, database server, application servers and more. From these services images we will deploy the virtual machines, bare metal servers, edge devices and more.

Why, do we start here and not with the base image? The services images will have the correct file and configuration structures in the base. If we deploy the VMs from the base image and then switch or upgrade to the services image, the VM will may have missing services file strucutres and config files that needs to be in the /var and /etc directories but aren't written during a bootc switch or upgrade command.

Once the services VM has been deployed we can then create the application images and pull them from the registry to the VM to make the VM funcational. In this demo we will do this for a homepage that we will upgrade during the process. We will start by deploying a very simple web page for the httpd service image. We will upgrade the homepage with content describing Image Mode for RHEL 9. At the same time we will use the base RHEL 9 image to create a database image and deploy it to a VM.

Then we will create a new RHEL 10 base image which we will use to upgrade the http and database services images, and when we then deploy a new web page for RHEL 10 we will pull the new RHEL 10 base as part of the upgrade. Similar we will do a more simple upgrade for the database.

## The Build

The first build will be on RHEL 9.6 where we build a base `demolab-rhel:9.6` image. We then build a second pair of images for our specific services, the httpd and mariadb services. Now we can deploy these services images to Virtual Machines. We then create a new homepage image that has more details on Image Mode, and use the `bootc switch` command to update our VM to the latest home page.

```mermaid
classDiagram
direction TB
    class demolab-rhel9.6 {
        FROM rhel9/rhel-bootc(latest)
    }

    class httpd9.6 {
        httpd service
        demolab-rhel(9.6)
    }

    class database9.6 {
        mariadb service
        demolab-rhel(9.6)
    }

    class homepage9.6 {
        rhel 9 Image mode homepage
        httpd(service)
        demolab-rhel(9.6)
    }

    class VMhomepage {
        httpd service
        demolab-rhel(9.6)
    }

    class VMdatabase {
        mariadb service
        demolab-rhel(9.6)
    }
    class VMhomepageImageMode {
        rhel 9 Image mode homepage
        httpd(service)
        demolab-rhel(9.6)
    }

    note for VMhomepageImageMode "bootc switch"

    demolab-rhel9.6 ..|> httpd9.6
    demolab-rhel9.6 ..|> database9.6
    httpd9.6 ..|> homepage9.6
    httpd9.6 --|> VMhomepage
    VMhomepage ..|> VMhomepageImageMode
    homepage9.6 --|> VMhomepageImageMode
    database9.6 --|> VMdatabase
```

Next we create a new RHEL 10.0 base image, from which we will upgrade our Virtual Machines. We need to go through the same steps, create an updated pair of services images for the httpd and mariadb services. We can upgrade our database service using `bootc upgrade` but before we upgrade our homepage Virtual Machine we also want to update the content to RHEL 10. We update our html source and build a new homepage image and then `bootc upgrade` our virtual machine.

```mermaid
classDiagram
direction TB
    class VMdatabase9.6 {
        mariadb service
        demolab-rhel(9.6)
    }

    class VMhomepage9.6 {
        rhel 9 Image mode homepage
        httpd(service)
        demolab-rhel(9.6)
    }

    class demolab-rhel10.0 {
        FROM rhel10/rhel-bootc(latest)
    }

    class httpd10.0 {
        httpd service
        demolab-rhel(10.0)
    }

    class database10.0 {
        mariadb service
        demolab-rhel(10.0)
    }

    class homepage10.0 {
        rhel 10 Image mode homepage
        httpd(service)
        demolab-rhel(10.0)
    }

    class VMhomepage10.0 {
        rhel 10 Image mode homepage
        httpd(service)
        demolab-rhel(10.0)
    }

    class VMdatabase10.0 {
        mariadb service
        demolab-rhel(10.0)
    }

    note for VMhomepage10.0 "bootc upgrade"
    note for VMdatabase10.0 "bootc upgrade"
  
    demolab-rhel10.0 ..|> httpd10.0
    httpd10.0 ..|> homepage10.0
    VMhomepage9.6 ..|> VMhomepage10.0
    homepage10.0 --> VMhomepage10.0
    demolab-rhel10.0 ..|> database10.0
    VMdatabase9.6 ..|> VMdatabase10.0
    database10.0 --|> VMdatabase10.0
```

## The workflow

The overall plan of the workflow is to create a base RHEL "golden image" that we will call `demolab-rhel` and will base all our Virtual Machine (VM) KVM deployments from this image.
We will be deploying an httpd server as we can visually see the updates we are doing. This is based on our `demolab-rhel` image and we will add the httpd service and a homepage that we will upgrade in the process, and with the upgrades also upgrade our RHEL release.

The diagram below shows the various flows that can be used during this demo.

There is additional optional parts that is described in the next section. First, there is a flow for minor release upgrades that can be incorporated into the overall workflow. Second, we use the `demolab-rhel` base image to deploy a `demolab-database` server and maintain it the same way as our `demolab-homepage` server.
As a future aspect I want to add an Ansible playbook to use Ansible automation to upgrade the servers.

The flow is as follow:

1. Create a RHEL 9.6 base image and add a user that is part of the wheel group and push that image to the registry as our 9.6 and latest images.
2. Create our application images and VMs.
    1. Create a httpd server image based on our RHEL 9.6 base image adding the Apache httpd service.
    2. Create a MariaDB server image based on our RHEL 9.6 base image
3. Deploy the application images as a virtual machine servers.
    1. Pull and convert the httpd:latest image to our new Homepage virtual machine server.
    2. Pull and convert the mariadb:latest image to our new Database virtual machine server.
    3. Create qcow2 disk VM files from the images in the registry.
    4. Copy the disk images to our KVM pool and create new Virtual machines.
    5. Start the virtual machines and log into the VMs.
        1. Open the URL to the homepage VM in a browser.
4. Create the new homepage image and switch the VM to the homepage image.
    1. Build the new homepage image and tag it as version 1 and latest.
    2. Push the homepage image to the registry as version 1 and latest.
    3. In the homepage VM we deployed in step 3 switch to the new homepage image in the registry.
    4. Reboot the VM
    5. Refresh the homepage in the browser. This should be broken and we should receive a 404.
5. Rollback to get our old homepage back up and running.
    1. In the homepage VM issue the rollback command.
    2. Reboot the VM and after it booted make sure that the old homepage is running.
6. Fix the error in the homepage container file and push the updated image.
    1. Fix the FROM registry in the Container file.
    2. Build a new version of the homepage image and push it as a new version and the latest version.
    3. In the Homepage VM run the bootc switch command again to load the new homepage.
    4. Reboot the VM and this time the new homepage should disply in the browser.
7. Optional: Upgrade the Database server. This shows how different application servers are updated when the latest base RHEL image is updated.
    1. Using the same MariaDB Container file, create a new database image version and push it to a new version and latest.
    2. Upgrade the Database Virtual Machine and reboot.
8. Upgrade the base RHEL image to RHEL 10.
    1. Build a new RHEL 10 OS image tagging it as the RHEL 10 and latest images.
    2. Push the RHEL 10 and latest images to the registry.
9. Build a new version of the httpd service image.
    1. Build a new version of the httpd service image and tag it as the next version and latest. This will automatically use the latest RHEL image and upgrade the httpd service to RHEL 10.
    2. Push the new httpd version to the registry tagging it as the next version and the latest version.
9. Upgrade the homepage from the RHEL 9 welcome page to the new RHEL 10 homepage. As we do the upgrade of the homepage we will also pull in the RHEL 10.0 latest base image as the latest tag of the RHEL image is pointing to the RHEL 10 image. Push the upgrades to our registry as a new homepage version and the latest tag.
    1. Build a new version of the homepage using the homepage upgrade container file that contains the new RHEL 10 web page.
    2. Push the new homepage image to the registry using the next version number and latest as tags.
10. Upgrade the Homepage VM to the latest RHEL version (RHEL 10) and reboot.
    1. Use the bootc upgrade command in the Homepage VM to pull the new layers from the registry and reboot.
    2. Check that the OS release is RHEL 10 and that the homepage is updated in a browser.
11. Optional: Upgrade the Database server. This shows how different application servers are updated to a new RHEL release when the latest base RHEL image is upgraded.
    1. Using the same MariaDB Container file, create a new database image version and push it to a new version and latest.
    2. Upgrade the Database Virtual Machine and reboot.

```mermaid
graph TD;

vm1homepage[Deploy Homepage VM v1];
vm21homepage[Homepage VM v2.1];
vm3homepage[Homepage VM v3];
vm4homepage[Homepage VM v4];

container_rhel96_1[demolab-rhel:9.6]-.->push_rhel96_1@{ shape: notch-rect, label: "Push RHEL 9.6 demolab image as demolab-rhel:9.6" };
push_rhel96_1-.->push_rhel96_latest1@{ shape: notch-rect, label: "Push RHEL 9.6 demolab image as demolab-rhel:latest" };

push_rhel96_latest1-->demolab_httpd_1[demolab-httpd:latest]
demolab_httpd_1-.->push_httpd_1@{ shape: notch-rect, label: "Push demolab-httpd:rhel9" };
push_httpd_1-.->push_httpd_latest1@{ shape: notch-rect, label: "Push demolab-httpd:latest" };

push_httpd_latest1-->convert_httpd_to_homepageVM[Convert demolab-httpd to homepage qcow2 file];
convert_httpd_to_homepageVM-->vm1homepage;

vm1homepage-->container_v1_homepage[Build homepage v1 image];
container_v1_homepage-.->push_v1_homepage@{ shape: notch-rect, label: "Push homepage:rhel9" };

push_v1_homepage-.->push_homepage-latest1@{ shape: notch-rect, label: "Push homepage:latest" };
push_homepage-latest1-->vm1homepage_switch[Switch VM to use homepage:latest];
vm1homepage_switch-->vm2homepage[Homepage VM v2];

vm2homepage-->rollback1[Rollback to homepage v1]-->vm1homepage;

vm1homepage-->fix_homepage1[Fix homepage Containerfile];
fix_homepage1-->container_v2_homepage[Build homepage v2 image];
container_v2_homepage-.->push_v2_homepage@{ shape: notch-rect, label: "Push homepage:rhel9-fix" };

push_v2_homepage-.->push_homepage_v2_latest@{ shape: notch-rect, label: "Push homepage:latest" };
push_homepage_v2_latest-->vm2homepage_switch[Switch VM to use homepage:latest];
vm2homepage_switch-->vm3homepage[Homepage VM v3];

vm3homepage-->container_rhel10[Build the RHEL 10 demolab container];
container_rhel10-.->push_rhel10@{ shape: notch-rect, label: "Push RHEL 10 demolab image" };
push_rhel10-.->push_rhel10_latest@{ shape: notch-rect, label: "Push RHEL 10 demolab image as latest tag" };
push_rhel10_latest-->container_v3_homepage[Build a new container, RHEL 10 and update the index.html to RHEL 10];
container_v3_homepage-.->push_v3_homepage@{ shape: notch-rect, label: "Push homepage v3" };
push_v3_homepage-.->push_v3_homepage_latest@{ shape: notch-rect, label: "Push homepage v3 to latest tag" };
push_v3_homepage_latest-->vm3homepage_upgrade[Upgrade VM to RHEL 10 and the homepage for RHEL 10];
vm3homepage_upgrade-->vm4homepage[Homepage VM v4];
```

### Optional additions to the workflow

This optional part to the workflow allows you to start with an early release of RHEL 9.6 and go through a minor release upgrade and CVE updates.

1. Start

## Building the demo

We include the httpd service from the start in the base image.
An alternative is to create a vanilla base image with only our login user in the base image and then do the steps to install httpd service.

> What we need to do if we install httpd in a next step is after the VM is upgraded to add the httpd log directories in the VM. See the Notes at the end of this document.

### Set the environment

Setup of the terminal for building Image Mode images that we are going to push to the registry.
In this workshop we will be pushing to Red Hat Quay.

We recommend that you set two variables in the terminal you are using for the logins to the Red Hat Registry and Quay.io.

Using Quay we recommend that when you push the images to Quay that you make the repositories *public* by selecting the repository and using the Actions to set *Make Public*
Replace `$QUAY_PASSWORD` and `REDHAT_PASSWORD` with your passowrds. If you decide to use these variables, we recommend you hash encrypt the passwords in the variables.

```bash
QUAY_USER="your quay.io username not the email address"
REDHAT_USER="your Red Hat username, full email address may no longer work"
podman login -u $QUAY_USER quay.io -p $QUAY_PASSWORD && podman login -u $REDHAT_USER registry.redhat.io -p $REDHAT_PASSWORD
sudo mkdir -p /run/containers/0
sudo cp /run/user/1000/containers/auth.json /run/containers/0/auth.json #The user number 1000 may be different for your user
```

### Build the demo base image for RHEL

The first steps we will build our base (golden) image that we are going to use within the workshop. We will start with RHEL 9.6 and during the workshop update to RHEL 10.0.

This sequence diagram show the steps that we are going to take.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_rhel_96 as Containerfile<br/>demolab-rhel:9.6
    participant registry as Registry

    %% Base RHEL Creation workflow
    containerfile_rhel_96->>registry: Push base RHEL 9.6<br/>as demolab-rhel:9.6 image to the registry
    containerfile_rhel_96->>registry: Push base RHEL 9.6<br/>as demolab-rhel:latest image to the registry
```

We will name our base (golden) image `demolab-rhel:9.6` and also tag it as our latest rhel base image as `demolab-rhel:latest`.

1. Use podman to build our corporate or demolab base RHEL "golden image". Change to the folder where you have cloned this repo and use `podman build` to build the image from the `Containerfile`.

```bash
cd $HOME/imagemodedemo/demolab-rhel9.6
```

```bash
podman build -t quay.io/$QUAY_USER/demolab-rhel:latest -t quay.io/$QUAY_USER/demolab-rhel:9.6 -f Containerfile
```

2. If we want to test our image we can run it in a container. You can log in with user `bootc-user` and password `redhat` and run `curl localhost` to test if the httpd service is running and you can see the base image welcome page. You can stop and exit the container with `sudo halt`. We are going to run our container in the next step to check that the httpd service is running and that we can see our homepage before deploying it to a VM.

```bash
podman run -it --rm --name demolab-rhel-96 -p 8080:80 quay.io/$QUAY_USER/demolab-rhel:9.6
```

3. Push the demolab base rhel image to our registry.

```bash
podman push quay.io/$QUAY_USER/demolab-rhel:latest && podman push quay.io/$QUAY_USER/demolab-rhel:9.6
```

> [!NOTE]
>In the optional steps we base the initial image on an older release of RHEL 9.6 so that we can demonstrate the upgrade process to the latest release and how to create base images based on a tested timestamp.
> In the optional section we create our base images for RHEL 9.6 on a specific version, for example `rhel:9.6-1747275992` and push this version to the registry as `demolab-rhel:9.6-1747275992` to reflect the version.

### Deploying the Homepage Virtual Machine

The following sequence diagram shows the steps that we will take to deploy our Homepage VM from the base image. We will name the VM as `homepage` and start to build

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_httpd_1 as Containerfile<br/>demolab-httpd:rhel9.6
    participant containerfile_homepage_1 as Containerfile<br/>demolab-homepage:rhel9.6
    participant registry as Registry
    participant homepage_vm_1 as homepage VM

    %% httpd VM Creation workflow
    registry-->>containerfile_httpd_1: Build httpd service image Containerfile<br/>from demolab-rhel:latest
    containerfile_httpd_1->>registry: Push httpd service v1<br/>as demolab-httpd:rhel9.6 to the registry
    containerfile_httpd_1->>registry: Push httpd service v1<br/>as demolab-httpd:latest to the registry
    registry-->>homepage_vm_1: Convert demolab-httpd:latest to Virtual Machine<br/> disk image and deploy VM homepage
```

We need to create an image for our httpd service based on the RHEL 9 base image we created in the previous step.
We will name our httpd service image `demolab-httpd:rhel9.6` and also tag it as our latest rhel base image as `demolab-httpd:latest`.

1. Use podman to build httpd service image. Change to the folder where you have cloned this repo and use `podman build` to build the image from the `Containerfile`.

```bash
cd $HOME/imagemodedemo/httpd-service
```

```bash
podman build -t quay.io/$QUAY_USER/demolab-httpd:latest -t quay.io/$QUAY_USER/demolab-httpd:rhel9.6 -f Containerfile
```

2. Push the demolab httpd service image to our registry.

```bash
podman push quay.io/$QUAY_USER/demolab-httpd:latest && podman push quay.io/$QUAY_USER/demolab-httpd:rhel9.6
```

3. If we want to test our image we can run it in a container.
```bash
podman run -it --rm --name demolab-httpd-96 -p 8080:80 quay.io/$QUAY_USER/demolab-httpd:rhel9.6
```

4. You can log in with user `bootc-user` and password `redhat` and run `curl localhost` to test if the httpd service is running and you can see the base image welcome page. You can stop and exit the container with `sudo halt`. You can test the homepage in a browser on the local machine by using the URL `http


Now we are ready to create the virtual machine disk image that we are going to import into our new VM.

In some cases the podman command is unable to initially pull the image from the registry and returns an error that you have to pull the image from the registry before building the disk. Use a pull command to syncronise the local images.

1. Since we need to run podman as root to build the virtual machine qcow2 image file, we need to pull the image as root.

```bash
sudo podman pull quay.io/$QUAY_USER/demolab-httpd:latest
```

2. We need to use podman to run the Image Mode virtual machine disk builder to pull the image from the registry and create the virtual machine disk file.

```bash
sudo podman run \
--rm \
-it \
--privileged \
--pull=newer \
--security-opt label=type:unconfined_t \
-v $(pwd)/config.toml:/config.toml:ro \
-v $(pwd):/output \
-v /var/lib/containers/storage:/var/lib/containers/storage registry.redhat.io/rhel9/bootc-image-builder:latest \
--type qcow2 \
--tls-verify=false \
quay.io/$QUAY_USER/demolab-httpd:latest
```

3. We will copy the new disk image to the libvirt images pool.

> You can move the disk image if you don't plan to use it for another VM using the mv command.

```bash
sudo cp ./qcow2/disk.qcow2 /var/lib/libvirt/images/homepage.qcow2
```

4. Create the VM from the copied virtual machine image qcow2 file. We will give it 4GB of RAM and set the boot option to UEFI.

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

5. Start the VM.

```bash
sudo virsh start homepage
```

6. Login via ssh. You can use the following command that will get the IP address from virsh and log you in. 

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

7. You can run a `curl localhost` to check if the httpd service with our base image homepage is working. Exit the VM with `exit`, `logout` or Ctrl-d.

8. Since we are going to refer to the quay.io registry, let us add $QUAY_USER to our .bashrc file.

```bash
sed -i '/unset rc[^\n]*/,$!b;//{x;//p;g};//!H;$!d;x;iQUAY_USER="your quay.io username not the email address"' .bashrc
```

9. and reload the .bashrc file to bring QUAY_USER into the variables.

```bash
source .bashrc
```

10. Finally for this section run the bootc status command to view the booted image registry source and the RHEL version.

```bash
sudo bootc status
```

>Booted image: quay.io/$QUAY_USER/demolab-rhel:9.6 \
    Digest: sha256:a48811e05........... \
    Version: 9.6 (2025-07-21 13:10:35.887718188 UTC)

Our virtual machine based on Image Mode is now running and we are ready to make updates to the web page.

#### Update the Homepage VM to our Image Mode web page

The next steps we will update the web page in our `homepage` VM from the basic RHEL webpage that we created to an more updated web page showing the advantages of using Image Mode.

The following sequence diagram shows the steps that we will take to deploy our Homepage VM from the base image. We will name the VM as `homepage` and start to build

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_homepage_1 as Containerfile<br/>demolab-homepage:rhel9
    participant registry as Registry
    participant homepage_vm_1 as homepage VM

    %% New Homepage workflow
    registry-->>containerfile_homepage_1: Build homepage image Containerfile<br/>from demolab-rhel:latest
    containerfile_homepage_1->>registry: Push homepage image v1<br/>as demolab-homepage:rhel9 to the registry
    containerfile_homepage_1->>registry: Push homepage image v1<br/>as demolab-homepage:latest to the registry
    registry-->>homepage_vm_1: Switch the homepage Virtual Machine to the demolab-homepage:latest <br/>image to update the web page
```

On our image builder server we will build a new Image Mode for RHEL 9 homepage image that we will deploy to the VM.

1. Change directory to the new web page Container file and the *RHEL 9 Image Mode* web page at `homepage-rhel9`. You can open the `index.html` file in the `html` directory to see the updates to the homepage.

```bash
cd ../homepage-create
```

2. Build the new homepage images from the `Containerfile`.

```bash
podman build -t quay.io/$QUAY_USER/demolab-homepage:rhel9 -t quay.io/$QUAY_USER/demolab-homepage:latest -f Containerfile
```

3. Push the image to the registry using the `demolab-homepage:rhel9` and `demopage-homepage:latest` tags.

```bash
podman push quay.io/$QUAY_USER/demolab-homepage:latest && podman push quay.io/$QUAY_USER/demolab-homepage:rhel9
```

4. Switch to the Homepage virtual machine and login to the `homepage` VM using ssh.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

5. We are now going to use the `bootc switch` command to switch the virtual machine to the homepage image in the registry.

> NOTE! If you didn't add the `$QUAY_USER` to the `.bashrc` file then run the following 

```bash
QUAY_USER="your quay.io username not the email address"
```

```bash
sudo bootc switch quay.io/$QUAY_USER/demolab-homepage:latest
```

6. Let us check the we have staged the new homepage image in the virtual machine.

```bash
sudo bootc status
```

> Staged image: quay.io/$QUAY_USER/homepage:latest \
        Digest:  sha256:2be7b1...... \
       Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
       \
● Booted image: quay.io/$QUAY_USER/demolab-rhel:9.6 \
        Digest: sha256:a48811...... \
       Version: 9.6 (2025-07-21 13:10:35.887718188 UTC)

7. and we check that we have the old RHEL 9 homepage without our new Image Mode content.

```bash
curl localhost
```

8. We need to reboot the virtual machine to activate the new layers and have our new home page.

```bash
sudo reboot
```

9. Login to the virtual machine to verify that we have a new updated Image Mode homepage.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
curl localhost
```

10. Something went wrong! Our httpd service has failed during the update! Let us check the service.

```bash
sudo systemctl status httpd
```

11. There is no httpd service. We will rollback in the next section and fix the problem.

#### Rollback and fix our homepage

In the previous section the httpd service wasn't in the image. This is due to a mistake we made in the Containerfile. First we will rollback so that we have the old homepage up and running and then we will fix the problem.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_homepage_1 as Containerfile<br/>demolab-homepage:rhel9
    participant registry as Registry
    participant homepage_vm_1 as homepage VM

    %% New Homepage workflow
    homepage_vm_1->>homepage_vm_1: Rollback to the working homepage VM
    containerfile_homepage_1->>containerfile_homepage_1: Fix the error in the Containerfile demolab-homepage
    registry-->>containerfile_homepage_1: Build homepage image Containerfile<br/>from demolab-httpd:latest
    containerfile_homepage_1->>registry: Push homepage image v2<br/>as demolab-homepage:rhel9-fix to the registry
    containerfile_homepage_1->>registry: Push homepage image v2<br/>as demolab-homepage:latest to the registry
    registry-->>homepage_vm_1: Switch the homepage Virtual Machine to the demolab-homepage:latest <br/>image to update the web page
```

On our image builder server we will build a new Image Mode for RHEL 9 homepage image that we will deploy to the VM.

1. If you aren't in the `homepage-rhel9` directory then change directory to the new web page Container file and the updated web page at `homepage-rhel9`. You can open the `index.html` file in the `html` directory to see the updates to the homepage.

```bash
cd ../homepage-create
```

2. We need to fix the Containerfile to pull the correct image from the registry. Use an editor to change the following line to

```
FROM quay.io/$QUAY_USER/demolab-rhel:latest
```

change to

```
FROM quay.io/$QUAY_USER/demolab-httpd:latest
```

3. Build the new homepage images from the `Containerfile` and tag to a new version `homepage:rhel9-fix`.

```bash
podman build -t quay.io/$QUAY_USER/demolab-homepage:rhel9-fix -t quay.io/$QUAY_USER/demolab-homepage:latest -f Containerfile
```

4. Push the image to the registry using the `demolab-homepage:rhel9-fix` and `demopage-homepage:latest` tags.

```bash
podman push quay.io/$QUAY_USER/demolab-homepage:latest && podman push quay.io/$QUAY_USER/demolab-homepage:rhel9-fix
```

5. Switch to the Homepage virtual machine and login to the `homepage` VM using ssh.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

6. We are now going to use the `bootc switch` command to switch the virtual machine to the homepage image in the registry.

> NOTE! If you didn't add the `$QUAY_USER` to the `.bashrc` file then run the following 

```bash
QUAY_USER="your quay.io username not the email address"
```
```bash
sudo bootc switch quay.io/$QUAY_USER/demolab-homepage:latest
```

7. Let us check the we have staged the new homepage image in the virtual machine.

```bash
sudo bootc status
```

> Staged image: quay.io/$QUAY_USER/homepage:latest \
        Digest:  sha256:2be7b1...... \
       Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
       \
● Booted image: quay.io/$QUAY_USER/demolab-rhel:9.6 \
        Digest: sha256:a48811...... \
       Version: 9.6 (2025-07-21 13:10:35.887718188 UTC)

8. and we check that we have the old RHEL 9 homepage without our new Image Mode content.

```bash
curl localhost
```

9. We need to reboot the virtual machine to activate the new layers and have our new home page.

```bash
sudo reboot
```

10. Login to the virtual machine to verify that we have a new updated Image Mode homepage.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

```bash
curl localhost
```

#### Build the database virtual machine

We will then deploy a new virtual machine named `database` as this will be our new demo database server.
We will build the two images in one linked command and push it as the version 1 and latest images to our registry.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_database_1 as Containerfile<br/>demolab-database:rhel9
    participant registry as Registry
    participant database_vm_1 as database VM

    %% MainVM Creationsu workflow
    registry-->>containerfile_database_1: Build mariadb service image Containerfile<br/>from demolab-rhel:latest
    containerfile_httpd_1->>registry: Push mariadb service v1<br/>as demolab-database:rhel9.6 to the registry
    containerfile_httpd_1->>registry: Push mariadb service v1<br/>as demolab-database:latest to the registry
    registry-->>homepage_vm_1: Convert demolab-database:latest to Virtual Machine<br/> disk image and deploy VM database
```

We are following a less complex deployment for the database server than the deployment we did for the homepage.
We are going to deploy the mariadb service using a bash script to automate the deployment.

In the `mariadb_service` directory we need to update the QUAY_USER variable with your quay user id.

1. Change to the `mariadb-service` directory.

```bash
cd $HOME/imagemodedemo/mariadb-service
```

2. Ensure that the `mariadb-deploy.sh` file is executable.

```bash
chmod +x mariadb-deploy.sh
```

3. Edit the mariadb-deploy.sh file and change the entry for the QUAY_USER to your quay.io user name.

4. Run the bash script `mariadb-deploy.sh` to create the database images and the database VM.

```bash
./mariadb_deploy.sh
```

### Create a new RHEL 10 base image

We are going to create a new demolab-rhel base image that is based on the latest RHEL, version 10. This base image we are going to use to upgrade our services, httpd and mariadb. We created a new RHEL 10 homepage and then will upgrade the VMs to RHEL 10.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_rhel_10 as Containerfile<br/>demolab-rhel:10.0
    participant registry as Registry

    %% Base RHEL Creation workflow
    containerfile_rhel_10->>registry: Push base RHEL 10.0<br/>as demolab-rhel:10.0 image to the registry
    containerfile_rhel_10->>registry: Push base RHEL 10.0<br/>as demolab-rhel:latest image to the registry
```

1. Change to the RHEL 10 Container file directory to build the new RHEL 10 base image.

```bash
cd ../demolab-rhel10.0
```

2. Use Podman build to build the new RHEL 10 image and tag it as `demolab-rhel:latest` and `demolab-rhel:10.0`.

```bash
podman build -t quay.io/$QUAY_USER/demolab-rhel:latest -t quay.io/$QUAY_USER/demolab-rhel:10.0 -f Containerfile
```

3. Push the new images to the registry.

```bash
podman push quay.io/$QUAY_USER/demolab-rhel:latest && podman push quay.io/$QUAY_USER/demolab-rhel:10.0
```

### Upgrade the VM to RHEL 10 and update the homepage

Next we are going to build the httpd services image on RHEL 10 and upgrade the homepage VM.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_httpd_1 as Containerfile<br/>demolab-httpd:rhel10.0
    participant containerfile_homepage_1 as Containerfile<br/>demolab-homepage:rhel10.0
    participant registry as Registry
    participant homepage_vm_1 as homepage VM

    %% httpd VM Creation workflow
    registry-->>containerfile_httpd_1: Build httpd service image Containerfile<br/>from demolab-rhel:latest
    containerfile_httpd_1->>registry: Push httpd service v2<br/>as demolab-httpd:rhel10.0 to the registry
    containerfile_httpd_1->>registry: Push httpd service v2<br/>as demolab-httpd:latest to the registry
    registry-->>containerfile_homepage_1: Build homepage image Containerfile<br/>from demolab-httpd:latest
    containerfile_homepage_1->>registry: Push homepage image v3<br/>as demolab-homepage:rhel10.0 to the registry
    containerfile_homepage_1->>registry: Push homepage image v3<br/>as demolab-homepage:latest to the registry
    registry-->>homepage_vm_1: Upgrade the homepage Virtual Machine to the demolab-homepage:latest <br/>image to update the web page
```

1. Change directory to the httpd-service directory. Since we base our httpd image on the latest tagged RHEL base image in the repository, we can reuse the same Container file.

```bash
cd ../httpd-service
```

2. Use Podman build to build the new httpd images and we will tag the images as `demolab-httpd:latest` and `demolab-httpd:rhel10.0`. It is best practice to tag these images with version numbers or date stamps, but for the demo it makes it easier to track the RHEL version we are using.

```bash
podman build -t quay.io/$QUAY_USER/demolab-httpd:latest -t quay.io/$QUAY_USER/demolab-httpd:rhel10.0 -f Containerfile
```

3. Push the new httpd services to the registry.

```bash
podman push quay.io/$QUAY_USER/demolab-httpd:latest && podman push quay.io/$QUAY_USER/demolab-httpd:rhel10.0
```

4. Change to the homepage-rhel10 directory. This has an updated homepage for RHEL 10 with RHEL 10 logos.

```bash
cd ../homepage-rhel10
```

5. Build the new homepage images with the tags `demolab-homepage:latest` and `demolab-homepage:rhel10`. We fixed the ContainerFile in the previous section, and it will now use the httpd image and deploy correctly.

```bash
podman build -t quay.io/$QUAY_USER/demolab-homepage:latest -t quay.io/$QUAY_USER/demolab-homepage:rhel10 -f Containerfile
```

6. Push the updated homepage images to the registry.

```bash
podman push quay.io/$QUAY_USER/demolab-homepage:latest && podman push quay.io/$QUAY_USER/demolab-homepage:rhel10
```

7. No we switch to the `homepage` VM, we will use our special ssh command to log into the VM.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

8. Let's check if there is an update in the registry using the `bootc upgrade --check` command.

```bash
sudo bootc upgrade --check
```

>Update available for: docker://quay.io/$QUAY_USER/homepage:latest \
  Version: 10.0 \
  Digest: sha256:0c5416...... \
Total new layers: 77    Size: 885.4 MB \
Removed layers:   76    Size: 1.4 GB \
Added layers:     76    Size: 885.4 MB

9. Apply the upgrage to our VM. This may take a while as we are pulling RHEL 10 and the homepage updates in one go.

```bash
sudo bootc upgrade
```

10. Use `bootc status` to check that we have an update and that is shows that the update RHEL version is version 10.


```bash
sudo bootc status
```

> Staged image: quay.io/$QUAY_USER/homepage:latest \
        Digest: sha256:0c5416...... \
       Version: 10.0 (2025-07-21 17:25:47.229186615 UTC) \
 \
● Booted image: quay.io/$QUAY_USER/homepage:latest \
        Digest: sha256:2be7b1...... \
       Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
 \
  Rollback image: quay.io/$QUAY_USER/demolab-rhel:latest \
          Digest: sha256:7c46d6...... \
         Version: 9.6 (2025-07-21 16:04:36.100285429 UTC)

11. Reboot the VM to change to the new homepage and run RHEL 10!

```bash
sudo reboot
```

12. We use our special ssh command again to log into the VM.

```bash
VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

13. and check the OS version using `bootc status`

```bash
sudo bootc status
```

14. Finally use the VMs ip address and go to the web site to confirm the web page upgrade showing RHEL 10 logos.

This is to show how we update the base OS on an existing deployment. Usually this will be done during an application, or in this case, a homepage update.

### Upgrade the database server to RHEL 10

Similar we are going to build the database services image on RHEL 10 and upgrade the database VM. Since we don't have any application tied to the database we can upgrade our database VM directly from the database services image.

```mermaid
sequenceDiagram
    %% Actors
    participant containerfile_database_1 as Containerfile<br/>demolab-database:rhel10
    participant registry as Registry
    participant database_vm_1 as database VM

    %% MainVM Creationsu workflow
    registry-->>containerfile_database_1: Build mariadb service image Containerfile<br/>from demolab-rhel:latest
    containerfile_httpd_1->>registry: Push mariadb service v1<br/>as demolab-database:rhel10 to the registry
    containerfile_httpd_1->>registry: Push mariadb service v1<br/>as demolab-database:latest to the registry
    registry-->>homepage_vm_1: Upgrade the database Virtual Machine<br/>to demolab-database:latest
```

1. Change directory to the mariadb-service directory. Since we base our database service image on the latest tagged RHEL base image in the repository, we can reuse the same Container file.

```bash
cd ../mariadb-service
```

2. Use Podman build to build the new httpd images and we will tag the images as `demolab-database:latest` and `demolab-database:rhel10`. It is best practice to tag these images with version numbers or date stamps, but for the demo it makes it easier to track the RHEL version we are using.

```bash
podman build -t quay.io/$QUAY_USER/demolab-database:latest -t quay.io/$QUAY_USER/demolab-database:rhel10.0 -f Containerfile
```

3. Push the new httpd services to the registry.

```bash
podman push quay.io/$QUAY_USER/demolab-database:latest && podman push quay.io/$QUAY_USER/demolab-database:rhel10.0
```

4. No we switch to the database VM, we will use our special ssh command to log into the VM.

```bash
VM_IP=$(sudo virsh -q domifaddr database | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

5. Let's check if there is an update in the registry using the `bootc upgrade --check` command.

```bash
sudo bootc upgrade --check
```

>Update available for: docker://quay.io/$QUAY_USER/demolab-database:latest \
  Version: 10.0 \
  Digest: sha256:0c5416...... \
Total new layers: 77    Size: 885.4 MB \
Removed layers:   76    Size: 1.4 GB \
Added layers:     76    Size: 885.4 MB

6. Apply the upgrage to our VM. This may take a while as we are pulling RHEL 10 and the homepage updates in one go. Using  `--apply` the VM will be rebooted after the upgrade is done. 

```bash
sudo bootc upgrade --apply
```

7. Use our special ssh command to log into the VM again.

```bash
VM_IP=$(sudo virsh -q domifaddr database | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

8. Use `bootc status` to check that we have an update and that is shows that the update RHEL version is version 10.


```bash
sudo bootc status
```

> Staged image: quay.io/$QUAY_USER/homepage:latest \
        Digest: sha256:0c5416...... \
       Version: 10.0 (2025-07-21 17:25:47.229186615 UTC) \
 \
● Booted image: quay.io/$QUAY_USER/homepage:latest \
        Digest: sha256:2be7b1...... \
       Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
 \
  Rollback image: quay.io/$QUAY_USER/demolab-rhel:latest \
          Digest: sha256:7c46d6...... \
         Version: 9.6 (2025-07-21 16:04:36.100285429 UTC)```

9. Finally check that mariadb is running.

```bash
sudo systemctl status mariadb
```

10. We can also check the Linux OS version.

```bash
cat /etc/redhat-release
```

## Conclusion

This concludes the workshop exercises. We encourage you to try different services and applications based on the base image `demolab-rhel` that we used in these exercise. We also encourage you to build your own base or corporate image and build and deploy servers using it. You can use Podman Desktop for many of the executions that we did in the command line, and using a desktop approach may be easier for you. Finally, we didn't incorporate any pipelines or CI/CD flows in these examples and using these tools to test and deploy updates makes the task of a system administrator a lot easier. Our Youtube channel "Into the Terminal" episode 151 has a great introduction to this.
