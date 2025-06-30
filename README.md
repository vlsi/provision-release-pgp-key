# Provision Release PGP Key

This is a GitHub Actions workflow that generates and updates PGP-key.

## When is this workflow useful?

Release repositories like Central Portal (aka Maven Central) [requires PGP signatures](https://central.sonatype.org/publish/requirements/gpg/)
for all release artifacts.

[SLSA](https://slsa.dev/) suggests [executing the build on a hosted platform that generates and signs provenance](https://slsa.dev/how-to/get-started#slsa-3)
along with some other requirements.

For instance, it means GitHub Actions could be a good platform for building the release artifacts.

However, you probably don't want to place your own PGP key to GitHub repository secrets, so you need to generate and extend the key from time to time.

The workflow solves exactly this problem:
* It can generate a PGP key and configure the repository secrets
* It can extend the PGP key if needed

# Usage

## Initial setup

1. Add a workflow to your repository.

    `.github/workflows/pgp-key-maintenance.yaml`

    ```yaml
    name: PGP Key Maintenance

    on:
      push:
        branches:
          - main
      workflow_dispatch:

    permissions: read-all

    env:
      KEY_NAME: vlsi/test-pgp-key
      KEY_EMAIL: vlsi--test-pgp-keys@example.org
      KEY_LIFETIME: 1825
    jobs:
      pgp-key-maintenance:
        name: PGP key maintenance
        secrets: inherit
        # Use digest pinning for the security and review the workflow before updating the pin
        uses: vlsi/provision-release-pgp-key/.github/workflows/pgp-key-maintenance.yaml@e25e2522533ee5ad5b6f9222a0adbadff7249d4d # v1
        with:
          key-name: vlsi/test-pgp-key
          key-email: vlsi--test-pgp-keys@example.org
          key-lifetime: 1825
    ```

    As an alternative option, you could copy the contents of [.github/workflows/pgp-key-maintenance.yaml](.github/workflows/pgp-key-maintenance.yaml)
to your own repository.

2. Configure `key-name`, `key-email`, `key-lifetime` as needed
3. Create a fine-grained access token with `secrets: write` and `variables: write` permissions
4. Store the created token as a repository secret `RELEASE_PGP_SECRET_UPDATE_TOKEN`
5. Execute the workflow: navigate to Actions, select `PGP Key Maintenance`, click `Run workflow`
6. The workflow would generate a key, create repository secrets, and it would send the key to the keyservers

## Extending the key

To update the expiration date for the PGP key, execute the workflow again.
It would load the current key, extend it, and send the updated key to the keyservers.

# Results

The workflow would generate the following:
* `RELEASE_PGP_PRIVATE_KEY` secret with a private PGP key
* `RELEASE_PGP_PASSPHRASE` secret with a passphrase for the PGP key
* `RELEASE_PGP_FINGERPRINT` variable with a PGP key fingerprint

# License

Apache 2.0
