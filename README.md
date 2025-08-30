# Flexible Profiles for Ansible Roles

This project demonstrates a powerful and reusable Ansible pattern called "Flexible Profiles" (FP). The core idea is to create a generic role (`FlexibleProfiles`) that can be consumed by other roles to manage their configurations across different environments (e.g., operating systems, deployment stages like DEV/PROD) in a clean, layered, and maintainable way.

This approach centralizes environment-specific logic and variables, keeping the main role tasks clean and free of excessive conditional statements.

## Core Components

* **`roles/FlexibleProfiles`**: This is the generic "engine" role. It is responsible for reading the active profile, loading variables from different layers, handling overrides, and exporting the final configuration variables for the consumer role to use.
* **Consumer Roles (`roles/myapp`, `roles/myapp02`, etc.)**: These are example roles that demonstrate how to use the `FlexibleProfiles` engine to manage their own specific configurations.

## How It Works: The Layered Approach

Flexible Profiles uses a three-layered system for defining variables. This provides a clear structure for default values, calculated variables, and just-in-time expressions.

1.  **Layer 1: Defaults (`fp_defaults`)**: Provides the most basic, overridable default values for a profile (e.g., a service name or default directory).
2.  **Layer 2: Instant Expressions (`fp_instant_expressions`)**: Contains variables that are calculated immediately ("eagerly") based on the values from the defaults layer.
3.  **Layer 3: Deferred Expressions (`fp_deferred_expressions`)**: Contains variables that are evaluated "lazily" every time they are used in a task. This is useful for values that depend on runtime facts.

A consumer role can choose to use any or all of these layers by providing a path to the corresponding variable directory or skipping a layer by setting its path to `"skip"`.

## Testing and Examples

This repository includes several test playbooks, each demonstrating a different use case and feature of the `FlexibleProfiles` role.

### `test_myapp.yml`

This is the primary example demonstrating the core features.

* **Profile Selection**: Automatically detects the profile based on Ansible facts (`ansible_os_family`).
* **Features**: Uses all three variable layers, demonstrates prefixed variables (`myapp_...`), and tests profile-specific handlers.
* **Command**:
    ```bash
    ansible-playbook test_myapp.yml
    ```

### `test_myapp02a.yml` & `test_myapp02b.yml`

This example demonstrates conditional profile selection based on a boolean flag.

* **Profile Selection**: Switches between `"RHEL"` and `"RHEL_legacy"` profiles based on the `rhel_legacy_mode` variable.
* **Features**: Shows how to disable the expression layers and how to export variables *without* a prefix.
* **Command**: These tests must be run as separate commands to prevent fact caching issues when using unprefixed variables.
    ```bash
    # Test the modern profile
    ansible-playbook test_myapp02a.yml

    # Test the legacy profile
    ansible-playbook test_myapp02b.yml
    ```

### `test_myapp03a.yml` & `test_myapp03b.yml`

This example demonstrates a common Dev vs. Prod environment setup.

* **Profile Selection**: Switches between `"WindowsProd"` and `"WindowsDev"` profiles based on the `non_prod` boolean variable.
* **Features**: Uses prefixed variables and demonstrates the power of the `instant expressions` layer to build environment-specific values like API endpoints.
* **Command**:
    ```bash
    # Test the production profile
    ansible-playbook test_myapp03a.yml

    # Test the development profile
    ansible-playbook test_myapp03b.yml
    ```

### `test_myapp04a.yml` & `test_myapp04b.yml`

This is an advanced example showcasing the full variable override precedence chain.

* **Profile Selection**: Switches between profiles based on the `target_env` variable (e.g., "UAT", "PROD").
* **Features**: Demonstrates how variables from the profile are overridden by values from `inventory.yml`, then by the playbook's `vars:` section, and finally by variables passed directly to the `include_role` task. It also tests conditional handlers.
* **Command**: These tests **must** be run with the `-i inventory.yml` flag to correctly test the inventory override feature.
    ```bash
    # Test the UAT profile with a playbook-level override
    ansible-playbook -i inventory.yml test_myapp04a.yml

    # Test the PROD profile with playbook and role-level overrides
    ansible-playbook -i inventory.yml test_myapp04b.yml
    ```
