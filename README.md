# ansible: ensure a service is running
or: how to get arround https://github.com/ansible/ansible/issues/41313

From my way of understanding a, configuration management and provisioning tool should leave a host in the desired and described state.
So, if a ansible role is about 'configure service xyz', at the end of a playbook run the target host should have the service set up with the desired configuration and the service should be up and running as long as the configuration has no errors.

And a role should be completely conditional!
