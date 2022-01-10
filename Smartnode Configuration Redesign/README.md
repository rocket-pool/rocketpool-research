# Smartnode Configuration Redesign

The current Smartnode configuration system is one of the largest pain points for node operators.
We have received several complaints from the community about it, including:
- All changes are overwritten upon updating / reinstallation
- Some settings live in `settings.yml`, some in `config.yml`, and some in the various `docker-compose` files; changing the appropriate setting requires knowing where it lives
- The `service config` system is lengthy and annoying to traverse
- Default settings cannot be adjusted once a user has run `service config` unless they run it again
- The CLI aspect is a turn-off for users that are more comfortable with CUI / GUIs

To address these concerns, we are going to completely re-architect Rocket Pool's configuration system - both in the way it stores files, and in the way users configure the Smartnode.
This collection of pages will capture the engineering efforts around this redesign.

- [Requirements.md](./Requirements.md) addresses the user needs and corresponding requirements that the new system must adhere to, as a baseline while designing new approaches.
- [Config-Candidates.md](./Config-Candidates.md) holds a collection of candidate architectures for the new configuration file and `docker-compose` files that satisfy the requirements.
- [Smartnode-Candidates.md](./Smartnode-Candidates.md) contain several candidate architectures for the Smartnode's `service config` function that satisfy the requirements.
- [Analysis.md](./Analysis.md) documents a trade study of the different candidates, ultimately determining which one is the best choice to design and implement.