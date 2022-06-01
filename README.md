# ansible: restart and/or ensure a service is running

From my way of understanding, a configuration management and provisioning tool should leave a host in the desired and described state.
So, if a ansible role is about ***configure service xyz***, at the end of a playbook run the target host should have the service set up with the desired configuration and the service should be up and running as long as the configuration has no errors. And a role should be completely conditional!

In order to demonstrate the (not-so-easy and not-so-obviously) way to accomplish this with ansible, I created different sample roles here ... all with some caveats and shortcomings, especially if the service in question is not running on the start of the playbook.
During a common deploy all might seem simple. But during tests a service might not be running on the start of the playbook due to a manual stop before, a test deploy with a broken config, etc... and here many playbools fail and finally do not leave you with a configured and running service at the end.

sample roles:
 * `testservicenormal` - the ansiblish way : just call a handler
 * `testservicedoublestart`- try to ensure the service is running, but this might restart the service twice
 * `testservicewarn` - avoid double restarts in exchange for a ansible warning
 * `testserviceok`- register the desired state of a service and finally leave it in that state ... in exchange for a ansible-lint warning

## the ansiblish way
Using handlers is the common ansible way, but handlers are run at the end of a play, so if subsequent roles rely on the service to be up and running this might be problematic, as the handler will not been run inbetween ...
```diff
TASK [testservicenormal : configure apache] ***********************************************************************************************************************************************************************
changed: [testhost]

TASK [testservice : check apache is listening on port 80] *********************************************************************************************************************************************************
-fatal: [testhost]: FAILED! => {"changed": false, "elapsed": 2, "msg": "Timeout waiting for 80 to respond"}

RUNNING HANDLER [testservicenormal : restart apache2]
```

Another problem with handlers not beeing run at the end of a role, is that the trigger of a restart will be lost if a subsequent role will fail. Imagine:
 * role _x_ changes the connfig of a service and trigger a restart
 * role _y_ executed after _x_ fails, the play ends
 * you fix the problem in _y_ and start a new run, but _x_ is finished and will make any changes, thus no restart of the service in _x_ will be triggered
 * finally your play is done, but the service in _x_ is **not** running with the desired config, as it was not restarted/reloaded

## try to ensure the service is running
So we can try to extend the role by ensuring the service is started at the end of the run
```yaml
  service:
    state: started
```
But this might lead to double (re)starts of the service
```
TASK [testservicedoublestart : ensure apache service state is started] ********************************************************************************************************************************************
changed: [testhost]

TASK [testservice : check apache is listening on port 80] *********************************************************************************************************************************************************
ok: [testhost]

RUNNING HANDLER [testservicedoublestart : restart apache2] ********************************************************************************************************************************************************
changed: [testhost]
```
and some services can take ages for restarting ...

## flush handlers before ensuring the service is running
So one can think: OK, LGTM, but in order to avoid the double (re)starts, we can call `flush_handlers` before ensuring the service is started ...
This is where the effect of https://github.com/ansible/ansible/issues/41313 comes in.
Unfortunately `flush_handlers` is a meta task and this will issue a warning if called conditionally.
```diff
TASK [testservicewarn : run all notified handlers at this point] **************************************************************************************************************************************************
@@[WARNING]: flush_handlers task does not support when conditional@@

RUNNING HANDLER [testservicewarn : restart apache2] ***************************************************************************************************************************************************************
changed: [testhost]

TASK [testservicewarn : ensure apache service state is started] ***************************************************************************************************************************************************
ok: [testhost]
```
All fine and as expected, but the f... warning is annoying ...

## register the desired state of the service and apply it
So, if the handlers make problems, we can register a required restart of the service and on the end ensure the service has the desired state, something like
```yaml
- name: configure service
  register: cfgchanged

- name: ensure service state
  service:
  state: "{% if cfgchanged.changed %}restarted{% else %}started{% endif %}"
```
or you can register the state after a task triggering a restart as fact
```yaml
- name: configure service
  register: cfgchanged

- name: register desired state
  set_fact:
    service_state: restarted
  when: cfgchanged.changed | default(false)

- name: ensure service state is {{ service_state | default('started') }}
  service:
    state: "{{ apache_service_state | default('started') }}"
```
resulting in
```
TASK [testserviceok : configure apache] ***************************************************************************************************************************************************************************
changed: [testhost]

TASK [testserviceok : register apache2 desired state] *************************************************************************************************************************************************************
ok: [testhost]

TASK [testserviceok : ensure apache service state is restarted] ***************************************************************************************************************************************************
changed: [testhost]

TASK [testservice : check apache is listening on port 80] *********************************************************************************************************************************************************
ok: [testhost]
```

So, the last step seems to bring us to the desired state, yeah, but ... ah, wait, I promised you that all solutions will have some caveats ... here ansible-lint will complain
```diff
!no-handler: Tasks that run when changed should likely be handlers
```

You can supress this using the `# noqa no-handler` [see example](main/roles/testserviceok/tasks/main.yml#L41) for the task in question, but it remains a supressed warning and the ansible-lint handlingh might change someday at some version ...

So, choose your poison ...

