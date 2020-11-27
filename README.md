# Ansible Molecule Testing
Example for Ansible role testing with Molecule, Testinfra and Linters to do Unit, integration and functional testing.

## Overview
- Ansible testing spectrum: This repository contains details on how to test an Ansible role using over a number of aspects,
    - yamllint
    - ansible-playbook --syntax-check
    - ansible-lint
    - molecule test
    - ansible-playbook --check
    - Parallel infrastructure
- Most of the Ansible role testing functionality is converged around the `Molecule` tool.
- `Molecule` is designed to automate all parts of Ansible role testing.

### Ansible Testing Spectrum
##### 1. YAML Lint
-  A linter is a basic static code analyzer, that detect source code for programmatic and stylistic errors by applying a set of guidelines. `yamllint` recursively checks all the yaml files in the current directory.
    ```bash
    $ yamllint .                # Run linter on all files in directory recursively
    $ yamllint playbook.yml     # Run linter on a specific file
    ```
- Default YAML lint rules can be overridden by placing a `.yamllint` file in the current working directory with required overrides.
- [YAML Lint rules](https://yamllint.readthedocs.io/en/stable/rules.html)

##### 2. Ansible Syntax check
- A passive Ansible level code analysis to validate roles, variable, modules and other integration points without running the playbook. This check doesn't offer in-depth guaranteed insights if the playbook will run without failing. 
    ```bash
    $ ansible-playbook playbook.yml --syntax-check
    ```

##### 3. Ansible Lint
- Test Ansible tasks and playbooks to check if best practices are followed when developing Ansible playbooks. This is a set of guide lines that helps to avoid bad coding practices.
- This is helpful to encourage developers to follow a uniform set of rules to improve the quality of Ansible playbooks.
- Also works if playbooks include other playbooks, or tasks, or handlers or roles.
- Ansible lint can be run on a playbook or role specifically or auto-detect them in git repositories.
    ```bash
    $ ansible-lint playbook.yml         # Evaluates both the playbook and the referenced roles
    $ ansible-lint <role_name>          # Perform linting on a specific role
    $ ansible-lint <role_1> <role_2>    # Perform linting on a multiple roles
    ```
- Default Ansible lint rules can be overridden by placing a `.ansible-lint` file in the current working directory with required overrides.
- [Ansible Lint Documentation](https://ansible-lint.readthedocs.io/en/latest/)

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

##### Molecule configuration files
- **VERY VERY IMPORTANT** Include explanations about _Configuration blocks_ in `molecule.yml`, `converge.yml` and `verify.yml`.
- `molecule.yml`
    - Describes how Molecules will execute the tests.
    - Set dependant roles, platform and driver (e.g. Docker, vagrant etc.) and unit tests
- `converge.yml`
    - An Ansible playbook that specifies the Ansible role in interest.
    - Can include additional tasks (pre-tasks and post-tasks) required to test the playbook. e.g. Update package cache
- `verify.yml`
    - A playbook of tasks to verify and validate the environment after executing the ansible playbook.

## Step-by-Step instructions (Molecule)
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

#### Molecule Commands
_**NOTE:**_ All these commands should be run inside the role directory.
- **Testing role with default test matrix**
    - Better suited to test a role once it’s complete rather than in developing phase.
    ```bash
    $ molecule test
    ```
- **Run playbooks in molecule test environment**
    - Converge step allows to re-run the playbook on the test molecule instance without needing to create/destroy the instances every time. This allows to test the role rapidly test the role development workflow to allow for the fastest possible iterations.
    - _Can set a breakpoint using `fail` in the tasks._
        ```bash
        $ molecule converge
        $
        $ MOLECULE_DISTRO=debian10 molecule converge        # Execute tests on a different OS family
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
1. Importing roles from Ansible Galaxy
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
2. Patterns:
    - These modules help to validate certain parameters of the infrastructure is set as needed before proceeding forward in a playbook.
        - e.g. Port open, environment variable set, file exist etc.
    - `debug` module to display results and messages to verify certain checkpoints in the playbooks execution.
    - `fail` and `assert` module can be used to validate assertions under conditions, fail over conditions.
    - Running `patters.yml`,
        ```bash
        $ ansible-playbook patterns.yml 
        ```
3. Pre-tasks
    - Use `pre_tasks` in playbooks to set parameters or validate the environment of the host before running the primary roles/tasks.
    - These actions can be included in **prepare** stage of the `molecule.yml` configuration file.
    - CI Molecule testing is generally done and is recommended during development of Ansible roles and committing to repository during *push* and *pull request* actions.
4. Running playbooks on different OS distributions
    - [Jeff Geerling](https://ansible.jeffgeerling.com/) @geerlingguy maintains a great set of Docker images of different OS platforms ideal to run Molecule testing for Ansible playbooks.
    - Switching the OS one at a time
        - The OS platform running the molecule testing can be switched by passing an environment variable to define the OS distribution in `molecule.yml`.
            ```yaml
            ...
            platforms:
            - name: centos8
                image: geerlingguy/docker-${MOLECULE_DISTRO:-centos8}-ansible:latest
                command: ""
                volumes:
                - /sys/fs/cgroup:/sys/fs/cgroup:ro
                privileged: true
                pre_build_image: true
            ...
            ```
        - The OS distribution can be changed by passing int the value for `MOLECULE_DISTRO` environment variable.
            ```bash
            $ MOLECULE_DISTRO=debian10 molecule converge
            ```
    - Run testing on multiple OS distributions simultaneously
        - Include the list of OS distribution config settings in _**platform**_ block of the `molecule.yml` file. A common config scenario is shown below. Running `molecule converge` will create test instances for all distributions and execute the playbook simultaneously.
            ```yaml
            ...
            platforms:
            - name: centos8
                image: geerlingguy/docker-centos8-ansible:latest
                command: ""
                volumes:
                - /sys/fs/cgroup:/sys/fs/cgroup:ro
                privileged: true
                pre_build_image: true
            - name: ubuntu2004
                image: "geerlingguy/docker-ubuntu2004-ansible:latest"
                command: ""
                volumes:
                - /sys/fs/cgroup:/sys/fs/cgroup:ro
                privileged: true
                pre_build_image: true
            - name: debian10
                image: geerlingguy/docker-debian10-ansible:latest
                command: ""
                volumes:
                - /sys/fs/cgroup:/sys/fs/cgroup:ro
                privileged: true
                pre_build_image: true
            ...
            ```
5. Running verification and validation on hosts after running playbooks.
    - Use the `verify.yml` to define actions as Ansible tasks that can be use to validate the playbooks executed.
        - e.g. Required services running, application service traffic, configuration values set etc.
    - The default _**verifier**_ used by Molecule is _ansible_ (define tests as Ansible tasks). Molecule also support other verifiers _**Testinfra**_ to run unit testing on Ansible playbooks.


### Follow up:
- [Ansible lint for Github Actions](https://ansible-lint.readthedocs.io/en/latest/usage.html#ci-cd)

## References
- [Ansible Molecule](https://molecule.readthedocs.io/en/latest/)
- [Ansible Lint Documentation](https://ansible-lint.readthedocs.io/en/latest/)
- [yamllint documentation](https://yamllint.readthedocs.io/en/stable/index.html)
- [Ansible 101 - Episode 7 - Molecule Testing and Linting and Ansible Galaxy](https://youtu.be/FaXVZ60o8L8)
- [Ansible 101 - Episode 8 - Playbook testing with Molecule and GitHub Actions CI](https://youtu.be/CYghlf-6Opc)
- [Rapidly Build & Test Ansible Roles with Molecule + Docker](https://www.toptechskills.com/ansible-tutorials-courses/rapidly-build-test-ansible-roles-molecule-docker/)
- [Container Images for Ansible Testing](https://ansible.jeffgeerling.com/#container-images-testing)