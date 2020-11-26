# Ansible Molecule Testing
Example for Ansible role testing with Molecule, Testinfra and Linters to do Unit, integration and functional testing.

## Overview
- Ansible testing spectram: This repository contains details on how to test an Ansible role using over a number of aspects,
    - yamllint
    - ansible-playbook --syntax-check
    - ansible-lint
    - molecule test
    - ansible-playbook --check
    - Parallel infrastructure
- Most of the Ansible role testing functionality is converged around the `Molecule` tool.
- `Molecule` is designed to automate all parts of Ansible role testing.


##### Molecule Setup
- In this example, we are using `docker` driver as it closely resembles many practical scenarios of VMs of Cloud computer offerings.
- [Optional] Molecule testing can be executed inside a Python *Virtual environment*. When doing so, install the `molecule-docker` package.
    ```bash
    $ pip3 install -r requirements.txt 
    $ python3 -m pip install molecule-docker
    ```
- Pre-requisites:
    - Docker >=19.03 (`docker info`)
    - Python >=3.6 (`python --version`)
    - ansible >=2.8 (`ansible --version`)
    - molecule >=3.2 (`molecule --version`)
- Python package requirements are included in _requirements.txt_. Installing pre-requisites,
    ```bash
    $ pip3 install -r requirements.txt 
    ```

## Step-by-Step instructions
1. Initialize a new role with Molecule
    - Can create a new Ansible role using Molecule, _**OR**_
        ```bash
        $ molecule init role <role_name> --driver-name docker
        $ molecule init role my-new-role --driver-name docker
        ```
    - Initializing Molecule within an existing Ansible role,
        ```bash
        $ ansible-galaxy init <role_name>
        $ cd <role_name>
        $ molecule init scenario -r <role_name> --driver-name docker
        ```
- **VERY VERY IMPORTANT** Include explanations about _Configuration blocks_ in `molecule.yml`, `converge.yml` and `verify.yml`.

#### Molecule Commands
All these commands should be run inside the role directory.
- **Testing role with default test matrix**
    - Better suited to test a role once it’s complete rather than in developing phase.
    ```bash
    $ molecule test
    ```
- **Run playbooks in molecule test environment**
    - Converge step allows to re-run the playbook on the test molecule instance without needing to create/destroy the instances every time. This allows to test the role rapidly test the role development workflow to allow for the fastest possible iterations.
    ```bash
    $ molecule converge
    ```
- **Destroy molecule test instance**
    - Tears down an existing molecule test instance already running.
    ```bash
    $ molecule destroy
    ```
- **Run unit testing on playbook run**
    - Enables to run unit tests against molecule test instance.
    ```bash
    $ molecule verify
    ```

- **Log into molecule instance**
    - Manually inspect the molecule test instance during testing.
    ```bash
    $ molecule login
    ```

### Notes
- Importing roles from Ansible Galaxy
    - Can install Ansible Galaxy roles into a global location in the system such that the role can be shared by other Ansible playboooks.
        `ansible-galaxy install <galaxy_role_name>` OR
    - Install the Galaxy role in the same directory as the playbook. Create a `ansible.cfg` file in the root for the Ansible role.
        ```ini
        [default]
        nocows = True
        roles_path = ./roles
        ```
    - When `roles_path` is overridden, use a `requirements.yml` file to define the Ansible Galaxy roles to download.
        ```bash
        $ ansible-galaxy install -r requirements.yml
        ```
- Patterns:
    1. `debug` module to display results and messages to verify certain checkpoints in the playbooks execution.
    2. `fail` and `assert` module can be used to validate assertions under conditions, fail over conditions.

## References
- [Ansible Molecule](https://molecule.readthedocs.io/en/latest/)
- [Ansible 101 - Episode 7 - Molecule Testing and Linting and Ansible Galaxy](https://youtu.be/FaXVZ60o8L8)
- [Ansible 101 - Episode 8 - Playbook testing with Molecule and GitHub Actions CI](https://youtu.be/CYghlf-6Opc)
- [Rapidly Build & Test Ansible Roles with Molecule + Docker](https://www.toptechskills.com/ansible-tutorials-courses/rapidly-build-test-ansible-roles-molecule-docker/)
