Install the Undercloud
======================

This section provides details on the installation of the RHOSP Undercloud Director node.

Procedure
#########

1. Install the RHEL 7.4 operating system
****************************************
Install the RHEL 7.4 operating system on the bare metal server and configure
the following settings during the installation:

::

    Select Infrastructure Server
    Select HDD Section “ I will create partitioning”
    Allocate at least 200 GB space to root partition
    Begin the RHEL installation , set the root password


2. Create the stack user on RHEL
********************************

The director installation process requires a non-root user to execute commands.
Create the stack user on RHEL and modify the setting as shown below.

::

    useradd stack
    passwd stack
    echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
    chmod 0440 /etc/sudoers.d/stack

3. Verify the selinux flag is set to permissive
***********************************************

Verify that the selinux flag in the /etc/selinux/config file to be permissive
or disabled as below:

::

    SELINUX=permissive

4. Set FQDN for Undercloud host
*******************************

The Director requires a fully qualified domain name for installation and
configuration. The Director also requires an entry for the system hostname and
base name in /etc/hosts.

For example, if the host name is rhel-dell-71, use the following settings:

::

    127.0.0.1 rhel-dell-71 localhost localhost.localdomain localhost4 localhost4.localdomain4

6. Create directories for images and templates
**********************************************

The Director uses system images and Heat templates to create the Overcloud
environment. To keep these files organized, create the following directories
for the stack user you created in `2. Create the stack user on RHEL`_.

::

    su - stack
    mkdir ~/images
    mkdir ~/templates

7. Register to Red Hat Subscription Manager
*******************************************
To install the RHEL OpenStack Platform installer, first register the host
system using Red Hat Subscription Manager, and subscribe to the required
channels as shown below.

.. code-block:: bash

    sudo subscription-manager register
    # Provide username and password when prompted
    sudo subscription-manager list --available --all
    sudo subscription-manager attach --pool=<pool id> ----- select a pool with openstack
    sudo subscription-manager repos --disable=*
    sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-13-rpms

8. Update packages
******************

Perform an update to put the latest base system packages on the Director node
and reboot.

::

    sudo yum update –y
    sudo reboot

9. Install python-tripleoclient
*******************************

After reboot, login as root and change to user stack by entering the su – stack
command so you can run all commands as the stack user. Install package
“python-tripleoclient” as shown below:

::

    su – stack
    sudo yum install -y python-tripleoclient


10. Copy undercloud.conf
************************

Red Hat provides a basic template to help determine the required settings for
your installation. Copy this template to the stack user home directory:

::

    cp /usr/share/instack-undercloud/undercloud.conf.sample /home/stack/undercloud.conf

The Director installation process requires certain settings to determine your
network configuration. The settings are stored in a template located in the
stack user home directory as undercloud.conf.

11. Ensure proper file ownership
********************************

Make sure to change the ownership of all the files under /home/stack by the stack user

::

    chown stack:stack *

12. Configure undercloud.conf
*****************************

Uncomment all the lines shown below from the undercloud.conf file, assuming
that the default IP address used does not conflict in your setup. Change only
the local interface (interface used for openstack node PXE, refer to figure 35)
matching your Undercloud physical network connection and save the file.

The undercloud.conf file contains the image path and Undercloud private IP
information:

::

    local_ip = 192.0.2.1/23
    network_gateway = 192.0.2.1
    local_interface = em2
    network_cidr = 192.0.2.0/23
    masquerade_network = 192.0.2.0/23
    dhcp_start = 192.0.2.5
    dhcp_end = 192.0.3.150
    inspection_interface = br-ctlplane
    inspection_iprange = 192.0.3.151,192.0.3.250
    undercloud_debug = true

.. note:: If you change IP subnet here for any reason that needs to be
          reflected later in Network-enviornment.yaml file later when we
          deploy the overcloud.

13. Deploy the Undercloud
*************************

Run the following command to launch the Director configuration script. The
Director installs additional packages and configures its services to match the
settings in undercloud.conf.
This script takes several minutes to complete.

::

    openstack undercloud install

14. Configure domain name for Undercloud neutron and nova
*********************************************************

Edit the nova.conf and neutron.com files to add the DHCP and DNS domain names
as in the following example:

::

    edit /etc/nova/nova.conf
    add dhcp_domain = <domain name>

    edit /etc/neutron/neutron.conf
    add dns_domain = <domain name>

    systemctl list-units | egrep 'nova|neutron' | awk ' {print $1} ' | xargs -I {} systemctl restart {}

15. Extract Overcloud disk images
*********************************

The Director requires several disk images for provisioning Overcloud nodes.
Install and copy the images to the stack user home on the directory host
(/home/stack/images/) and extract the images from the archives:

