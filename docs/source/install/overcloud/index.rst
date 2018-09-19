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

2. Add an extra mount point in the neutron-api service container
################################################################

Our Neutron plugin uses `os-net-config` to provide intelligent interface
grouping. So add `/etc/os-net-config` as a mount volume for neutron-api service.

::

    awk '{print} /^                      - \/var\/lib\/config-data\/puppet-generated\/neutron\/\:\/var\/lib\/kolla\/config_files\/src:ro/ && !n {print "                      - \/etc\/os-net-config\:\/etc\/os-net-config\:ro"; n++}' /home/stack/overcloud-plan/docker/services/neutron-api.yaml > /home/stack/overcloud-plan/docker/services/neutron-api.yaml

3. Validate final deployment plan before running overcloud deploy
#################################################################

Once you've edited all the templates under `/home/stack/templates` and updated
any yaml files under `/home/stack/overcloud-plan`, you can verify the final plan
by generating an output using the following command:

::

    # validating
    openstack orchestration template validate --show-nested \
        --template ~/test-validation/overcloud.yaml \
        -e ~/test-validation/overcloud-resource-registry-puppet.yaml \
        -e /home/stack/templates/node-info.yaml \
        -e /home/stack/templates/overcloud_images.yaml \
        -e /home/stack/test-validation/environments/network-isolation.yaml \
        -e /home/stack/templates/network-environment.yaml \
        -e /home/stack/templates/bigswitch-config-p.yaml  \
        -e /home/stack/templates/overcloud_images_bigswitch.yaml


3. Sample deployment command
##################################

::

    # QUEENS
    #!/bin/bash

    source /home/stack/stackrc
    openstack overcloud deploy \
        --templates ~/test-validation \
        -e ~/test-validation/overcloud-resource-registry-puppet.yaml \
        -e /home/stack/templates/node-info.yaml \
        -e /home/stack/templates/overcloud_images.yaml \
        -e /home/stack/test-validation/environments/network-isolation.yaml \
        -e /home/stack/templates/network-environment.yaml \
        -e /home/stack/templates/bigswitch-config-p.yaml  \
        -e /home/stack/templates/overcloud_images_bigswitch.yaml \
        --ntp-server 10.5.4.109 --timeout 150 | tee /home/stack/deploy.log

.. note:: We include the `test-validation` as the base template directory. This
          is because `openstack-tripleo-heat-templates` is not longer used for
          that purpose.
