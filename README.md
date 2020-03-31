Role Name
=========

Ansible role to install Ansible Tower and Seed Ansible Tower with some simple initial settings

Requirements
------------

A valid `license.json` file needs to be supplied using variable `tower_manage_licence` so that it can be uploaded to Tower once installed.
```yaml
tower_manage_licence: "{{ lookup('file','license.json') }}"
```

NOTE: Currently Organisations, Projects, Inventories and Job Templates are seeded, but the role can be easily expanded to handle all Tower objects.

Users can seed Tower objects by passing a variable `tower_objects` that contains a dictionary of objects. The standard `default/main.yml` file, contains an example set of high level objects that can be used as a guide. Each object has a distinct set of variables that it can consume, based upon the appropriate Ansible module employed. For example, the `organisation` object can take `name` and `description` as valid variables. Each object has been documented within the example `tower_objects`, use it as a guide for your seeding activities.

An example VARS file has been included to demostrate how to provide the objects.

The role heavily utilises the existing `tower_*` Ansible modules, which in turn leverage the `tower-cli`. This role installs `tower-cli` using the `pip` ansible module, into a Python Virtualenv on the target virtuaalenv paths .

The role installs all pre-requisites on the target (eg `python-virtualenv` and `python-setuptools`) to complete the above tasks, but it does require access to the appropriate RHEL/CentOS channels and access to PyPi.


There are no other strict Role dependencies.

Role Variables
--------------

Available variables are listed below, along with default values defined (see defaults/main.yml)

```
    # Working location for installation files
    tower_manage_working_location: "/root/"

    # Ansible Group name for master tower systems
    tower_manage_platform_group_name: "tower"

    # Default Passwords used within Tower Deployment
    tower_manage_admin_password: "password"
    tower_manage_pg_password: "password"
    tower_manage_rabbitmq_password: "password"

    # Tower Installer Version and Location
    tower_manage_tower_release_version: 3.5.2-1
    tower_manage_tower_releases_url: https://releases.ansible.com/ansible-tower/setup-bundle/
    tower_manage_tower_setup_file: ansible-tower-setup-{{ tower_manage_tower_release_version }}.tar.gz

    # LDAP Configuration
    tower_manage_ldap_server_uri: ldap://<ad_server>
    tower_manage_ldap_bind_dn: <bind_dn>
    tower_manage_ldap_start_tls: "false"
    tower_manage_ldap_user_search: \[ "DC=example,DC=com", "SCOPE_SUBTREE", "(sAMAccountName=%(user)s)" \]
    tower_manage_ldap_user_dn_template: null
    tower_manage_ldap_user_attr_map: '{ "first_name": "givenName", "last_name": "sn", "email": "mail" }'
    tower_manage_ldap_group_search: '[ "OU=Groups,OU=Administration,DC=example,DC=com", "SCOPE_SUBTREE", "(objectClass=group)" ]'
    tower_manage_ldap_group_type: MemberDNGroupType
    tower_manage_ldap_group_type_params: '{ "name_attr": "cn", "member_attr": "member" }'
    tower_manage_ldap_require_group: CN=ROL_BUILD_Administrator,OU=example - Protected,OU=Role Groups,OU=Groups,OU=Administration,DC=example,DC=com
    tower_manage_ldap_deny_group: null
    tower_manage_ldap_user_flags_by_group: '{ "is_superuser": [ "CN=ROL_BUILD_Administrator,OU=example - Protected,OU=Role Groups,OU=Groups,OU=Administration,DC=example,DC=com" ] }'
    tower_manage_ldap_organization_map: "{}"
    tower_manage_ldap_team_map: "{}"
```

Example Playbook
----------------

Role Variables
--------------

Available variables are listed below, along with default values (see defaults/main.yml):

```
# URL of the Tower API instance/host
tower_manage_server: https://localhost

# Should certs be validated? Set false if using self-signed certs
tower_managevalidate_certs: true

# Dictionary of Tower Objects to seed
tower_objects:
  organisations:
    - name: tom
      description: Test project
      projects:
        - name: Prefect
          description: "Ansible Playground Manager"
          scm_url: "https://github.com/danhawker/prefect.git"
          scm_type: git
          scm_credential: github
          custom_virtualenv: "{{ tower_manage_tower_default_virtualenv_path }}/mon"
      inventories:
        - name: AWS
          description: AWS Test
      inventory_source:
        - name: AWS test sources
          description: AWS test sources
          credential:
          source:
          source_project:
          instance_filters:
      job_templates:
        - name: "Ping"
          job_type: run
          playbook: "playbooks/ping.yml"
          project: "Prefect"
          inventory: "AWS"
          credential: "Local"
          survey_enabled: yes
          survey_spec: "{{ lookup('file', 'ping_survey.json') }}"
          ask_credential: yes
          ask_extra_vars: yes
          ask_skip_tags: yes
          verbosity: 2
      workflow_templates:
        - name: Workflow Template
          description: My very first Workflow Template
          organization: tom
          schema: "{{ lookup('file', 'my_workflow.json') }}"
```


Example Playbook
----------------

The following playbook and accompanying vars file containing the defined seed objects can be invoked in the following manner.

```
$ ansible-playbook playbook.yml -e @tower_vars.yml tower
```

```
---
# Playbook to install Ansible Tower as a single node

- name: Setup Ansible Tower
  hosts: awx
  become: true
  vars:
    tower_manage_tower_releases_url: https://releases.ansible.com/ansible-tower/setup-bundle
    tower_manage_tower_release_version: bundle-3.6.3-1.tar.gz
  tasks:
    - include_role:
        name: ansible-deploy-awx
        tasks_from: "{{awx_tasks}}.yml"
      loop:      # Include  specific task file
        - tower_install
        - tower_license
      loop_control:
        loop_var: awx_tasks
```

