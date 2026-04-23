<!--
---
description: This end-to-end sample shows how implement an intelligent PDF summarizer using Durable Functions in C#. 
page_type: sample
products:
- azure-functions
- azure
urlFragment: durable-func-pdf-summarizer-csharp
languages:
- csharp
- bicep
- azdeveloper
---
-->
# Intelligent PDF Summarizer - .NET
The purpose of this sample application is to demonstrate how Durable Functions can be leveraged to create intelligent applications, particularly in a document processing scenario. Order and durability are key here because the results from one activity are passed to the next. Also, calls to services like Cognitive Service or Azure Open AI can be costly and should not be repeated in the event of failures.

This sample integrates various Azure services, including Azure Durable Functions with **Azure Durable Task Scheduler (DTS)** as the orchestration backend, Azure Storage (for blob input/output only), Azure Document Intelligence (Form Recognizer), and Azure OpenAI.

The application showcases how PDFs can be ingested and intelligently scanned to determine their content.

![Architecture Diagram](./media/dotnet-architecture.png)

The application's workflow is as follows:
1.	PDFs are uploaded to a blob storage input container.
2.	A durable function is triggered upon blob upload.
- - Downloads the blob (PDF).
- - Utilizes the Azure Cognitive Service Form Recognizer endpoint to extract the text from the PDF.
- - Sends the extracted text to Azure Open AI to analyze and determine the content of the PDF.
- - Save the summary results from Azure Open AI to a new file and upload it to the output blob container.

Below, you will find the instructions to set up and run this app locally..

## Prerequsites
- [Create an active Azure subscription](https://learn.microsoft.com/en-us/azure/guides/developer/azure-developer-guide#understanding-accounts-subscriptions-and-billing).
- [Install the latest Azure Functions Core Tools to use the CLI](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local)
- [.NET 8](https://dotnet.microsoft.com/en-us/download/dotnet)
- [Docker](https://www.docker.com/) — required to run the Durable Task Scheduler emulator locally.
- [Azure Developer CLI (`azd`)](https://aka.ms/azd) for deployment.
- Access permissions to [create Azure OpenAI resources and to deploy models](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/role-based-access-control).
- [Start and configure an Azurite storage emulator for local blob input/output](https://learn.microsoft.com/azure/storage/common/storage-use-azurite).
- Ensure that your Developer credentials have been granted access to your Azure services for local development.

## local.settings.json
Copy `local.settings.json.sample` to `local.settings.json` at the root of the repo and replace the placeholders with your values. The sample is pre-wired for the DTS emulator on `localhost:8080`.

```json
{
    "IsEncrypted": false,
    "Values": {
      "AzureWebJobsFeatureFlags": "EnableWorkerIndexing",
      "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
      "AzureWebJobsStorage": "UseDevelopmentStorage=true",
      "DURABLE_TASK_SCHEDULER_CONNECTION_STRING": "Endpoint=http://localhost:8080;Authentication=None",
      "TASKHUB_NAME": "default",
      "COGNITIVE_SERVICES_ENDPOINT": "https://<YOUR_VALUE_HERE>.cognitiveservices.azure.com/",
      "AZURE_OPENAI_ENDPOINT": "https://<YOUR_VALUE_HERE>.openai.azure.com/",
      "CHAT_MODEL_DEPLOYMENT_NAME": "<YOUR_VALUE_HERE>"
    }
  }
```

## Running the app locally
1. Start the Durable Task Scheduler emulator:

   ```bash
   docker run --rm -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
   ```

   The emulator exposes the gRPC endpoint on `8080` and the local dashboard on [http://localhost:8082](http://localhost:8082).

2. Start Azurite (for the `input`/`output` blob containers).

3. Create two containers in your local storage account: `input` and `output`.

4. Start the Function App:

   ```bash
   func start --verbose
   ```

5. Upload PDFs to the `input` container. That will execute the blob storage trigger in your Durable Function. Watch the orchestration reach **Completed** in the DTS emulator dashboard.

6. After several seconds, your application should have finished the orchestrations. Switch to the `output` container and notice that the PDFs have been summarized as new files.

>Note: The summaries may be truncated based on token limit from Azure Open AI. This is intentional as a way to reduce costs.

## Inspect the code
This app leverages Durable Functions with the **Azure Durable Task Scheduler (DTS)** backend to orchestrate the application workflow. State is managed by DTS instead of Azure Storage queues/tables — no extra queue or table infrastructure is required. Azure Storage is used only for the PDF `input`/`output` blob containers.

Take a look at the code snippet below, the `ProcessDocument` defines the entire workflow, which consists of a series of steps (activities) that need to be scheduled in sequence. Coordination is key, as the output of one activity is passed as an input to the next. Additionally, Durable Functions handle durability and retries, which ensure that if a failure occurs, such as a transient error or an issue with a dependent service, the workflow can recover gracefully.

![Orchestration Code](./media/code.png)

## Deploy the app to Azure

Use the [Azure Developer CLI (`azd`)](https://aka.ms/azd) to easily deploy the app. 

1. In the root of the project, run the following command to provision and deploy the app:

    ```bash
    azd up
    ```

1. When prompted, provide:
   - A name for your [Azure Developer CLI environment](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/faq#what-is-an-environment-name).
   - The Azure subscription you'd like to use.
   - The Azure location to use.

Once the azd up command finishes, the app will have successfully provisioned and deployed. 
> Note: Encountering a 409 conflict error during deployment is expected and not a cause for concern.

## Monitor orchestrations

After deployment, navigate runs in the Azure Durable Task Scheduler dashboard at [https://dashboard.durabletask.io](https://dashboard.durabletask.io). You can find the DTS task hub endpoint in the deployed function app's `DURABLE_TASK_SCHEDULER_CONNECTION_STRING` app setting or via `azd show`.

# Using the app
To use the app, simply upload a PDF to the Blob Storage `input` container. Once the PDF is transferred, it will be processed using Durable Functions making calls to document intelligence and Azure OpenAI. The resulting summary will be saved to a new file and uploaded to the `output` container.
