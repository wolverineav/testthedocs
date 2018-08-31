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

5. Enable IP forwarding
***********************

Enable IP forwarding on the Director host by completing this step:

::

    vi /etc/sysctl.conf
    net.ipv4.ip_forward = 1
    sudo sysctl -p /etc/sysctl.conf

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
    sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-7-server-openstack-12-rpms

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
    for i in /usr/share/rhosp-director-images/overcloud-full-latest-12.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-12.0.tar; do tar -xvf $i; done

16. Configure Overcloud container images
****************************************
A containerized Overcloud requires access to a registry with the required
container images.

This process is outlined in the following document -
https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html-single/director_installation_and_usage/#Configuring-Registry_Details

The following example uses the “Local Registry” type to create a container registry.

16.1. Discover the tag for the latest images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    source stackrc
    sudo openstack overcloud container image tag discover \
    --image registry.access.redhat.com/rhosp12/openstack-base:latest \
    --tag-from-label version-release
    # The result from this command is used below for the value of <TAG>.
    <TAG> == 12.0-20180124.1 # << Sample Output

16.2. Create a template to pull the images to the local registry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    openstack overcloud container image prepare \
    --namespace=registry.access.redhat.com/rhosp12 \
    --prefix=openstack- \
    --tag=<TAG> \
    --output-images-file /home/stack/local_registry_images.yaml
.. note:: Note: For more information about “openstack overcloud container
          image prepare” command refer to the following:
          https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html-single/director_installation_and_usage/#Configuring-Preparing_the_Container_Images_File

16.3. Pull container images to local regsitry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This creates a file called local_registry_images.yaml with your container image
 information in the home directory. Pull the images using the
 local_registry_images.yaml file:
::

    sudo openstack overcloud container image upload \
        --config-file /home/stack/local_registry_images.yaml \
        --verbose

.. note:: Note: Pulling the required images might take some time depending on
          the speed of your network and your undercloud disk.

16.4. Find the namespace of the local images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The namespace uses the following pattern:

::

    <REGISTRY IP ADDRESS>:8787/rhosp12

Use the IP address of your undercloud, which you previously set with the
local_ip parameter in your undercloud.conf file. Alternatively, you can also
obtain the full namespace with the following command:

::

    (undercloud) $ docker images | grep -v redhat.com | grep -o '^.*rhosp12' | sort -u

16.5. Create a template for using the images in our local registry on the undercloud
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For example:

::

    (undercloud) $ openstack overcloud container image prepare \
        --namespace=192.168.24.1:8787/rhosp12 --prefix=openstack- \
        --tag=<TAG> \
        --output-env-file=/home/stack/templates/overcloud_images.yaml

This creates an overcloud_images.yaml environment file, which contains image
locations on the Undercloud. Include this file with the overcloud deployment
command.

17. Download BCF plugin tarball
*******************************

Download the release specific BCF-RHOSP-plugins tar file from the Big Switch
Networks repository and copy it to the /home/stack/images directory and extract
the file.

::

    # The format of the file name is as follows:
    #BCF-RHOSP-12-plugins-<release-name-date>.tar.gz, where <release-name-date> is the name and date of the release.
    # Extract the rpm packages from tar file
    tar –xvf BCF-RHOSP-12-plugins-<release-name-date>.tar.gz

18. Move BCF plugin files to 'images'
*************************************
Move the extracted rpm/script files from this directory
“BCF-RHOSP-plugins-<release-name-date>” to /home/stack/images:

::

    mv BCF-RHOSP-12-plugins-<release-name-date>/* /home/stack/images

19. Set root password for Overcloud image (optional)
****************************************************

Optionally, you can set the root password in the overcloud-full.qcow2 image,
by updating the overcloud image.
This password is used to log in to the compute node when you need to access
Overcloud compute nodes.

::

    virt-customize -a overcloud-full.qcow2 --root-password password:<password>

20. Patch overcloud image with BCF plugins
******************************************
Patch all the extracted RPM files from the Big Switch RHOSP plugin tar files
to overcloud-full.qcow2 by running following script from extracted file.

::

    source /home/stack/stackrc
    cd /home/stack/images
    chmod +x customize.sh
    ./customize.sh

21. Upload the customized Overcloud image
*****************************************
Run the following command to import these images into the Director:

::

    openstack image delete overcloud-full # Only needed If you have uploaded image before.
    openstack overcloud image upload

22. Customize container images
******************************

Customize Horizon and Nova compute container images to include Big Switch packages:

::

    cd /home/stack/images
    ./customize_horizon_container.sh
    ./customize_nova_compute_container.sh

.. note:: The above step assumes the installation is using the Local Registry
          type as outlined in `16. Configure Overcloud container images`_.

23. Define the nameserver for the environment
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
