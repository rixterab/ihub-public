# iHub Public Resources

This repository contains public resources for iHub users and teams.

The main content is reusable iHub integration templates. These templates can be imported into iHub to help teams get started with common integrations, automation flows, and asset import use cases without building everything from scratch.

## Repository Contents

- `templates/` contains exported iHub integration templates. Each template has its own folder with the exported flow JSON, a `manifest.json`, a short README, and any icon or supporting files needed by the template.
- `aws/` contains AWS integration resources, including XSD schemas that can be referenced from iHub actions when parsing AWS responses.

## Templates

Templates are exported from the iHub integrations list by using the `Export as template` action.

An exported template usually includes:

- one or more integration flow `.json` files
- a `manifest.json` file describing the template
- a `README.md` explaining what the template does and when to use it
- optional icons or supporting files

Before publishing a template, make sure it does not contain secrets, private URLs, customer data, personal data, access tokens, or environment-specific values that should not be shared.

## Contributing Templates

iHub users are welcome to contribute templates to this repository.

To contribute a template:

1. Export the integration from the iHub integrations list with `Export as template`.
2. Add the exported files to a new folder under `templates/`.
3. Include a clear `README.md` that explains what the template does, what systems it connects to, and any setup required before import.
4. Check that credentials, tokens, private endpoints, and sensitive data have been removed.
5. Open a pull request with a short description of the template and its intended use case.

Good templates should be easy for another iHub user to understand, import, configure, and adapt.
