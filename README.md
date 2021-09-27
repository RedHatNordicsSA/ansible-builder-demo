# Simple Ansible Execution Enviroment Demo

A short and simple demo of how to build an Ansible Execution Environment and execute playbooks in it.

### Setup: (Tested with Fedora 34 and RHEL)

We will run this demo in a Python virtual enviroment, so let's set that up first:

```
> mkdir ~/ansible-builder && cd ~/ansible-builder
> python -m venv builder
> source builder/bin/activate

> pip install ansible
> pip install ansible-builder
> pip install ansible-navigator
```
Verify that the modules are installed:
```
> pip list
Package             Version
------------------- -------
ansible             4.5.0
ansible-builder     1.0.1
ansible-core        2.11.4
ansible-navigator   1.0.0
ansible-runner      2.0.2
bindep              2.9.0
cffi                1.14.6
cryptography        3.4.8
distro              1.6.0
docutils            0.17.1
Jinja2              3.0.1
lockfile            0.12.2
MarkupSafe          2.0.1
onigurumacffi       1.1.0
packaging           21.0
Parsley             1.3
pbr                 5.6.0
pexpect             4.8.0
pip                 21.2.4
ptyprocess          0.7.0
pycparser           2.20
pyparsing           2.4.7
python-daemon       2.3.0
PyYAML              5.4.1
requirements-parser 0.2.0
resolvelib          0.5.4
setuptools          53.0.0
six                 1.16.0
wheel               0.37.0
```
Now we need to create some files to define our Execution Enviroment (EE): (..or just do a git clone from this repo, and move the files to ansible-builder folder)

Create execution-enviroment.yml
```
---
version: 1
dependencies:
  galaxy: requirements.yml
  python: requirements.txt


additional_build_steps:
  prepend: |
    RUN pip3 install --upgrade pip setuptools
  append:
    - RUN ls -la /etc
```
Create requirements.yml where we add desired ansible collections:
```
---
collections:
  - community.general
```
Then we create requirements.txt where we define needed Python module:
```
urllib3
```
Last but least, we create a playbook (test.yml) that checks if the required Python module is installed in our EE:
```
---
- hosts: localhost
  connection: local
  gather_facts: false


  tasks:
  - name: Just get the list from default pip
    community.general.pip_package_info:
      clients: pip
    register: pips

  - debug:
      msg: "{{ pips.packages.pip.urllib3 | default('nope, not installed') }}"
```
Now, let's build a container with our EE!
```
ansible-builder build -t jbond_ee_image:0.0.7
```
Verify that the images has been created:
```
> podman images
REPOSITORY                                 TAG         IMAGE ID      CREATED         SIZE
localhost/jbond_ee_image                   0.0.7       6485ad35289a  10 seconds ago  747 MB
```
For the purpose of the demo, run the test.yml locally and let it fail (given that you have not installed the required Python module locally). 
Depending on setup, this will throw an error, or give an output like this:
```
❯ ansible-playbook test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ******************************************************************************************

TASK [Just get the list from default pip] *****************************************************************
ok: [localhost]

TASK [debug] **********************************************************************************************
ok: [localhost] => {
    "msg": "nope, not installed"
}

PLAY RECAP ************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Notice the "nope, not installed". Then let's run the playbook inside our newly created EE:
```
ansible-navigator run -m stdout --eei jbond_ee_image:0.0.7 test.yml
```
Output should contain:
```
TASK [debug] *******************************************************************
ok: [localhost] => {
    "msg": [
        {
            "name": "urllib3",
            "source": "pip",
            "version": "1.25.7"
        }
    ]
}
```
That concludes the basic demo. Now try executing without defining release number, or add requirements. Have fun!

This demo is heavily based on Phil Griffiths´ LinkedIn posting here: https://www.linkedin.com/pulse/ansible-execution-environments-phil-griffiths/
