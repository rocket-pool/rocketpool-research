# Config File Candidate Architectures

This file describes the candidate architectures for the underlying configuration files that are used to serialize the user's desired Smartnode configuration.


## Components

### Parameter Definitions 

Parameter definitions refer to the hierarchy of settings definitions themselves, their names / descriptions / default values, the contract addresses for each network, and so on. 

| ID | Description |
| - | - |
| PD1 | There is one file for *non-changeable* parameter definitions. |
| PD2 | The parameter definitions are no longer stored in a file, but are instead built directly into the Smartnode stack. |
| PD3 | The parameters are stored as a set of hierarchical files based on their location, instead of all in one monolithic file. |


### Settings / Values

These refer to the way that the user's settings for each parameter are stored.

| ID | Description |
| - | - |
| SV1 | There is one file for the values of each setting. Each setting specifies whether it was configured (or just uses the default value, as default scan change between versions), and what the current value is. |
| SV2 | The values are not stored directly in an explicit file. |
| SV3 | The values are stored in a set of hierarchical files based on the location of their respective parameters. |


### Docker-Compose Files

These describe how the YAML files used by `docker-compose` are stored, modified, and/or generated.

| ID | Description |
| - | - |
| DC1 | We store one `docker-compose` file for all of the containers. This is pregenerated, as it is today, and included with the installer. This file uses environment variables to substitute all of the parameter values. |
| DC2 | We store many `docker-compose` files, one per container. They are all pregenerated. They use environment variables to substitute all of the parameter values. |
| DC3 | We dynamically generate one `docker-compose` file at the end of `service config`, with all of the settings baked in based on the specified values. |
| DC4 | We dynamically generate many `docker-compose` files, one per container, at the end of `service config`. All of their settings are baked in based on the specified values. |


## Architectures

Each architecture is a combination of the above options, where they're compatible.

*(Note: Combinations with SV2 and DC1 / DC2 are not possible.)*


### CFG1

**Components: PD1; SV1; DC1**

- There is one file for parameter definitions (`parameters.yml`) and one that stores the specified values (`settings.yml`).
- There is one standard container definition file (`docker-compose.yml`).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
- `service start` will use `settings.yml` to define environment variables for each parameter with the corresponding value, and call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG2

**Components: PD1; SV1; DC2**

- There is one file for parameter definitions (`parameters.yml`) and one that stores the specified values (`settings.yml`).
- There are multiple container definition files, one per container (e.g. `docker-compose-eth1.yml`, `docker-compose-eth2.yml`...).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
- `service start` will use `settings.yml` to define environment variables for each parameter with the corresponding value, and call `docker-compose` on each container definition file to create / update the containers.


### CFG3

**Components: PD1; SV1; DC3**

- There is one file for parameter definitions (`parameters.yml`) and one that stores the specified values (`settings.yml`).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
  - It will also generate a single container definition file (`docker-compose.yml`) with all of the appropriate values based on the contents of `settings.yml`.
  - If this file already exists, it will be overwritten.
- `service start` will call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG4

**Components: PD1; SV1; DC4**

- There is one file for parameter definitions (`parameters.yml`) and one that stores the specified values (`settings.yml`).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
  - It will also generate multiple container definition files based on the settings (the exact files generated will depend on the settings) with all of the appropriate values based on the contents of `settings.yml`.
  - If any of the files already exist, they will be overwritten.
  - Any old files that were *not* overwritten during this process will be deleted.
- `service start` will call `docker-compose` on the container definition files to create / update the containers.
  - It may need to call it with the `--remove-orphans` to get rid of containers that no longer exist due to configuration changes.
  

### CFG5

**Components: PD1; SV2; DC3**

- There is one file for parameter definitions (`parameters.yml`).
- There will be a single container definition file (`docker-compose.yml`).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it by parsing the contents of `docker-compose.yml` to find the appropriate values.
- Upon saving, the Smartnode will regenerate the container definition file with all of the appropriate values based on the user's settings choices.
  - If this file already exists, it will be overwritten.
