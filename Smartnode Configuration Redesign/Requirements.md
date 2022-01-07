# User Needs and Requirements

This file captures the needs that users of Rocket Pool's Smartnode will encounter when configuring their system.
Each one must be satisfied by one or more requirements, which dictate the rules and features that the new configuration system must adhere to and provide respectively.


## User Needs
| ID | Description |
|--|--|
| U1 | **[ Config Files ]** |
| U1.1 | I need Rocket Pool to remember my previous settings, even after an upgrade. |
| U1.2 | I need Rocket Pool to understand and retain my existing settings when going from the legacy config system to this new one. |
| U1.3 | I need the config files to work with Docker Compose, as they do now. |
| U1.4 | I should not need to manually edit any of the files in order to change settings. |
| | |
| U2 | **[ Service Config Interface ]** |
| U2.1 | I need to be able to change all of the options associated with Rocket Pool through a single configuration UI. |
| U2.2 | I want the controls to be intuitive - checkboxes for booleans, dropdowns for enums, textboxes for numbers or strings, etc. |
| U2.3 | I want the UI to explain each of the options to me. |
| U2.4 | I want the UI to let me know which settings are new / which have changed with each new version. |
| U2.5 | I only want to see the options that are relevant to the selections I've made so far. |
| U2.6 | I need to quickly switch between arbitrary settings without exiting. |
| U2.7 | I need to specify which containers I want Rocket Pool to run, and which I control myself (hybrid mode). |
| U2.8 | I need to be able to cancel without saving anything, so my original settings are preserved. |
| U2.9 | I want the UI to show me the navigation controls at all times, so I don't get confused. | 


## Requirements
| ID | Description | Needs Satisfied |
|--|--|--|
| R1 | The system shall store all of the settings on disk. | U1.1 |
| R2 | The system shall generate YML files compatible with Docker Compose. | U1.3 |
| R3 | The system shall load all of the previous settings upon being run. | U1.1, U1.2 |
| R4 | The system shall allow the user to traverse all settings in a navigable, hierarchical format. | U2.1, U2.6 |
| R5 | The system shall use standard form elements to display the values (and options) of each setting. | U2.2 |
| R6 | The system shall display the name, a detailed description, the value, and the hierarchy of the setting. | U2.2, U2.3 |
| R7 | The system shall indicate, on each setting, if it has been modified in (or is new for) the current version being configured. | U2.4 |
| R8 | The system shall hide options that are made irrelevant by other settings' values. | U2.5 |
| R9 | The system shall allow users to state whether their selection of Execution, Fallback, and Beacon clients should be managed by Rocket Pool, or are managed externally (hybrid mode). | U2.7 |
| R10 | The system shall hold all changes temporarily in memory until the user explicitly commits them to disk. | U2.8 |
| R11 | The system shall be able to exit without committing changes to disk. | U2.8 |
| R12 | The system shall be able to commit changes to disk without exiting. | U2.8 |
| R13 | The system shall display the navigation controls in a persistent segment of the screen at all times. | U2.9 |
| R14 | The system shall document which version it is currently using, so future releases understand how to interpret the current configuration. | U1.1, U1.2 |