# 🔐 SFDX Login

This repository implements a simple GitHub composite action that allows logging into any Salesforce org from CI/CD automations based on either a Salesforce DX (SFDX) authorization URL or using a JSON web token (JWT). Logging into an org authorizes the CLI to run other commands that connect to that org, such as deploying or retrieving a project. You can log into different types of orgs, such as sandboxes, Dev Hubs, Env Hubs, production orgs, and scratch orgs.

## Usage

### Log in to a Salesforce org using a Salesforce DX authorization URL

To be able to log in with an SFDX Auth URL, you must first generate it. The easiest option to achieve this is to redirect the output of the following command for an already authorized org to a JSON file like:

```
sf org display --target-org my-org --verbose --json > authFile.json
```

The resulting JSON file contains the URL in the "sfdxAuthUrl" property of the "result" object. Since we need the _authFile.json_ contents for the login action, but saving raw JSON inputs in GitHub secrets is known to cause problems, we perform an additional step and encode the contents as a Base64 string to avoid headaches like:

```
cat authFile.json | base64
```

We then only have to store the Base64 string received in a GitHub action secret, e.g. SFDX_AUTH_URL, and can reference it whenever we are using the action in one of our workflows. A complete guide to secrets can be found here: [Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)

In a GitHub workflow, the use of the action after the initial checkout step and the installation of the SF CLI could then look like this:

```
jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@main
        
      - name: Salesforce Org Login
        uses: svierk/sfdx-login@main
        with:
          sfdx-url: ${{ secrets.SFDX_AUTH_URL }}
          alias: awesome-org
```

The SF CLI in this example workflow is installed via the action [sfdx-cli-setup](https://github.com/svierk/sfdx-cli-setup).

### Log in to a Salesforce org using a JSON web token (JWT)

The JWT login flow requires a custom connected app to be created as well as a digital certificate, also called a digital signature, to sign the JWT request. You can create a self-signed certificate using OpenSSL. How to achieve this is already well documented:

- [Authorize an Org Using the JWT Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm) | Salesforce DX Developer Guide
- [How To Use GitHub Actions, OAuth and SFDX-CLI for Continuous Integration](https://salesforcedevops.net/index.php/2022/04/05/how-to-use-github-actions-oauth-and-sfdx-cli-for-continuous-integration/) | Blog Post

The following three parameters must be passed to the login action:

1. **client-id** | OAuth client ID (consumer key) of the custom connected app
2. **jwt-secret-key** | Contents of the server.key file containing the private key
3. **username** | Username of the user logging in

In a GitHub workflow, the use of the action after the initial checkout step and the installation of the SF CLI could then look like this:

```
jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@main
        
      - name: Salesforce Org Login
        uses: svierk/sfdx-login@main
        with:
          client-id: ${{ secrets.SFDX_CONSUMER_KEY }}
          jwt-secret-key: ${{ secrets.SFDX_JWT_SECRET_KEY }}
          username: ${{ secrets.SFDX_USERNAME }}
```

The SF CLI in this example workflow is installed via the action [sfdx-cli-setup](https://github.com/svierk/sfdx-cli-setup).

## Inputs

Provide **either** `sfdx-url` (SFDX Auth URL flow) **or** the `client-id` / `jwt-secret-key` / `username` trio (JWT flow).

| Name                  | Required | Default | Description                                                                                  |
| --------------------- | -------- | ------- | -------------------------------------------------------------------------------------------- |
| `sfdx-url`            | no       |         | Base64 encoded SFDX authorization URL (uses the `force://` protocol) for the SFDX Auth URL flow. |
| `client-id`           | no       |         | OAuth client ID (consumer key) of the connected app for the JWT login flow.                  |
| `jwt-secret-key`      | no       |         | Contents of the `server.key` private key file for the JWT login flow.                        |
| `username`            | no       |         | Username of the user logging in for the JWT login flow.                                      |
| `set-default-dev-hub` | no       | `false` | Set the authenticated org as the default Dev Hub.                                            |
| `set-default`         | no       | `true`  | Set the authenticated org as the default that all org-related commands run against.          |
| `alias`               | no       |         | Alias for the org.                                                                           |

## Outputs

After a successful login the action exposes details of the authenticated org so that follow-up steps can reference them:

| Name           | Description                            |
| -------------- | -------------------------------------- |
| `username`     | Username of the authenticated org.     |
| `org-id`       | ID of the authenticated org.           |
| `instance-url` | Instance URL of the authenticated org. |

```yaml
- name: Salesforce Org Login
  id: login
  uses: svierk/sfdx-login@main
  with:
    sfdx-url: ${{ secrets.SFDX_AUTH_URL }}
    alias: awesome-org

- name: Show authenticated org
  run: echo "Logged in as ${{ steps.login.outputs.username }} (${{ steps.login.outputs.instance-url }})"
```

## Security

Credentials are passed to the underlying CLI commands via environment variables instead of being interpolated into the shell script, which avoids leaking them into the rendered command and prevents command injection.

The temporary `authFile.json` file created during the SFDX Auth URL flow is removed automatically when the step finishes, even if the login fails. For the JWT flow, the private key is written to a `server.key` file in the runner's temporary directory (`$RUNNER_TEMP`) with `600` permissions and is intentionally kept for the duration of the job. This is required because the Salesforce CLI stores the key's path in the auth record and re-reads it on every JWT token refresh — for example when a later step runs `sf org create scratch`, deploys metadata, or deletes the org. Removing the key right after login would cause those steps to fail with `ENOENT: no such file or directory ... server.key`. The key never persists beyond the job: GitHub-hosted runners are destroyed afterward, and `$RUNNER_TEMP` is cleared between jobs on self-hosted runners.


## References

The two authorisation options supported by this GitHub composite action can be found in the Salesforce CLI Command Reference here: 

- [org login sfdx-url](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm#cli_reference_org_login_sfdx-url_unified)
- [org login jwt](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm#cli_reference_org_login_jwt_unified)

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-login/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-login/blob/main/LICENSE).