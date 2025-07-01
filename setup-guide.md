# Custom GitHub Copilot Chat Extension Setup Guide

This guide provides step-by-step instructions for building a custom version of the GitHub Copilot Chat extension, pre-configuring it with your own models and API keys, and installing it in a VS Code web environment.

## Part 1: Building the Extension (`.vsix` file)

First, you need to set up the development environment and compile the extension into a `.vsix` package.

### Prerequisites

- **Node.js**: Version 22.x or higher.
- **Python**: Version 3.10 to 3.12.
- **npm**: Should be installed with Node.js.
- **Git**: For cloning the repository.

### Steps

1.  **Clone the Repository**:
    ```bash
    git clone https://github.com/microsoft/vscode-copilot-chat.git
    cd vscode-copilot-chat
    ```

2.  **Install Dependencies**:
    ```bash
    npm install
    ```

3.  **Build the VSIX Package**:
    The project uses `@vscode/vsce` for packaging. The simplest way to create the `.vsix` file is to use the `npx` command.

    ```bash
    npx @vscode/vsce@latest package --no-dependencies
    ```

    This command will create a file named `copilot-chat-x.x.x.vsix` (e.g., `copilot-chat-0.29.0.vsix`) in the root of the project directory. This is the extension package you will install.

## Part 2: Pre-configuring Custom Models (Simple Method)

To have the extension use your own models out-of-the-box, you can programmatically add your model configurations and API keys during the extension's startup sequence.

The best place to add this custom logic is in the `BYOKContrib` class, which manages the "Bring Your Own Key" functionality.

### Steps

1.  **Navigate to the BYOK Contribution File**:
    Open the file `src/extension/byok/vscode-node/byokContribution.ts`.

2.  **Add Your Custom Configuration Logic**:
    You can modify the `_onDidAuthStatusChange` method in this file. This method runs when the user is authenticated, making it a good place to register your custom models.

    ```typescript
    // In src/extension/byok/vscode-node/byokContribution.ts
    // ... other imports
    import { BYOKAuthType } from '../../byok/common/byokProvider';

    // ... inside the BYOKContrib class
      private async _onDidAuthStatusChange(authService: IAuthService): Promise<void> {
        // ... existing code
        if (authService.copilotToken && isBYOKEnabled(authService.copilotToken, this._capiClientService)) {
          // ... existing code
          this._byokUIService = new BYOKUIService(this._byokStorageService, this._modelRegistries);

          // --- START: CUSTOM MODEL PRE-CONFIGURATION ---
          const myCustomProviderApiKey = 'sk-your-secret-api-key'; // See security note below
          await this._byokStorageService.storeAPIKey('MyAIService', myCustomProviderApiKey, BYOKAuthType.GlobalApiKey);
          await this._byokStorageService.saveModelConfig('custom-model-1', 'MyAIService', {
              isCustom: true, modelId: 'custom-model-1', name: 'My Custom Model', authType: BYOKAuthType.GlobalApiKey,
            }, BYOKAuthType.GlobalApiKey);
          this._logService.logger.info('BYOK: Pre-configured custom models have been registered.');
          // --- END: CUSTOM MODEL PRE-CONFIGURATION ---

          this._register(this._configurationService.onDidChangeConfiguration(async e => { /* ... */ }));
        }
      }
    ```

3.  **Security Warning**:
    Hardcoding API keys directly into the source code is a major security risk. This approach is only suitable for a temporary, personal build. **Do not share a `.vsix` file built this way.** For a more secure and flexible approach suitable for automated deployments (e.g., Docker), see **Part 5**.

4.  **Re-build the Extension**:
    After saving your changes, run the package command again:
    ```bash
    npx @vscode/vsce@latest package --no-dependencies
    ```

## Part 3: Installing in VS Code for the Web

You can install your custom `.vsix` file in a web-based VS Code environment like `vscode.dev`.

1.  **Open VS Code for the Web** (`https://vscode.dev`).
2.  **Go to the Extensions View**.
3.  Click the `...` menu and select **"Install from VSIX..."**.
4.  Choose the `.vsix` file you created.

The extension will be installed and ready to use with your pre-configured models.

## Part 4: Local Code Indexing in the Web Environment

The extension’s code search and indexing features function locally in the browser.

-   **How it Works**: The extension uses Web Workers to index files without freezing the UI. It accesses your local files via the **File System Access API**.
-   **Required Action**: When you open a local folder in `vscode.dev`, your browser will ask for permission. You **must grant permission** for the extension to read your files for indexing to work.

