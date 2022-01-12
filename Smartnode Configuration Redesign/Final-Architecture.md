# New Smartnode Configuration Architecture

This describes the selected architecture of the new Smartnode configuration system based on the candidates and trade studies performed as part of this effort.


## Third-Party Interoperability

Currently, the daemon (`rocketpool_api`) is responsible for all of the Smartnode operations that involve interacting with the chain.
Calls and transactions are all handled by invoking the executable with various command line arguments.
Control over Docker and the configuration files, however, is handled by the CLI binary (all of the `rocketpool service` commands).
As the daemon is *in* a Docker container, control over the Docker configuration and starting / stopping the containers must be done via an external application.

This means that other (e.g. third-party) applications can interact with the daemon for chain-related commands.
For these tasks, they can effectively act as a replacement for the CLI (which is essentially just a frontend).
However, they will need to either replicate its behavior for these `service` commands, or simply call upon the CLI directly.

To make this easier for other developers, we will be expanding upon the command line arguments offered by both the CLI and the daemon.
We will also be documenting these surfaces more thoroughly.
Ultimately, doing this enables things such as third-party GUIs or webapps to leverage the underlying functionality of the CLI where necessary, and replace it where possible.


## Configuration Files

### Parameter Definitions

The new system will remove the `config.yml` file, which was previously used to define some top-level parameters for the Smartnode and the parameters for each ETH1 / ETH2 client.
Some of these things (such as the parameter descriptions) are not meant to be modifiable by the user, while others (such as the automatic transaction max fee threshold or the Docker images for each client) are.
This causes confusion with the users, as some things are meant to be modified here and not in e.g. `service config`, while other are specified in `service config`.

We believe that read-only information, such as parameter definitions, should not be editable by the user and thus should not be stored in a file.
The Smartnode CLI will instead store all of this information directly in its own code.
This comes with another advantage: some components of the parameters (e.g. their default values or descriptions) can now be constructed *dynamically*, which was not possible with the `config.yml` approach before.
For example: Geth's default recommended cache setting can now be changed based on the user's total system RAM rather than a blanket setting based on its CPU architecture.

To provide support for third-party applications that intend to replace the CLI, we will be adding a new command: `rocketpool service parameters` which prints a YAML (or JSON, TBD) representation of the parameter definitions for the version of the Smartnode that the CLI was built for.


### Parameter Values

Once the user finishes `service config`, the Smartnode will store the parameter values for every setting in a persistent file.
This role is currently played by `settings.yml`.
It will continue to do so in the new architecture.
Upon calling `service start`, the CLI will parse this file, construct environment variables for each setting, and pass them into the call to `docker-compose` that launches the containers.

Third party applications can replicate this behavior, or simply invoke the CLI directly to do the heavy lifting.


### Docker Compose Files

For the sake of maximizing modularity, we will be moving to a new setup where each container managed by Rocket Pool will now get its own `docker-compose` file rather than rolling all of them up into a single file.
These files will be stored during the installation process as templates.
After `service config`, the Smartnode will copy the templates for every relevant container into a runtime directory, omitting the files for the containers that were not relevant, and execute `docker-compose` on the entire directory at once.
The template files will be extremely parameterized via the use of environment variable substitution - each of which is controlled by a `service config` parameter - such that modifications to the files themselves should not be necessary except under the most unusual of infrastructures. 

This has a major advantage: it allows first-class support of Hybrid Mode (where the Smartnode attaches to an externally-managed ETH1 and/or ETH2 client).
In such a configuration, the Smartnode can simply omit the `eth1` and/or `eth2` container definition files so the user no longer has to modify them manually.

Users that *do* need to modify things should modify the templates, rather than the copied runtime files as the templates will be copied into the runtime directory with each execution of `service config`.

To mitigate situations where the user's changes are overwritten during updates via `service install`, we are exploring the use of three-way merge libraries, much like `openssh` does for updates to its configuration file.
Ideally the `docker-compose` templates are build modularly enough that such merges never need to occur because we never modify the files, but this capability will cover such a case in the event that it happens.
As the only ones that will ever encounter it are advanced users with custom Docker needs, we believe such individuals are capable of navigating a three-way merge and don't consider this requirement to be an issue.


## User Interface

In addition to the above backend changes, the Smartnode will now come with a Terminal UI specifically for the `service config` action.
The exact implementation details are still under design, but this fundamental component will be a part of the new solution.

It's possible that, if popular enough, the CLI could gradually be migrated so that more of the commands use the TUI instead of the current interview-based actions.
We will keep things modular enough to permit this in the future.


## Settings

We will be adding many more configurable settings to `service config`, but due to the easily-traversable hierarchy structure afforded by the TUI, this will not cause a burden on the user.
They are free to ignore the parameters they don't want to change, and jump directly to the parameters they *do* want to change.
Furthermore, to prevent overwhelming the users, we will be hiding the less-common settings under "advanced" sections of the hierarchy.