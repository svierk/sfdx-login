# 🔐 SFDX Login

This repository implements a simple GitHub composite action that allows logging into any Salesforce org from CI/CD automations based on a Salesforce DX (SFDX) authorization URL.

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
```

The SF CLI in this example workflow is installed via the action [sfdx-cli-setup](https://github.com/svierk/sfdx-cli-setup).

### Log in to a Salesforce org using a JSON web token (JWT)

TBD

## References

The two authorisation options supported by this GitHub composite action can be found in the Salesforce CLI Command Reference here: 

- [org login sfdx-url](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm#cli_reference_org_login_sfdx-url_unified)
- [org login jwt](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm#cli_reference_org_login_jwt_unified)

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-login/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-login/blob/main/LICENSE).