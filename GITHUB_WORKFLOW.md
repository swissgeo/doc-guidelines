# Github Workflow for SWISSGEO repositories

All SWISSGEO repositories should be configured to use github workflow to enforce documentation quality and automate the release process.

We use github reusable workflow that are centralized in [swissgeo/.github](https://github.com/swissgeo/.github) repository. In the repository we only need the github workflow to call the reusable workflow.

Workflows are automatically set during creation of the repository and then managed by terraform.
