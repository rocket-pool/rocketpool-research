# Candidate Analysis and Trade Study

Once a collection of candidate architectures has been established, this page will document a trade study between them to determine their relative strengths, weaknesses, risks and mitigations, development times, and other factors.
Ultimately, this will result in the selection of one or two to pursue for further development.


## Config Candidates

### The "Other" Configs with PD3 and SV3

**PD3** and **SV3** refer to storing the parameter definitions and settings values in several hierarichical files.
Most file-based serialization standards (XML, JSON, YAML) already support hierarchies internally, so this doesn't really give us any benefit and adds overall complexity. 

Pros:
- Makes diffs a little easier to interpret, as only the file that stores the relevant settings changes will be modified

Cons:
- More cumbersome to parse things
- Smartnode needs extra logic to serialize and deserialize from arbitrary hierarchies


**Conclusion**: These options are not worth the effort and should be discarded.


### DC1-Based Configurations

These candidates rely on **DC1** - a single prebuilt `docker-compose` file that's configured via environment variables.
This conflicts with requirement **R9** (support for externally-managed containers) as the docker file can't be modified to conditionally add or remove containers, which is the problem we have today.

**Conclusion**: Anything with **DC1** cannot be used.


### SV2-Based Configurations

These candidates use **SV2** - there is no intermediate parameter value file, and everything is stored in the dynamically generated `docker-compose` file / files.

Pros:
- The configuration comes directly from the `docker-compose` files, so if the user made any modifications to them, they'll persist in `service config` as long as whatever they've changed can be managed by it.

Cons:
- It makes **R20** and **R21** harder to satisfy, since third-party config tools now need to generate entire `docker-compose` files instead of simple parameter value files.
- There's not an easy-to-follow one-to-one mapping between parameters and their manifestations in the `docker-compose` files, so it's harder for humans to read.
- Some settings are meant to be environment variables for the launch scripts, so those would have to be dynamically generated too - this is kind of a deal breaker.

**Conclusion**: **SV2** is inferior to SV1, so these configurations are suboptimal.


### PD1 vs. PD2

**PD1** (there is a static file that describes all of the parameters) is more or less how things are done today.
We could remove that and bake everything directly into the Smartnode (this is **PD2**).

Pros of PD1:
- Separation of concerns; changes to the parameters (e.g. default settings, descriptions, additions or removals) can be done entirely through this file and thus don't require changes to the Smartnode CLI.
  - That being said, each release of the Smartnode is directly coupled to a snapshot of the parameter definition, so any changes would require a new release anyway.
  - This is more convenient from a developer perspective, not from an end user one.
- Third party configuration files can be built around the parameter definitions for each Smartnode release easily (see **R20** and **R21**).


Cons of PD1:
- The user can change the parameter definitions, resulting in undefined or erroneous behavior.
- We're limited in terms of defaults, descriptions, and other things to the constructs that the serialization format (e.g. YAML) provides


Pros of PD2:
- The parameter definitions are fixed in code, so users can't accidentally mess with them
- We are not limited by anything; for example, we could dynamically scan the system RAM and come up with a default for Geth's cache that is tailored to the individual system, or we could include string substitution in parameter definitions if it made sense to do so

Cons of PD2:
- The parameter definitions aren't explictly exposed, so third party config utilities need to review the code to stay in lock-step with Smartnode releases

**Conclusion**: After further discussion, the pros of PD2 outweigh the cons.
**We will use PD2**. 


### DC2, DC3, and DC4

**DC2** is mostly what we have today, but it breaks each container up into its own file.
With respect to this architecture, it would mean we actually have several **template files**, and only fill in / copy the ones that are relevant.
For example, we would have a template for every container, but if the user runs hybrid with their own eth2 client, we just don't copy the eth2 container file into the working directory.
All of the settings are converted into via environment variables as they are today, and passed into the `docker-compose` files where appropriate.

Pros:
- Supports hybrid mode and enabling / disabling of Grafana
- Completely standard `docker-compose` files, their contents don't change per setup
- Doesn't require the Smartnode to know how to create `docker-compose` files
- Users can change the template, and their modifications will be preserved between iterations

Cons:
- None?


**DC3** involves generating a unique `docker-compose` file that captures the entire setup at the end of `service config`.

Pros:
- Supports hybrid mode and enabling / disabling of Grafana
- Greatly simplifies the call to `docker-compose up` in the Smartnode
- One place to look for inconsistencies or errors

Cons:
- The Smartnode needs to be taught how to generate `docker-compose` files
- Possible to create an inconsistency between the settings value file and this file by editing it directly


**DC4** is the same as DC4 but splits the generated files up - one file per container.

Pros:
- Supports hybrid mode and enabling / disabling of Grafana
- Lends to more modular implementation

Cons:
- The Smartnode needs to be taught how to generate `docker-compose` files
- Possible to create an inconsistency between the settings value file and these files by editing them directly

**Conclusion**: More discussion is needed first.


## UI Candidates

**UI1** (the text-based adventure) is the simplest option and the closest to what we have today, but it's going to be very cumbersome to navigate and change lots of settings.
It will likely take *more* time via this route than it already does, and one of the things we're trying to accomplish is limiting the amount that the UI gets in your way.

**UI2** is an interesting approach.
It's still within the terminal, so it still works via SSH (e.g. no radical change from a user's interaction workflow), but it provides a much more flexible platform for configuring things.
It will require its own trade study regarding which framework to use though, and it will likely take longer to build than **UI1** would.

**UI3** is less of a UI architecture around `service config` and trends more towards replacing the CLI entirely with a different UI.
This is what Dave started experimenting with via the old GUI, for example.
It's the most ill-defined of the three because it's purposefully vague (you could really implement this with any technology you wanted, including the TUI from **UI2** or even have multiple implementations).
It would require building a supplemental UI anyway.

**Conclusion**: Leaning towards UI2 for initial experimentation.