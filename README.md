# Microsoft Foundry prompt agent CI/CD demo

Build a tool-free IT support triage agent in Microsoft Foundry (Azure AI Foundry), export a saved version to JSON, and promote the same reviewed file through staging and production. The pipeline creates **agent versions in existing projects**; it does not provision Azure resources, publish a separate agent application, or manage live traffic routing.

## 1. Build and test in Foundry

1. Open the [Microsoft Foundry portal](https://ai.azure.com/) in the **new** experience, not Foundry (classic). Select or [create a Foundry project](https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup) for development. You need the **Foundry User** role (formerly **Azure AI User**) on the project to build agents.
2. Under **Build > Models**, deploy a supported model such as `gpt-5-mini`. For this demo use the same **deployment name** in every project; Foundry resolves that name within each target project. The deployment name is not necessarily the model catalog name.
3. From the project home page, under **Build an agent**, select **Create an agent**. Name it `support-triage`, use **Text** mode, and select your model deployment in the playground. Copy the `instructions` text from [`agents/support-triage.json`](agents/support-triage.json) into **Instructions**. Leave tools unconfigured.
4. In the playground, try `I can't connect to the office Wi-Fi.` Check that the response contains **Category**, **Priority**, and **Next step** without requesting credentials. Save your changes and note the saved version number from the version selector, not a draft ID.

## 2. Export a saved version to JSON

Install the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and `jq`. Run the following in Bash from the repository root. Copy the **project endpoint** from the development project overview and replace the example endpoint and version:

```bash
az login
PROJECT_ENDPOINT='https://my-foundry.services.ai.azure.com/api/projects/development'
AGENT_NAME='support-triage'
AGENT_VERSION='1'

if payload=$(
  set -o pipefail
  az rest --method get \
    --url "${PROJECT_ENDPOINT%/}/agents/${AGENT_NAME}/versions/${AGENT_VERSION}?api-version=v1" \
    --resource https://ai.azure.com --output json |
    jq -e --arg name "$AGENT_NAME" '
      if .name == $name and .definition.kind == "prompt"
         and (.definition.tools // []) == [] then
        {name, definition: (.definition | {kind, model, instructions})}
      else error("Expected the requested tool-free prompt agent version")
      end'
); then
  printf '%s\n' "$payload" > agents/support-triage.json
else
  printf 'Export failed; the existing JSON was not overwritten.\n' >&2
fi
```

This uses the [Foundry Agents data-plane API](https://learn.microsoft.com/en-us/azure/foundry/reference/rest/) to read an exact version and write the fields used by this demo's create request. It is not a full backup: it intentionally omits IDs, timestamps, descriptions, model settings, and other fields. Agents with tools or connections need additional export and target-project provisioning steps.

Review the diff and open a pull request. Never commit secrets, customer data, or sensitive internal instructions. The checked-in sample can also be deployed as-is without exporting first.

## 3. Configure GitHub and Azure for deployment

1. Create separate staging and production Foundry projects and provision the model deployment named in the JSON in each. Use the same model version and configuration if you want staging to represent production.
2. In your repository settings, create GitHub environments named `staging` and `production`. Restrict both to deployments from the `main` branch. On `production`, configure **required reviewers** and prevent self-review if your GitHub plan supports it. Do not use production for this demo if you cannot enforce the approval gate. Protect `main` with PR reviews and the **Validate agent JSON** check.
3. Create a separate Microsoft Entra deployment identity (app registration/service principal) per environment. Configure a [GitHub OIDC federated credential](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) for each identity, using the matching GitHub environment as its subject: `repo:OWNER/REPO:environment:staging` or `repo:OWNER/REPO:environment:production`. Use audience `api://AzureADTokenExchange`. Grant each identity **Foundry User** on only its target project; no client secret or long-lived Azure key is needed.
4. Set these **environment variables** separately in the GitHub settings for each environment:

   | Variable | Value |
   | --- | --- |
   | `AZURE_CLIENT_ID` | Application (client) ID for that environment's identity |
   | `AZURE_TENANT_ID` | Microsoft Entra tenant ID |
   | `FOUNDRY_PROJECT_ENDPOINT` | Target project endpoint, e.g. `https://my-foundry.services.ai.azure.com/api/projects/staging` |

   The workflow targets Azure public cloud and calls the project data plane directly, without selecting a subscription. For a private project, adapt the workflow to use a runner with network access to the project endpoint.

## 4. Promote a change

1. Merge the reviewed JSON to `main`. PRs and pushes validate the JSON without Azure access.
2. In **Actions > Prompt agent CI/CD**, choose **Run workflow** from `main`. The run validates its commit and deploys to staging; production is requested only if staging deployment succeeds.
3. Before approving production, open the new staging version in the playground and repeat the prompt from step 1. Check the version printed in the staging deployment log and the commit associated with the workflow run. Approving production deploys the same committed JSON, without rebuilding or editing it.
4. Confirm the new version is listed in the production project. For a rollback, restore the last known-good JSON in a new PR and run the same promotion process; this creates another version rather than deleting history. If consumers are pinned to a version, update their reference separately. Consumers configured to use the latest version can pick up a newly created version immediately.

The [create-agent API](https://learn.microsoft.com/en-us/azure/foundry/reference/rest/agents-create-agent-agents-create-agent-from-code) creates an agent on the first run and a new version for an existing name on subsequent runs. Rerunning a deployment can therefore create an additional version. A successful deployment only confirms creation of the version; it is not an automated evaluation of model behavior.