```
---
# Playbook to install Ansible Tower as a cluster

- name: Setup Ansible Tower
  hosts: awx
  become: true
  vars:
    tower_hosts:
      - "clusternode[1:3].example.com"
    tower_database: "dbnode.example.com"
    tower_database_port: "5432"
  tasks:
    - include_role:
        name: ansible-deploy-awx
        tasks_from: "{{awx_tasks}}.yml"
      loop:      # Include  specific task file
        - tower_install
        - tower_license
      loop_control:
        loop_var: awx_tasks
```

```
---
# Playbook to seed Ansible Tower

- name: Setup Ansible Tower
  hosts: awx
  become: true
  vars:
    tower_manage_tower_custom_virtualenv_path_per_team_jobs:
      demo:
        venv_cmd: "/usr/bin/pyvenv"
        packages:
          - ansible
          - setuptools
          - psutil
          - awscli
          - boto
          - boto3
    tower_objects:
      organisations:
        - name: working demo
          description: working demo project
          credential_type:
            - name: infoblox
              inputs:
                fields:
                  - id: username
                    type: string
                    label: Username
                  - id: host
                    type: string
                    label: Hostname
                  - id: password
                    type: string
                    label: Password
                    secret: true
                required:
                  - username
                  - password
              injectors: "{{ lookup('file', '{{ var_location }}/infoblox_extra_vars.json') }}"
          credential:
            - name: "Infoblox Prod"
              kind: infoblox
              inputs:
                host: ipam.example.com
                username: user
                password: password
            - name: "Ansible vCenter"
              kind: vmware
              host: ukthvmvcal01.example.com
              username: user
              password: password
          projects:
            - name: working
              description: "Ansible Playground Manager"
              scm_url: "https://github.com/ansible/ansible-tower-samples.git"
              scm_type: git
              scm_update_on_launch: yes
              custom_virtualenv: "{{ tower_manage_tower_default_virtualenv_path }}/demo"
          inventories:
            - name: working
              description: working demo Test
          inventory_source:
            - name: working demo sources
              description: working demo sources
              credential:
              source:
              source_project:
              instance_filters:
          job_templates:
            - name: "demo"
              job_type: run
              playbook: "hello_world.yml"
              project: working
              inventory: "working"
              credential: "Local"    # That must be created before use or use already created credential
              survey_enabled: yes
              survey_spec: "{{ lookup('file', 'tower_setup_survey.json') }}"
              verbosity: 2
            - name: "Regular Job"
              job_type: run
              playbook: "playbooks/regular.yml"
              project: "working"
              inventory: "working"
              credential: "Local"
              schedule:
                name: "regular scheduled job"
                startdatetime: 20200101T010000 # defaults to now
                frequency: 'daily' # {hourly, daily, weekly, monthly}
                interval: 1 # Defaults to 1
          job_template_credential:
            - job_template: "demo"
              credential: "cred1"
            - job_template: "demo"
              credential: "cred2"
          workflow_templates:
            - name: demo Template
              description: My very first Workflow Template
              organization: tom
              schema:
                - job_template: "Demo Job Template"
                  success_nodes:
                  - job_template: "Demo Job Template"
  tasks:
    - include_role:
        name: ansible-deploy-awx
        tasks_from: "{{awx_tasks}}.yml"
      loop:      # Include  specific task file
        - tower_create_venv
        - tower_orgs
        - tower_inventories
        - tower_projects
        - tower_job
        - tower_job_credential
        - tower_workflow
      loop_control:
        loop_var: awx_tasks
```
```
---
- name: Configure Tower LDAP
  hosts: localhost

  vars:
    tower_manage_server: https://tower.example.com
    tower_manage_validate_certs: false
    tower_manage_ldap_server_uri: ldap://ad_server.example.com
    tower_manage_ldap_bind_dn: CN=tower_ad_account,OU=Service Accounts,DC=example,DC=com
    tower_manage_ldap_start_tls: "false"
    tower_manage_ldap_user_search: \[ "DC=example,DC=com", "SCOPE_SUBTREE", "(sAMAccountName=%(user)s)" \]
    tower_manage_ldap_user_dn_template: null
    tower_manage_ldap_user_attr_map: '{ "first_name": "givenName", "last_name": "sn", "email": "mail" }'
    tower_manage_ldap_group_search: '[ "OU=Groups,OU=Administration,DC=example,DC=com", "SCOPE_SUBTREE", "(objectClass=group)" ]'
    tower_manage_ldap_group_type: MemberDNGroupType
    tower_manage_ldap_group_type_params: '{ "name_attr": "cn", "member_attr": "member" }'
    tower_manage_ldap_require_group: CN=ROL_BUILD_Administrator,OU=example - Protected,OU=Role Groups,OU=Groups,OU=Administration,DC=example,DC=com
    tower_manage_ldap_deny_group: null
    tower_manage_ldap_user_flags_by_group: '{ "is_superuser": [ "CN=ROL_BUILD_Administrator,OU=example - Protected,OU=Role Groups,OU=Groups,OU=Administration,DC=example,DC=com" ] }'
    tower_manage_ldap_organization_map: "{}"
    tower_manage_ldap_team_map: "{}"

  tasks:
    - name: "Load vars"
      include_vars: "../vars/tower.yml"

    - include_role:
        name: ansible-tower-manage
        tasks_from: "{{awx_tasks}}.yml"
      loop:      # Include  specific task file
        - tower_ldap
      loop_control:
        loop_var: awx_tasks
```

License
-------

MIT


Author Information
------------------
Tom Page