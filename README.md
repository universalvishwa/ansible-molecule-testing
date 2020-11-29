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
- Majority of the Ansible role testing functionality can be converged within _Molecule_.

## TL;DR Creating Ansible roles with Molecule Testing
Repo Link: [ansible-molecule-testing-tldr](https://github.com/universalvishwa/ansible-molecule-testing-tldr)
1. Create Virtual environment and install Python dependencies
    ```bash
    $ python3 -m venv venv
    $ source venv/bin/activate
    $ pip3 install -r requirements.txt 
    ```
2. Create Ansible role using Molecule 
    ```bash
    $ molecule init role <role_name> --driver-name docker
    ```
3. Update the _metadata_ for Ansible role
    - Complete the `meta/main.yml` with appropriate details. **OR**
    - Delete this folder completely to avoid linting errors.
4. Configure `molecule.yml` configuration file
    - _**dependency**_: Add dependent collections/roles to `molecule/default/requirements.yml`
    - _**driver**_: Set the driver for infrastructure to `docker`
    - _**lint**_: Configure linter script (`yamllint`, `ansible-lint`) and place the rule override files in root path of role.
    - _**platforms**_: Set the Dockerized OS distribution platform(s) to run Molecule test
    - _**verifier**_: Set verification provider use to validate the test environment. `ansible`(default), `testinfra`
    - Example of a `molecule.yml` file,
        ```yaml
        ---
        dependency:
          name: galaxy
          options:
            ignore-certs: true
            requirements-file: requirements.yml
            role-file: requirements.yml
        driver:
          name: docker
        lint: |
          set -e
          yamllint .
          ansible-lint
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
        provisioner:
          name: ansible
        verifier:
          name: ansible
        ```
4. Configure `converge.yml`
    - Add `pre_tasks` and `vars` (static variables) to converge playbook.
5. Proceed with writing code for the Ansible role
    - Populate `tasks/`, `handlers/`, `vars/`, `defaults/` etc...
6. Create Molecule Test instance and run playbook
    ```bash
    $ cd <role_name>
    $ molecule converge
    ```
7. Add Ansible Tasks, playbooks or Testinfra script to `verify.yml` to validate the Molecule test instance.
    - This can be individual tasks or a separate playbook. **OR**
    - Testinfra script
8. Run the validation tests on the Molecule instance to verify the code
    ```bash
    $ cd <role_name>
    $ molecule verify
    ```
9. Follow this cycle process iteratively by adding new code to the role followed by running `molecule converge` and `molecule verify` until the code for the role is code complete.
    - If there is a need to recreate the Molecule testing during development phase use `molecule destroy` terminate test instance and start from a clean slate.
        ```bash
        $ molecule converge
        $ molecule verify
        $ molecule destroy
        ```
    - If required to log into the Molecule testing instance during development use `molecule login`,
        ```bash
        $ molecule login --host <platform_name>
        ```
    - Ansible modules `fail`, `debug` and `assert` can be use to extract certain information or set breakpoints to troubleshoot errors.
10. After role development is complete run a full Molecule test life cycle.
    ```bash
    $ molecule test
    ```
11. Molecule testing with GitHub Actions (CI)
    - Create the workflow to trigger during *push* and *pull requests* to main branches.
    - Example of YAML configuration file for Github Action workflow CI testing `.github/workflow/ci.yml`,
        ```yaml
        ---
        name: CI
        'on':
          pull_request:
          push:
            branches:
              - master

        jobs:
          Ansible-role-test:
            name: Molecule
            runs-on: ubuntu-latest
            strategy:
              matrix:
                distro:
                  - centos8
                  - ubuntu2004
                  - debian10
            steps:
              - name: Check out the codebase.
                uses: actions/checkout@v2

              - name: Set up Python 3.
                uses: actions/setup-python@v2
                with:
                python-version: '3.x'

              - name: Install test dependencies.
                run: pip3 install -r requirements.txt

              - name: Run Molecule tests.
                run: |
                  cd myrole
                  molecule test
                env:
                  PY_COLORS: '1'
                  ANSIBLE_FORCE_COLOR: '1'
                  MOLECULE_DISTRO: ${{ matrix.distro }}
        ```

### Ansible Testing Spectrum
#### 1. YAML Lint
-  A linter is a basic static code analyzer, that detect source code for programmatic and stylistic errors by applying a set of guidelines. `yamllint` recursively checks all the yaml files in the current directory.
    ```bash
    $ yamllint .                # Run linter on all files in directory recursively
    $ yamllint playbook.yml     # Run linter on a specific file
    ```
- Default [YAML lint rules](https://yamllint.readthedocs.io/en/stable/rules.html) can be overridden by placing a `.yamllint` file in the current working directory with required overrides.

#### 2. Ansible Syntax check
- A passive Ansible level code analysis to validate roles, variables, modules and other integration points without running the playbook. This check doesn't offer in-depth guaranteed insights on the ability to run the playbook without failing. 
    ```bash
    $ ansible-playbook playbook.yml --syntax-check
    ```

#### 3. Ansible Lint
- Test Ansible tasks and playbooks to check if best practices are followed when developing Ansible playbooks. This is a set of guide lines that helps to avoid bad coding practices.
- This is helpful to encourage developers to follow a uniform set of rules to improve code quality.
- Also works if playbooks include other playbooks, or tasks, or handlers or roles.
- Ansible lint can be run on a playbook or role specifically or auto-detect them in git repositories.
    ```bash
    $ ansible-lint playbook.yml         # Evaluates both the playbook and the referenced roles
    $ ansible-lint <role_name>          # Perform linting on a specific role
    $ ansible-lint <role_1> <role_2>    # Perform linting on a multiple roles
    ```
- Default Ansible lint rules can be overridden by placing a `.ansible-lint` file in the current working directory with required overrides.

#### 4. Molecule
- Molecule supports integrating with CI environments to run unit test on Ansible playbooks when committing to repositories during *push* and *pull requests* actions.
- _Molecule_ is designed to automate all parts of Ansible role testing.


## Molecule Deep-dive
#### 1. Molecule configuration files
- **`molecule.yml`**
    - Describes how Molecule will execute the tests.
    - Set dependant roles, platform, linters, driver (e.g. Docker, vagrant etc.) and unit tests
- **`converge.yml`**
    - An Ansible playbook that specifies the Ansible role in interest.
    - Can include additional tasks (pre-tasks and post-tasks) required to test the playbook. e.g. Update package cache
- **`verify.yml`**
    - A playbook, tasks or _testinfra_ script to verify and validate the environment after executing the ansible playbook.

#### 2. Setup Molecule environment
- This example demonstrate using `docker` driver as it closely resembles many practical scenarios of VMs of cloud compute offerings.
- [Optional] Molecule testing can be executed inside a Python *Virtual environment*.
    ```bash
    $ python3 -m venv venv
    $ source venv/bin/activate
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

#### 3. Initialize role with Molecule
This creates the Molecule configuration files structure
1. Create a new Ansible role using Molecule, _**OR**_
    ```bash
    $ molecule init role <role_name> --driver-name docker
    $ molecule init role myrole --driver-name docker
    ```
2. Initializing Molecule within an existing Ansible role,
    ```bash
    $ ansible-galaxy init <role_name>           # Create Ansible role structure ONLY
    $ cd <role_name>
    $ molecule init scenario -r <role_name> --driver-name docker    # Add molecule structure to role
    ```

### Molecule Commands
_**NOTE:**_ All these commands _**must**_ be run inside the role directory.
- **Run full Testing lifecyle on the role**
    - Test role with default test matrix
    - Better suited to test a role once it’s development complete rather than in developing phase.
        ```bash
        $ molecule test
        ```
- **Create molecule test instance**
    - Create Molecule test instance without running the Ansible role.
        ```bash
        $ molecule create
        ```
- **Run playbooks in molecule test environment**
    - Converge step allows to re-run the playbook on the test molecule instance without needing to create/destroy the instances every time. This allows to test the role rapidly during the role development workflow to allowing fastest possible iterations.
        ```bash
        $ molecule converge
        $ MOLECULE_DISTRO=debian10 molecule converge       # Execute tests on a different OS family
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
        $ molecule login                          # Single platform
        $ molecule login --host <platform_name>   # Multi platform
        $ molecule login --host centos8
        ```
- **Run linters on Ansible role code**
    - Supports passing multiple linters to be applied on the code. e.g `yamllint`, `ansible-lint`
        ```bash
        $ molecule lint
        ```

### Important Notes
1. **Importing roles from Ansible Galaxy**
    - Install Ansible Galaxy roles into a global (shared) location in the system. _**OR**_
        ```bash
        $ ansible-galaxy install <galaxy_role_name>
        ```
    - To install Ansible Galaxy roles in the same directory as the playbook, create a `ansible.cfg` file in the parent directory of the Ansible role with the following.
        ```ini
        [defaults]
        roles_path = ./roles
        ```
    - When `roles_path` is overridden, use a `requirements.yml` file to define the Ansible Galaxy roles to download. Install Galaxy roles by executing,
        ```bash
        $ ansible-galaxy install -r requirements.yml
        ```
2. **Testing Patterns and Tips**
    - These modules help to validate certain parameters of the infrastructure is set as needed before proceeding forward in a playbook.
        - e.g. Port open, environment variable set, file exist etc.
    - `debug` module can display results and messages to verify certain checkpoints in the playbooks execution.
    - `fail` and `assert` module can be used to validate assertions under conditions or fail on conditions.
        - `fail` can be used to set a breakpoints in tasks.
    - Running `patters.yml`,
        ```bash
        $ ansible-playbook patterns.yml 
        ```
3. **Pre-tasks**
    - Use `pre_tasks` in playbooks to set parameters or validate the environment of the host before running the primary roles/tasks.
    - These actions can be included in **prepare** stage of the `molecule.yml` configuration file.
4. **Testing roles on different OS distributions**
    - [Jeff Geerling](https://ansible.jeffgeerling.com/) ([@geerlingguy](https://github.com/geerlingguy)) maintains a great set of Docker images of different OS platforms ideal to run Molecule testing for Ansible roles.
    1. _Switching the OS one at a time_
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
            $ MOLECULE_DISTRO=ubuntu2004 molecule converge
            $ MOLECULE_DISTRO=debian10 molecule converge
            ```
    2. _Run testing on multiple OS distributions simultaneously_
        - Include the list of OS distribution settings in _**platforms**_ block of the `molecule.yml` file.
        - Running `molecule converge` will create test instances for all distributions and execute the playbook simultaneously.
        - A common config scenario is shown below.
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
5. **Verification and validation on hosts after running playbooks**
    - Use the `verify.yml` to define actions as Ansible tasks (or an entire playbook using `include_role`) that can be use to validate the playbooks executed.
        - e.g. Required services running, application service traffic, configuration values set etc.
    - The default _**verifier**_ used by Molecule is _ansible_ (i.e. define tests as Ansible tasks).
    - Molecule also support other verifier, _**Testinfra**_ to run unit testing on Ansible playbooks.
6. **Running linters on Ansible playbooks with Molecule**
    - Molecule allows running one or more linters as a part of the playbook testing through a script block.
    - This avoids the need to run Linting as part of CI pipeline separately.
    - Rule overrides for `.yamllint` and `.ansible-lint` should be included in the root path of the role.
    - Add the following block to run linters and fail if not up to requirements,
        ```yaml
        ...
        lint: |
          set -e
          yamllint .
          ansible-lint
        ...
        ```
7. **Setting Role dependencies with Ansible Galaxy**
    - If the Ansible role has any dependencies (role, collections), add a `molecule/default/requirements.yml` and Molecule will automatically run ansible-galaxy to download them on the test instance.
    - Set the **dependency** configuration block in `converge.yml` as follows,
        ```yaml
        ...
        dependency:
          name: galaxy
          options:
            ignore-certs: true
            requirements-file: requirements.yml
            role-file: requirements.yml
        ...
        ```
8. **Continuous integration (CI) Molecule testing with GitHub Actions**
    - Generally Molecule testing is integrated with CI testing action triggers during *push* and *pull requests* to main branches.
    - Morevover, Molecule testing is also executed during development of Ansible roles in addition to CI testing actions.
    - A build strategy/matrix can be used to run Molecule testing on multiple OS platforms with the use of _**platforms**_ config setting in `molecule.yml`.
    - Add colours to Python and Ansible outputs by setting the following environment variables.
        ```yaml
        ...
        env:
          PY_COLORS: '1'                          # Molecule colors
          ANSIBLE_FORCE_COLOR: '1'                # Ansible colors
          MOLECULE_DISTRO: ${{ matrix.distro }}   # Switch OS platform of testing
        ...
        ```

## References
- [Ansible Molecule](https://molecule.readthedocs.io/en/latest/)
- [Ansible Lint Documentation](https://ansible-lint.readthedocs.io/en/latest/)
- [yamllint documentation](https://yamllint.readthedocs.io/en/stable/index.html)
- [YAML Lint rules](https://yamllint.readthedocs.io/en/stable/rules.html)
- [Ansible 101 - Episode 7 - Molecule Testing and Linting and Ansible Galaxy](https://youtu.be/FaXVZ60o8L8)
- [Ansible 101 - Episode 8 - Playbook testing with Molecule and GitHub Actions CI](https://youtu.be/CYghlf-6Opc)
- [Testing your Ansible roles with Molecule](https://www.jeffgeerling.com/blog/2018/testing-your-ansible-roles-molecule)
- [Rapidly Build & Test Ansible Roles with Molecule + Docker](https://www.toptechskills.com/ansible-tutorials-courses/rapidly-build-test-ansible-roles-molecule-docker/)
- [Container Images for Ansible Testing](https://ansible.jeffgeerling.com/#container-images-testing)
- [Ansible Testing Using Molecule with Ansible as Verifier](https://www.adictosaltrabajo.com/2020/05/08/ansible-testing-using-molecule-with-ansible-as-verifier/)