- `service start` will call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG6

**Components: PD1; SV2; DC4**

- There is one file for parameter definitions (`parameters.yml`).
- There will be multiple container definition files (e.g. `docker-compose-eth1.yml`, `docker-compose-eth2.yml`...).
- The Smartnode will read `parameters.yml` during `service config` to dynamically create the configuration UI.
  - It will pre-populate it by parsing the contents of the container definition files to find the appropriate values.
- Upon saving, the Smartnode will regenerate the container definition files based on the settings (the exact files generated will depend on the settings) with all of the appropriate values based on the user's settings choices.
  - If any of the files already exist, they will be overwritten.
  - Any old files that were *not* overwritten during this process will be deleted.
- `service start` will call `docker-compose` on the container definition files to create / update the containers.
  - It may need to call it with the `--remove-orphans` to get rid of containers that no longer exist due to configuration changes.


### CFG7

**Components: PD2; SV1; DC1**

- There is one file that stores the specified values (`settings.yml`).
- There is one standard container definition file (`docker-compose.yml`).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
- `service start` will use `settings.yml` to define environment variables for each parameter with the corresponding value, and call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG8

**Components: PD2; SV1; DC2**

- There is one file that stores the specified values (`settings.yml`).
- There are multiple container definition files, one per container (e.g. `docker-compose-eth1.yml`, `docker-compose-eth2.yml`...).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
- `service start` will use `settings.yml` to define environment variables for each parameter with the corresponding value, and call `docker-compose` on each container definition file to create / update the containers.


### CFG9

**Components: PD2; SV1; DC3**

- There is one file that stores the specified values (`settings.yml`).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
  - It will also generate a single container definition file (`docker-compose.yml`) with all of the appropriate values based on the contents of `settings.yml`.
  - If this file already exists, it will be overwritten.
- `service start` will call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG10

**Components: PD2; SV1; DC4**

- There is one file that stores the specified values (`settings.yml`).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it with the existing settings in `settings.yml`.
- Upon saving, the Smartnode will update `settings.yml`.
  - It will also generate multiple container definition files based on the settings (the exact files generated will depend on the settings) with all of the appropriate values based on the contents of `settings.yml`.
  - If any of the files already exist, they will be overwritten.
  - Any old files that were *not* overwritten during this process will be deleted.
- `service start` will call `docker-compose` on the container definition files to create / update the containers.
  - It may need to call it with the `--remove-orphans` to get rid of containers that no longer exist due to configuration changes.
  

### CFG11

**Components: PD2; SV2; DC3**

- There are no predefined files.
- There will be a single container definition file (`docker-compose.yml`).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it by parsing the contents of `docker-compose.yml` to find the appropriate values.
- Upon saving, the Smartnode will regenerate the container definition file with all of the appropriate values based on the user's settings choices.
  - If this file already exists, it will be overwritten.
- `service start` will call `docker-compose` on `docker-compose.yml` to create / update the containers.


### CFG12

**Components: PD2; SV2; DC4**

- There are no predefined files.
- There will be multiple container definition files (e.g. `docker-compose-eth1.yml`, `docker-compose-eth2.yml`...).
- The Smartnode's UI during `service config` will be baked into the CLI.
  - It will pre-populate it by parsing the contents of the container definition files to find the appropriate values.
- Upon saving, the Smartnode will regenerate the container definition files based on the settings (the exact files generated will depend on the settings) with all of the appropriate values based on the user's settings choices.
  - If any of the files already exist, they will be overwritten.
  - Any old files that were *not* overwritten during this process will be deleted.
- `service start` will call `docker-compose` on the container definition files to create / update the containers.
  - It may need to call it with the `--remove-orphans` to get rid of containers that no longer exist due to configuration changes.


### Others

These architecture are just variants of the above, using PD3 and SV3.
As there are no functional / behavioral changes of note, they're not fully assigned and will just be referenced during the trade study by their component composition if they become relevant.