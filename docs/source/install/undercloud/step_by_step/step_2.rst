Create the stack user on RHEL
=============================

The director installation process requires a non-root user to execute commands.
Create the stack user on RHEL and modify the setting as shown below.

::

    useradd stack
    passwd stack
    echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
    chmod 0440 /etc/sudoers.d/stack