## Part 5: Automated Configuration for Docker/Deployment

For containerized environments, you need an automated way to provide credentials without hardcoding them. This method uses a JSON configuration file combined with environment variables—a security best practice.

### Step 1: Create a `custom-models.json` File

Create this file in the root of the project. It defines your models and points to environment variables for the secrets.

**Example `custom-models.json`:**
```json
{
  "providers": [
    {
      "name": "Ollama",
      "models": [
        { "id": "llama3", "displayName": "Llama 3 (Docker)" }
      ]
    },
    {
      "name": "OpenAI",
      "apiKeyEnv": "MY_OPENAI_API_KEY",
      "models": [
        { "id": "gpt-4-turbo", "displayName": "GPT-4 Turbo (Docker)" }
      ]
    },
    {
      "name": "AzureBYOK",
      "models": [
        {
          "id": "my-azure-deployment",
          "displayName": "Azure GPT-4 (Docker)",
          "deploymentUrl": "https://my-azure-instance.openai.azure.com/",
          "apiKeyEnv": "MY_AZURE_API_KEY"
        }
      ]
    }
  ]
}
```

### Step 2: Implement the Configuration Importer

Modify `src/extension/byok/vscode-node/byokContribution.ts` to read your config file on startup.

```typescript
// In src/extension/byok/vscode-node/byokContribution.ts
import * as fs from 'fs';
import * as path from 'path';
// ... other imports

export class BYOKContrib extends Disposable implements IExtensionContribution {
  // ... existing properties

  constructor(...) {
    // ... existing constructor logic
    this.importModelsFromConfig().catch(err => {
      this._logService.logger.error('Failed to import custom models from configuration file.', err);
    });
  }

  private async importModelsFromConfig(): Promise<void> {
    const configPath = path.join(this._extensionContext.extensionPath, 'custom-models.json');
    if (!fs.existsSync(configPath)) { return; }

    try {
      const config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
      if (!config.providers) { return; }

      for (const provider of config.providers) {
        const providerRegistry = this._modelRegistries.find(r => r.name === provider.name);
        if (!providerRegistry) { continue; }

        if (provider.apiKeyEnv && process.env[provider.apiKeyEnv]) {
          await this._byokStorageService.storeAPIKey(provider.name, process.env[provider.apiKeyEnv]!, providerRegistry.authType);
        }

        for (const model of provider.models) {
          if (model.apiKeyEnv && process.env[model.apiKeyEnv]) {
            await this._byokStorageService.storeAPIKey(provider.name, process.env[model.apiKeyEnv]!, providerRegistry.authType, model.id);
          }
          await this._byokStorageService.saveModelConfig(model.id, provider.name, {
            isCustom: true, modelId: model.id, name: model.displayName,
            deploymentUrl: model.deploymentUrl, authType: providerRegistry.authType,
          }, providerRegistry.authType);
        }
      }
      this._logService.logger.info('Successfully imported custom models from custom-models.json.');
      // Optional: remove the file to prevent re-running on every startup
      // fs.unlinkSync(configPath);
    } catch (error) {
      this._logService.logger.error('Error processing custom-models.json:', error);
    }
  }

  // ... rest of the class
}
```

### Step 3: Build Your Docker Image

Your `Dockerfile` should copy the source code, include `custom-models.json`, build the extension, and install it.

**Example `Dockerfile`:**
```dockerfile
# Use a base image with Node.js
FROM mcr.microsoft.com/vscode/devcontainers/universal:2

# Copy the extension source code
COPY . /workspace/vscode-copilot-chat
WORKDIR /workspace/vscode-copilot-chat

# Build the extension VSIX package
RUN npm install && 
    npx @vscode/vsce@latest package --no-dependencies

# Install the extension for a VS Code server like open-vscode-server
# The exact path may vary depending on your server.
RUN mkdir -p /home/vscode/.open-vscode-server/extensions && 
    unzip *.vsix -d /home/vscode/.open-vscode-server/extensions/github.copilot-chat

# Command to start the VS Code server
CMD [ "open-vscode-server", "--host=0.0.0.0", "--without-connection-token" ]
```

### Step 4: Run the Docker Container

Pass your API keys as environment variables when you run the container.

```bash
docker run -d -p 8080:8080 
  -e MY_OPENAI_API_KEY="sk-..." 
  -e MY_AZURE_API_KEY="..." 
  my-custom-vscode-image:latest
```

When the container starts, the importer script will automatically and securely configure your custom models.

