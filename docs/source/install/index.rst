Installation Summary
====================

The RedHat OpenStack installation consists of two phases:

    1. Install the RHOSP Undercloud Director node. This is a single-system
       OpenStack installation that includes components for provisioning and managing
       the OpenStack nodes that comprise the OpenStack environment (known as the
       Overcloud).
    2. The Overcloud deployment, which is the resulting Red Hat Enterprise
       Linux OpenStack Platform environment created using the Undercloud. This
       includes one or more of the following node types: OpenStack controllers,
       compute nodes, and storage nodes that part of the OpenStack cluster. The
       Big Cloud Fabric OpenStack integration components are installed as part of
       the Overcloud deployment.

This document describes the procedure for a joint RHOSP and BCF deployment as
used during testing by Big Switch Networks. It provides a good reference point
when deploying a joint solution. But we recommend referring to the Red Hat
documentation for an in-depth description of all available deployment options.

The RHOSP deployment guide for RHOSP 12 is available at
https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html/director_installation_and_usage
