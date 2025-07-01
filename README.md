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
      workflow_dispatch:

    permissions: read-all

    jobs:
      pgp-key-maintenance:
        name: PGP key maintenance
        # Use digest pinning for the security and review the workflow before updating the pin
        uses: vlsi/provision-release-pgp-key/.github/workflows/pgp-key-maintenance.yaml@47caa11d98dd9e897523af1f16532bf6152e8444 # v1
        secrets:
          RELEASE_PGP_SECRET_UPDATE_TOKEN: ${{ secrets.RELEASE_PGP_SECRET_UPDATE_TOKEN }}
          RELEASE_PGP_PRIVATE_KEY: ${{ secrets.RELEASE_PGP_PRIVATE_KEY }}
          RELEASE_PGP_PASSPHRASE: ${{ secrets.RELEASE_PGP_PASSPHRASE }}
        with:
          key-name: vlsi/test-pgp-key
          key-email: vlsi--test-pgp-keys@example.org
          key-lifetime: 1825
          # The list of keyservers is optional, and the default value is shown below
          keyservers: |
            keyserver.ubuntu.com
            keys.openpgp.org
            pgp.mit.edu
    ```

    As an alternative option, you could copy the contents of [.github/workflows/pgp-key-maintenance.yaml](.github/workflows/pgp-key-maintenance.yaml)
to your own repository.

2. Configure `key-name`, `key-email`, `key-lifetime` as needed
3. Create a fine-grained access token with `secrets: write` and `variables: write` permissions
4. Store the created token as a repository secret `RELEASE_PGP_SECRET_UPDATE_TOKEN`
5. Execute the workflow: navigate to Actions, select `PGP Key Maintenance`, click `Run workflow`
6. The workflow would generate a key, create repository secrets, and it would send the key to the keyservers
7. Publish the key fingerprint on your website so the users would know the official release signing key

## Extending the key

To update the expiration date for the PGP key, execute the workflow again.
It would load the current key, extend it, and send the updated key to the keyservers.

# Results

The workflow would generate the following:
* `RELEASE_PGP_PRIVATE_KEY` secret with a private PGP key
* `RELEASE_PGP_PASSPHRASE` secret with a passphrase for the PGP key
* `RELEASE_PGP_FINGERPRINT` variable with a PGP key fingerprint

# FAQ

### What is the purpose of this workflow?
This workflow helps maintain PGP keys for signing release artifacts in repositories, ensuring compliance with platforms like Maven Central and standards like [SLSA](https://slsa.dev/).

### Why should I use this workflow instead of my personal PGP key?
It enables collaboration by allowing any committer to extend the key without sharing private keys or passphrases. It also avoids storing sensitive personal PGP keys in repository secrets.

### Is storing generated PGP keys secure?
If you already trust GitHub with source code and other secrets, generating and storing PGP keys in secrets doesn't reduce security.

### What configurations are required before using this workflow?
You need to:
1. Add the workflow to your repository.
2. Configure parameters (`key-name`, `key-email`, `key-lifetime`) as needed.
3. Create a fine-grained GitHub access token with `secrets: write` and `variables: write` permissions.
4. Store the token as a repository secret.
5. Run the workflow via GitHub Actions.

### What does the workflow generate upon execution?
It generates:
- `RELEASE_PGP_PRIVATE_KEY`: Secret containing the private PGP key.
- `RELEASE_PGP_PASSPHRASE`: Secret containing the passphrase for the key.
- `RELEASE_PGP_FINGERPRINT`: Variable containing the PGP key fingerprint.

### How do I extend the PGP key expiration date?
Rerun the workflow. It will update the expiration date and sync the updated key with the keyservers.

### What permissions are needed for the GitHub access token?
The token requires `secrets: write` (for creating and updating the secret) and `variables: write` permissions (for creating
a variable with PGP key fingerprint).

### What keyservers are used to publish the public PGP keys?
The default list of keyservers is:
- `keyserver.ubuntu.com`
- `keys.openpgp.org`
- `pgp.mit.edu`

- You can customize the list in the workflow configuration.

### Can this workflow be reused across multiple repositories?
Yes, the workflow supports `workflow_call` for reuse. Alternatively, you can copy the workflow content to other repositories.

### What happens if there is already an existing PGP key?
The workflow will extend the existing key instead of generating a new one.

### How can I trigger this workflow manually?
Uncomment the `workflow_dispatch` trigger in the YAML file and use the GitHub Actions UI to manually start the workflow.

### How do I replace the existing PGP key?
To generate a new key, clear the `RELEASE_PGP_FINGERPRINT` variable and rerun the workflow.

### Does this workflow need changes for private repositories?
It works for both public and private repositories. Make sure the required permissions are correctly configured for the token.

### How can I ensure security while using fine-grained access tokens?
Use short-lived tokens when possible and create tokens specifically for PGP key updates, limiting token exposure.

# License

Apache 2.0
