# Provision Release PGP Key

A reusable GitHub Actions workflow that generates a PGP key for signing release artifacts, stores it in the repository secrets, extends its expiration date, and publishes the public key to keyservers.

## When is this workflow useful?

Release repositories like Central Portal (aka Maven Central) [require PGP signatures](https://central.sonatype.org/publish/requirements/gpg/)
for all release artifacts.

[SLSA](https://slsa.dev/) suggests [executing the build on a hosted platform that generates and signs provenance](https://slsa.dev/how-to/get-started#slsa-3)
along with some other requirements.

For instance, it means GitHub Actions could be a good platform for building the release artifacts.

However, you probably don't want to place your own PGP key to GitHub repository secrets, so you need to generate and extend the key from time to time.

The workflow solves exactly this problem:
* It can generate a PGP key and configure the repository secrets
* It can extend the PGP key if needed

# Usage

Changes to files go through a pull request, which a coding agent can prepare. The token, the workflow run, and the fingerprint need a person with access to the repository settings.

| Task | Done by | Result |
| --- | --- | --- |
| Add `.github/workflows/pgp-key-maintenance.yaml` | A pull request | The `PGP Key Maintenance` workflow in the Actions tab |
| Create a fine-grained token and store it as `RELEASE_PGP_SECRET_UPDATE_TOKEN` | A repository admin | A repository secret |
| Run `PGP Key Maintenance` | Anyone with write access | The `RELEASE_PGP_PRIVATE_KEY` and `RELEASE_PGP_PASSPHRASE` secrets, the `RELEASE_PGP_FINGERPRINT` variable, and the public key on the keyservers |
| Publish the fingerprint | A maintainer | Users can verify the release signatures |
| Sign the release artifacts with the generated key | A pull request | Signed artifacts from the release workflow |

[pgjdbc/pgjdbc](https://github.com/pgjdbc/pgjdbc) is a working integration: see its [`pgp-key-maintenance.yaml`](https://github.com/pgjdbc/pgjdbc/blob/master/.github/workflows/pgp-key-maintenance.yaml) and the `Publish to Central Portal` step in [`release.yml`](https://github.com/pgjdbc/pgjdbc/blob/master/.github/workflows/release.yml).

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
        uses: vlsi/provision-release-pgp-key/.github/workflows/pgp-key-maintenance.yaml@7c6e3c4eb82ecdd18d1959f3f1a41af9ea0aac0e # v1.0.0
        secrets:
          RELEASE_PGP_SECRET_UPDATE_TOKEN: ${{ secrets.RELEASE_PGP_SECRET_UPDATE_TOKEN }}
          RELEASE_PGP_PRIVATE_KEY: ${{ secrets.RELEASE_PGP_PRIVATE_KEY }}
          RELEASE_PGP_PASSPHRASE: ${{ secrets.RELEASE_PGP_PASSPHRASE }}
        with:
          # Replace with your project's name and an address the project controls
          key-name: my-project releases
          key-email: my-project-releases@example.org
          key-lifetime: 1825
          # The list of keyservers is optional, and the default value is shown below
          keyservers: |
            keyserver.ubuntu.com
            keys.openpgp.org
            pgp.mit.edu
    ```

    Pass all three secrets. Without `RELEASE_PGP_PRIVATE_KEY` and `RELEASE_PGP_PASSPHRASE` the second run can't extend the key. Passing them by name, rather than with `secrets: inherit`, keeps the other repository secrets away from the called workflow.

    As an alternative option, you could copy the contents of [.github/workflows/pgp-key-maintenance.yaml](.github/workflows/pgp-key-maintenance.yaml)
to your own repository.

2. Configure `key-name`, `key-email`, `key-lifetime` as needed (see [Inputs](#inputs)).
3. Create a [fine-grained personal access token](https://github.com/settings/personal-access-tokens/new):
    * Resource owner: the owner of the repository. For an organization repository, the organization must allow fine-grained tokens, and it may have to approve the token.
    * Repository access: only the repository that holds the key.
    * Repository permissions: `Secrets: Read and write` and `Variables: Read and write`.
4. Store the created token as a repository secret `RELEASE_PGP_SECRET_UPDATE_TOKEN`:

    ```bash
    gh secret set RELEASE_PGP_SECRET_UPDATE_TOKEN --repo <owner>/<repo>
    ```

    `gh` prompts for the value, so the token stays out of the shell history.
5. Execute the workflow: navigate to Actions, select `PGP Key Maintenance`, click `Run workflow`. Or run it from the command line:

    ```bash
    gh workflow run pgp-key-maintenance.yaml --repo <owner>/<repo>
    ```

6. The workflow would generate a key, create repository secrets, and it would send the key to the keyservers
7. Publish the key fingerprint on your website so the users would know the official release signing key. Read the fingerprint from the repository variable:

    ```bash
    gh variable get RELEASE_PGP_FINGERPRINT --repo <owner>/<repo>
    ```

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `key-name` | On the first run | `Name-Real` of the generated key, for example `<project> releases`. The workflow looks the new key up by this name, so it must be set when the key is generated. |
| `key-email` | On the first run | `Name-Email` of the generated key. Use an address the project controls, such as a project mailing list. |
| `key-lifetime` | Yes | Days until the key expires, counted from the run. `1825` is five years. |
| `keyservers` | No | Whitespace-separated list of keyservers that receive the public key. The default is `keyserver.ubuntu.com`, `keys.openpgp.org`, and `pgp.mit.edu`. |

| Secret | Required | Description |
| --- | --- | --- |
| `RELEASE_PGP_SECRET_UPDATE_TOKEN` | Yes | The fine-grained token from [Initial setup](#initial-setup). The workflow uses it to write the secrets and the variable. |
| `RELEASE_PGP_PRIVATE_KEY` | After the first run | The current key. Pass `${{ secrets.RELEASE_PGP_PRIVATE_KEY }}` from the start: it is empty before the first run. |
| `RELEASE_PGP_PASSPHRASE` | After the first run | The passphrase of the current key, passed the same way. |

## Signing the release artifacts

The workflow stores the key as repository secrets, not as environment secrets, so the release workflow reads them without an `environment:`. Pass them to the build step that signs the artifacts, as environment variables:

```yaml
      - name: Publish to Central Portal
        run: ./gradlew publishToCentralPortal
        env:
          SIGNING_PGP_PRIVATE_KEY: ${{ secrets.RELEASE_PGP_PRIVATE_KEY }}
          SIGNING_PGP_PASSPHRASE: ${{ secrets.RELEASE_PGP_PASSPHRASE }}
```

`RELEASE_PGP_PRIVATE_KEY` holds an ASCII-armored secret key, which the Gradle [Signing plugin](https://docs.gradle.org/current/userguide/signing_plugin.html) accepts as an in-memory key:

```kotlin
plugins {
    `maven-publish`
    signing
}

signing {
    useInMemoryPgpKeys(
        System.getenv("SIGNING_PGP_PRIVATE_KEY"),
        System.getenv("SIGNING_PGP_PASSPHRASE"),
    )
    sign(publishing.publications)
}
```

A build that signs with the `gpg` command-line tool imports the key first:

```bash
gpg --batch --import --passphrase-fd 3 --pinentry-mode loopback 3<<<"$SIGNING_PGP_PASSPHRASE" <<<"$SIGNING_PGP_PRIVATE_KEY"
```

## Extending the key

To update the expiration date for the PGP key, execute the workflow again.
It would load the current key, extend it, and send the updated key to the keyservers.

# Results

The workflow would generate the following:
* `RELEASE_PGP_PRIVATE_KEY` secret with a private PGP key
* `RELEASE_PGP_PASSPHRASE` secret with a passphrase for the PGP key
* `RELEASE_PGP_FINGERPRINT` variable with a PGP key fingerprint

# Checklist for coding agents

An agent asked to install this workflow in a repository can prepare the pull request. It can't create the token, so it leaves three tasks to a person: create and store the token, run the workflow, and publish the fingerprint.

1. Read the repository's release workflow and build scripts. Find the step that publishes artifacts, and check whether it signs them already and which secret names it reads.
2. Add `.github/workflows/pgp-key-maintenance.yaml` from [Initial setup](#initial-setup), pinned to the commit SHA shown there. Ask the maintainer for `key-name` and `key-email`; don't invent an email address.
3. If the build already signs with other secrets, point it at `RELEASE_PGP_PRIVATE_KEY` and `RELEASE_PGP_PASSPHRASE`. The first run then generates a new key, so the signing fingerprint changes, and the pull request says so for the maintainer to announce. To keep the existing key instead, store it in `RELEASE_PGP_PRIVATE_KEY` and `RELEASE_PGP_PASSPHRASE` and its fingerprint in the `RELEASE_PGP_FINGERPRINT` variable before the first run; the run then extends that key. If it doesn't sign yet, add signing as in [Signing the release artifacts](#signing-the-release-artifacts). If the repository has no release workflow, leave signing for the pull request that adds one, and say so in the description.
4. List the remaining steps in the pull request description: create the token, store it with `gh secret set RELEASE_PGP_SECRET_UPDATE_TOKEN`, run `gh workflow run pgp-key-maintenance.yaml` after the merge, and publish the fingerprint from `gh variable get RELEASE_PGP_FINGERPRINT`.
5. After the run, check that `gh secret list` shows `RELEASE_PGP_PRIVATE_KEY` and `RELEASE_PGP_PASSPHRASE`, and that `gh variable get RELEASE_PGP_FINGERPRINT` prints a 40-character fingerprint.

# FAQ

### What is the purpose of this workflow?
This workflow helps maintain PGP keys for signing release artifacts in repositories, ensuring compliance with platforms like Maven Central and standards like [SLSA](https://slsa.dev/).

### Why should I use this workflow instead of my personal PGP key?
It enables collaboration by allowing any committer to extend the key without sharing private keys or passphrases. It also avoids storing sensitive personal PGP keys in repository secrets.

### Is storing generated PGP keys secure?
If you already trust GitHub with source code and other secrets, generating and storing PGP keys in secrets doesn't reduce security.

### What configurations are required before using this workflow?
See [Initial setup](#initial-setup).

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

You can customize the list with the `keyservers` input.

### Can this workflow be reused across multiple repositories?
Yes, the workflow supports `workflow_call` for reuse. Alternatively, you can copy the workflow content to other repositories.

Each repository gets its own key, because the workflow writes the secrets to the repository that runs it.

### What happens if there is already an existing PGP key?
If the `RELEASE_PGP_FINGERPRINT` variable is set, the workflow extends that key instead of generating a new one.

### How can I trigger this workflow manually?
The caller workflow in [Initial setup](#initial-setup) has the `workflow_dispatch` trigger, so run it from the Actions tab or with `gh workflow run pgp-key-maintenance.yaml`.
If you copied the reusable workflow instead, uncomment `workflow_dispatch`, comment out `workflow_call`, and hard-code `KEY_NAME`, `KEY_EMAIL`, and `KEY_LIFETIME` in its `env:` block: the `inputs` are empty under `workflow_dispatch`.

### How do I replace the existing PGP key?
To generate a new key, clear the `RELEASE_PGP_FINGERPRINT` variable and rerun the workflow.

### Does this workflow need changes for private repositories?
It works for both public and private repositories. Make sure the required permissions are correctly configured for the token.

### How can I ensure security while using fine-grained access tokens?
Use short-lived tokens when possible and create tokens specifically for PGP key updates, limiting token exposure.

# License

Apache 2.0
