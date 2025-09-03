# **Flexible Profiles for Ansible Roles**

This project provides a powerful and reusable Ansible pattern called "Flexible Profiles" (FP). The core idea is to create a generic engine role (fp) that other "consumer" roles can use to manage their configurations across different environments (e.g., operating systems, deployment stages like DEV/PROD) in a clean, layered, and maintainable way.

This approach centralizes environment-specific logic and variables, keeping the main role tasks clean and free of excessive conditional statements.

**For a detailed introduction to the concepts behind this pattern, please see the [Flexible Profiles for Ansible Roles: Easy Management of Variables and Handlers](https://www.linkedin.com/pulse/flexible-profiles-ansible-roles-easy-management-michael-knyazev-phd-iwglc).**

## **Core Concepts**

* **The Engine (fp role)**: This is the generic, reusable core. It is responsible for reading the active profile, loading variables from different layers, handling overrides, and exporting the final configuration variables for the consumer role to use.  
* **The Consumer Roles (myapp, myapp02, etc.)**: These are the roles that implement business logic. They consume the fp engine to abstract away their environment-specific configurations.  
* **Profiles**: A profile is a collection of settings for a specific target. This could represent a RedHat-9 operating system, a Windows-2022 server, or a deployment environment like UAT or PROD.

## **The Three-Layered Approach**

Flexible Profiles uses a three-layered system for defining variables. This provides a clear order of operations for default values, calculated variables, and just-in-time expressions. A consumer role can choose to use any or all of these layers by providing a path to the corresponding variable directory or by skipping a layer by setting its path to "skip".

### **Layer 1: Defaults (fp\_defaults)**

This layer provides the most basic, static, overridable default values for a profile. These are plain key-value pairs.

* **Example**: roles/myapp/vars/fp/fp\_defaults/RedHat-9.yml  

```yaml
  _fp_exists: true  
  service_name: "httpd"  
  config_dir: "/etc/httpd"
```

### **Layer 2: Instant Expressions (fp\_instant\_expressions)**

This layer contains variables that are calculated immediately ("eagerly") when the fp role runs. They are useful for building variables that depend on values from the defaults layer.

* **Example**: roles/myapp03/vars/fp/fp\_instant\_expressions/WindowsProd.yml  

```yaml
  _fp_exists: true  
  # This expression builds upon 'myapp03_api_server' from the defaults layer.  
  api_endpoint: "https://{{ myapp03_api_server }}/v1/data"
```

### **Layer 3: Deferred Expressions (fp\_deferred\_expressions)**

This layer contains variables whose values are stored as unrendered Jinja2 templates. They are evaluated "lazily" every time they are accessed in a task. This is extremely powerful for values that depend on runtime facts or variables that may be defined or changed *after* the fp role has already run.

* **Example**: roles/myapp04/vars/fp/fp\_deferred\_expressions/PROD.yml  

```yaml
  _fp_exists: true  
  # The value of 'myapp04_app_version' is not known when this is loaded.  
  # It will be resolved later, using the final, overridden value of the variable.  
  welcome_message: "Welcome to PROD environment. App version is {{ myapp04_app_version }}."
```

## **How to Use**

To make a role a "consumer" of the FP engine, you include the fp role and provide it with a set of specific inputs.

### **1\. Include the fp Role**

Typically done in an init\_fp.yml file which is then included by the consumer's main.yml.

```yaml
# In roles/myapp/tasks/init_fp.yml  
- name: MYAPP | init_fp.yml | Load Flexible Profiles for myapp  
  ansible.builtin.include_role:  
    name: fp  
  vars:  
    # ... FP inputs go here ...
```

### **2\. Provide Inputs**

The fp role is configured using the following variables passed to include\_role:

* fp\_consumer\_role\_name (string, **required**): The name of your role (e.g., "myapp"). Used for internal namespacing.  
* fp\_export\_with\_consumer\_role\_prefix (boolean, optional, default: false): If true, all exported variables will be prefixed with the fp\_consumer\_role\_name (e.g., service\_name becomes myapp\_service\_name).  
  * **Important**: When this is set to true, any variable overrides (e.g., in playbook vars: or inventory) and any variables used within expression files **must also use the prefix**. For example, in an instant expression, you must use {{ myapp\_api\_server }} not {{ api\_server }}.  
* fp\_active\_profile (string, **required**): The name of the profile to activate (e.g., "PROD"). This is often determined dynamically using Ansible facts or other input variables.  
* fp\_profile\_names (list, **required**): A list of all valid profile names for this role. The fp role will fail if fp\_active\_profile is not in this list.  
* fp\_defaults\_path, fp\_instant\_expressions\_path, fp\_deferred\_expressions\_path (string, **required**): The absolute paths to the three variable layer directories. Set a path to "skip" to disable that layer.

### **3\. Accessing Variables**

* **Default and Instant Variables**: Access them directly in your tasks using their prefixed (or unprefixed) names (e.g., {{ myapp\_service\_name }}).  
* **Deferred Variables**: These are stored inside a special dictionary named {{ fp\_consumer\_role\_name }}\_fp\_deferred. To render the expression, you access the key within that dictionary (e.g., {{ myapp\_fp\_deferred.welcome\_message }}).

## **Examples & Use Cases**

This repository includes several examples, each demonstrating a different feature.

### **test\_myapp.yml: Core Features & OS Detection**

This example demonstrates fact-based profile selection, prefixed variables, and profile-specific handlers.

* **Profile Selection**: Automatically switches between "RedHat-9" and "Windows-2022" based on the ansible\_os\_family fact. It also includes a **fallback profile** that is used if the OS family does not match, preventing the role from failing on unknown systems.  
* **Features**: Uses all three variable layers and shows how a notify action can trigger a handler that is specific to the active profile.  
* **Command**:  

```bash
ansible-playbook test_myapp.yml
```

### **test\_myapp02a.yml & test\_myapp02b.yml: Boolean Toggles & Unprefixed Vars**

This example shows how to switch profiles based on a simple boolean flag and exports variables without a prefix.

* **Profile Selection**: Switches between "RHEL" and "RHEL\_legacy" based on the rhel\_legacy\_mode boolean.  
* **Features**: Skips the expression layers. It also highlights an important consideration: when not using prefixes, variables from one profile can persist in Ansible's fact cache.  
* **Command**: These tests must be run as separate commands.  

```bash
# Test the modern profile  
ansible-playbook test_myapp02a.yml

# Test the legacy profile  
ansible-playbook test_myapp02b.yml
```

### **test\_myapp03a.yml & test\_myapp03b.yml: Dev vs. Prod Environments**

This example demonstrates a common Dev vs. Prod setup using the instant expressions layer.

* **Profile Selection**: Switches between "WindowsProd" and "WindowsDev" based on the non\_prod boolean.  
* **Features**: Uses instant expressions to dynamically construct an api\_endpoint URL based on the environment.  
* **Command**:

```bash
# Test the production profile  
ansible-playbook test_myapp03a.yml

# Test the development profile  
ansible-playbook test_myapp03b.yml
```

### **test\_myapp04a.yml & test\_myapp04b.yml: Advanced Precedence**

This advanced example demonstrates the full variable override precedence chain.

* **Variable Precedence Order** (from lowest to highest):  
  1. **Profile Default**: The base value from the active profile's fp\_defaults file.  
  2. **Inventory**: A value defined in an inventory file.  
  3. **Playbook vars**: A value defined in the vars: section of the playbook.  
  4. **Role vars**: A value passed directly to the include\_role task's vars: section (highest precedence).  
* **Features**: Shows how a variable like api\_key or db\_host can be defined in a profile but overridden at multiple higher levels for ultimate flexibility.  
* **Command**: These tests **must** be run with the \-i inventory.yml flag.  

```bash
# Test UAT profile with playbook-level override  
ansible-playbook -i inventory.yml test_myapp04a.yml

# Test PROD profile with playbook and role-level overrides  
ansible-playbook -i inventory.yml test_myapp04b.yml
```

## **Testing with Molecule**

This project uses **Molecule** for isolated, container-based testing of the fp role's internal logic. This ensures the core engine is robust and predictable. The test scenarios are located in roles/fp/molecule/ and cover different aspects of the role's functionality:

* defaults\_and\_prefixing: Tests basic variable loading, with and without prefixes.  
* deferred\_and\_precedence: Verifies that variable override precedence works correctly and that deferred expressions use the final, highest-precedence values.  
* invalid\_input\_validation: Ensures the role fails predictably and with clear error messages when given bad data.  
* layering\_and\_combination: A dedicated test to ensure all three variable layers interact correctly with each other.
* test\_ansible\_handlers: Tests the integration with Ansible handlers, ensuring that profile-specific handlers are correctly triggered and that profile variables are available within their scope.
* unprefixed\_variable\_clash: An explicit test to confirm the variable caching behavior when not using prefixes.

### **Running Molecule Tests**

To run a specific test scenario, navigate to the fp role directory and use the molecule command.

```bash
# Navigate to the role directory  
cd roles/fp

# Run the full test sequence for the 'invalid_input_validation' scenario  
molecule test -s invalid_input_validation
```

## **Linting and Quality**

This project uses ansible-lint to maintain code quality and adherence to best practices.

### **.ansible-lint**

The .ansible-lint file is used to explicitly suppress certain linting warnings that are not applicable to this project's design. For example, the var-naming\[no-role-prefix\] rule is ignored. This is intentional, as profile variables are designed to be generic within their files before the fp role optionally applies a prefix.

### **Running Ansible Lint**

To run the linter on the entire project, execute the following command from the project's root directory:

```bash
ansible-lint
```