::

    source /home/stack/stackrc
    sudo yum -y install libguestfs-tools
    sudo yum install rhosp-director-images rhosp-director-images-ipa
    # Extract the images
    cd ~/images
    for i in /usr/share/rhosp-director-images/overcloud-full-latest-13.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-13.0.tar; do tar -xvf $i; done

16. Download BCF plugin tarball
*******************************

Download the release specific BCF-RHOSP-plugins tar file from the Big Switch
Networks repository and copy it to the /home/stack/images directory and extract
the file.

::

    # The format of the file name is as follows:
    #BCF-RHOSP-13-plugins-<release-name-date>.tar.gz, where <release-name-date> is the name and date of the release.
    # Extract the rpm packages from tar file
    tar –xvf BCF-RHOSP-13-plugins-<release-name-date>.tar.gz

17. Move BCF plugin files to 'images'
*************************************
Move the extracted rpm/script files from this directory
“BCF-RHOSP-plugins-<release-name-date>” to /home/stack/images:

::

    mv BCF-RHOSP-13-plugins-<release-name-date>/* /home/stack/images

18. Set root password for Overcloud image (optional)
****************************************************

Optionally, you can set the root password in the overcloud-full.qcow2 image,
by updating the overcloud image.
This password is used to log in to the compute node when you need to access
Overcloud compute nodes.

::

    virt-customize -a overcloud-full.qcow2 --root-password password:<password>

19. Patch overcloud image with BCF LLDP script
**********************************************

Download the neutron-bsn-lldp from bigtop:

::

    cd /home/stack/images
    curl -O http://bigtop.eng.bigswitch.com/~bsn/neutron-bsn-lldp/centos7-x86_64/origin/master/0.0.1/neutron-bsn-lldp-0.0.1-1.el7.centos.noarch.rpm ./

Create startup.sh script:

::

    # ensure startup.sh has the following:
    (undercloud) $ vi startup.sh
    yum remove -y openstack-neutron-bigswitch-agent
    yum remove -y openstack-neutron-bigswitch-lldp
    yum remove -y python-networking-bigswitch

    yum remove -y neutron-bsn-lldp
    rpm -ivhU --force /root/neutron-bsn-lldp-0.0.1-1.el7.centos.noarch.rpm
    systemctl enable neutron-bsn-lldp.service
    systemctl restart neutron-bsn-lldp.service

Create customize.sh script:

::

    # ensure customize.sh has the following:
    (undercloud) $ vi customize.sh
    export LIBGUESTFS_BACKEND=direct

    image_dir="/home/stack/images"

    virt-customize -a ${image_dir}/overcloud-full.qcow2 --upload neutron-bsn-lldp-0.0.1-1.el7.centos.noarch.rpm:/root/
    virt-customize -a ${image_dir}/overcloud-full.qcow2 --firstboot startup.sh

Make both customize.sh and startup.sh as executables and run customize.sh:

::

    chmod +x startup.sh customize.sh
    ./customize.sh

19.1 Include IVS in customize.sh and startup.sh [PV ONLY]
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the current step, uncomment the lines from `customize.sh` and `startup.sh` regarding IVS, when doing a P+V installation.
This step  is not required for P-only deployments.

Insert the appropriate version number for IVS based on the package that you received.

20. Upload the customized Overcloud image
*****************************************
Run the following command to import these images into the Director:

::

    openstack image delete overcloud-full # Only needed If you have uploaded image before.
    openstack overcloud image upload

21. Configure Overcloud container images
****************************************
A containerized Overcloud requires access to a registry with the required
container images.

This process is outlined in the following document -
https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/configuring-a-container-image-source

The following example uses the “Local Registry” type to create a container registry.

Upstream docker containers are pulled from `registry.access.redhat.com` and Big Switch Networks specific containers are pulled from `registry.connect.redhat.com`.
Login to redhat portal is required before pulling Big Switch Networks specific containers. All the steps are detailed below.

21.1. Create a template to upload the images to the local registry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    (undercloud) $ source stackrc
    (undercloud) $ sudo openstack overcloud container image prepare \
    --namespace=registry.access.redhat.com/rhosp13 \
    --push-destination=192.168.24.1:8787 \
    --prefix=openstack- \
    --tag-from-label {version}-{release} \
    --output-env-file=/home/stack/templates/overcloud_images.yaml \
    --output-images-file /home/stack/local_registry_images.yaml

21.2. This creates two files, check that they exist
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The previous step creates two files:

 - `local_registry_images.yaml`, which contains container image information from the remote source. Use this file to pull the images from the Red Hat Container Registry (registry.access.redhat.com) to the undercloud.
 - `overcloud_images.yaml`, which contains the eventual image locations on the undercloud. You include this file with your deployment.

Check that both files exist.

.. note::  Also make sure to copy `local_registry_images_bigswitch.yaml` and
           `overcloud_images_bigswitch.yaml` from the extracted plugin folder
           to the respective folders `/home/stack` and `/home/stack/templates`.

21.3. Pull the container images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Copy `local_registry_images_bigswitch.yaml` from the `yamls` directory of the extracted tarball of Big Switch Networks plugin.

::

    (undercloud) $ cp /home/stack/images/<BCF Plugin tarball>/yamls/local_registry_images_bigswitch.yaml /home/stack
    (undercloud) $ cat local_registry_images_bigswitch.yaml
    container_images:
    - imagename: registry.connect.redhat.com/bigswitch/rhosp13-openstack-neutron-server-bigswitch:13.0-1
      push_destination: <REGISTRY_IP>:8787

    # REGISTRY_IP == undercloud ctlplane IP == 192.168.24.1

.. note:: Update the `REGISTRY_IP` to the undercloud ctlplane IP.

.. warning:: Remove Horizon related container images if present. These are
             not working right now due to HTTPD permission issue. I will update
             the guide and sent a note as soon as its available.

Login to the Red Hat portal before pulling the container images:

::

   (undercloud) $ sudo docker login registry.connect.redhat.com

Upload images to local registry:

::

    (undercloud) $ sudo openstack overcloud container image upload \
    --config-file /home/stack/local_registry_images.yaml \
    --config-file /home/stack/local_registry_images_bigswitch.yaml \
    --verbose

21.3.1 Include IVS and Neutron BSN Agent [PV ONLY]
__________________________________________________

Ensure that you're using the Big Switch nova-compute container when deploying in P+V mode.

The `local_registry_images_bigswitch.yaml` looks as follows:

::

    container_images:
    - imagename: registry.connect.redhat.com/bigswitch/rhosp13-openstack-neutron-server-bigswitch:13.0-1
      push_destination: 192.168.25.1:8787
    - imagename: registry.connect.redhat.com/bigswitch/rhosp13-openstack-nova-compute-bigswitch:13.0-2
      push_destination: 192.168.25.1:8787

The `templates/overcloud_images_bigswitch.yaml` looks as follows:

::

    parameter_defaults:
      DockerNeutronApiImage: 192.168.25.1:8787/bigswitch/rhosp13-openstack-neutron-server-bigswitch:13.0-1
      DockerNeutronConfigImage: 192.168.25.1:8787/bigswitch/rhosp13-openstack-neutron-server-bigswitch:13.0-1
      DockerNeutronBigswitchAgentImage: 192.168.25.1:8787/bigswitch/rhosp13-openstack-neutron-server-bigswitch:13.0-1
      DockerNovaComputeImage: 192.168.25.1:8787/bigswitch/rhosp13-openstack-nova-compute-bigswitch:13.0-2
      DockerNovaLibvirtConfigImage: 192.168.25.1:8787/bigswitch/rhosp13-openstack-nova-compute-bigswitch:13.0-2

.. note:: Make sure you run `sudo openstack overcloud container image upload`
          after you edit the `local_registry_images_bigswitch.yaml` file and
          add nova-compute-bigswitch container.

21.4. The images are now stored on the undercloud’s docker-distribution registry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To view the list of images on the undercloud’s docker-distribution registry using the following command:

::

    (undercloud) $  curl http://192.168.24.1:8787/v2/_catalog | jq .repositories[]

To view a list of tags for a specific image, use the skopeo command:

::

    (undercloud) $ skopeo inspect --tls-verify=false docker://192.168.24.1:8787/rhosp13/openstack-keystone | jq .RepoTags[]

To verify a tagged image, use the skopeo command:

::

    (undercloud) $ skopeo inspect --tls-verify=false docker://192.168.24.1:8787/rhosp13/openstack-keystone:13.0-44

The registry configuration is ready.

22. Define the nameserver for the environment
*********************************************

Overcloud nodes require a nameserver so that they can resolve hostnames through
DNS. The nameserver is defined in the Undercloud neutron subnet.

::

    openstack subnet list
    openstack subnet set --dns-nameserver 8.8.8.8 [subnet-uuid]
    # To Verify
    openstack subnet show [subnet-uuid]

This completes the Undercloud Director node installation and configuration.
Now it is ready to deploy the Overcloud (OpenStack cluster).

23. Install Big Switch's Neutron Client for CLI
***********************************************

Install `python-bsn-neutronclient` package on the undercloud to enable Neutron
CLI commands specific to Big Switch Network's plugin. The package is included
in the tarball package. Exceute the following to ensure a clean install:

::

    sudo rpm -e python-bsn-neutronclient
    cd /home/stack/images/<RHOSP-PLUGINS-TARBALL-EXTRACT>
    sudo rpm -ivhU python-bsn-neutronclient*.noarch.rpm

You can now execute BSN plugin commands such as `force-bcf-sync` and
`bcf-sync-status` from the undercloud after doing a `source overcloudrc`.
