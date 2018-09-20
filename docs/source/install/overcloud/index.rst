Install the Overcloud
=====================

Deploying RHOSP overcloud is the same as the deployment guide. Except that
before you start the deployment, you need to execute a couple of extra steps
as given below:



1. Render the jinja2 templates into yaml format
###############################################

The templates under `/usr/share/openstack-tripleo-heat-templates/` are now in
`jinja2` format instead of plan `yaml`. Hence, we first need to render it in
yaml using the following commands:

::

    # create an overcloud plan
    openstack overcloud plan create --templates /usr/share/openstack-tripleo-heat-templates overcloud-plan

    # download the plan to /home/stack/overcloud-plan
    mkdir ~/overcloud-plan
    cd ~/overcloud-plan
    swift download overcloud-plan

2. Add service file for Neutron Bigswitch Agent [PV ONLY]
#########################################################

Copy neutron-bigswitch-agent.yaml from `<extracted plugin tarball>/yamls` to
`overcloud-plan/docker/services`.

3. Add an extra mount point in the neutron-api service container
################################################################

Our Neutron plugin uses `os-net-config` to provide intelligent interface
grouping. So add `/etc/os-net-config` as a mount volume for neutron-api service.

::

    sed -i '0,/                      - \/var\/lib\/config-data\/puppet-generated\/neutron\/\:\/var\/lib\/kolla\/config_files\/src:ro/s//                      - \/var\/lib\/config-data\/puppet-generated\/neutron\/\:\/var\/lib\/kolla\/config_files\/src:ro\n                      - \/etc\/os-net-config\:\/etc\/os-net-config\:ro/' /home/stack/overcloud-plan/docker/services/neutron-api.yaml

3.1 Add extra mount point in the nova-compute service container [PV ONLY]
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Our Nova Compute plugin uses IVS commands to create interfaces and perform other IVS functions.
This requires access to IVS that is installed on the baremetal. This is done by
sharing the runtime, by mounting the `/usr/sbin` volume.

::

    sed -i '0,/                  - \/lib\/modules\:\/lib\/modules\:ro/s//                  - \/lib\/modules\:\/lib\/modules\:ro\n                  - \/usr\/sbin\:\/usr\/sbin/' /home/stack/overcloud-plan/docker/services/nova-compute.yaml

4. Add Neutron Bigswitch Agent service to Compute role [PV ONLY]
################################################################

Unlike previous versions of RHOSP, deployment command no longer takes
`-r /path/to/roles_data.yaml` argument. It assumes that `roles_data.yaml` is
present in the plan directory passed as `--templates /path/to/overcloud-plan`.

Hence, we edit `/home/stack/overcloud-plan/roles_data.yaml` to add Neutron
BSN Agent service for compute nodes in P+V deployments.

::

    (undercloud) $ vi /home/stack/overcloud-plan/roles_data.yaml
    # add this as the last service in the list of services for Compute role
    - OS::TripleO::Services::NeutronBigswitchAgent


5. Validate final deployment plan before running overcloud deploy
#################################################################

Once you've edited all the templates under `/home/stack/templates` and updated
any yaml files under `/home/stack/overcloud-plan`, you can verify the final plan
by generating an output using the following command:

::

    # validating
    openstack orchestration template validate --show-nested \
        --template ~/overcloud-plan/overcloud.yaml \
        -e ~/overcloud-plan/overcloud-resource-registry-puppet.yaml \
        -e /home/stack/templates/node-info.yaml \
        -e /home/stack/templates/overcloud_images.yaml \
        -e /home/stack/overcloud-plan/environments/network-isolation.yaml \
        -e /home/stack/templates/network-environment.yaml \
        -e /home/stack/templates/bigswitch-config-p.yaml  \
        -e /home/stack/templates/overcloud_images_bigswitch.yaml

.. note:: Ensure you're using the correct Bigswitch template when running PV
          mode i.e. `bigswitch-config-pv.yaml`.
          Same in the next step as well.


6. Sample deployment command
##################################

::

    # QUEENS
    #!/bin/bash

    source /home/stack/stackrc
    openstack overcloud deploy \
        --templates ~/overcloud-plan \
        -e ~/overcloud-plan/overcloud-resource-registry-puppet.yaml \
        -e /home/stack/templates/node-info.yaml \
        -e /home/stack/templates/overcloud_images.yaml \
        -e /home/stack/overcloud-plan/environments/network-isolation.yaml \
        -e /home/stack/templates/network-environment.yaml \
        -e /home/stack/templates/bigswitch-config-p.yaml  \
        -e /home/stack/templates/overcloud_images_bigswitch.yaml \
        --ntp-server 10.5.4.109 --timeout 150 | tee /home/stack/deploy.log

.. note:: We include the `overcloud-plan` as the base template directory. This
          is because `openstack-tripleo-heat-templates` is not longer used for
          that purpose.
