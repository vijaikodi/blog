---
layout: post
date: 2025-03-19 12:41:28
title: Chocolatey package automation via github workflow
category: chocolatey 
tags: chocolatey github
---

# Chocolatey Package Deployment Workflow

This repository uses a GitHub Actions workflow to automate the packaging and deployment of Chocolatey packages to GitHub Packages.

## Workflow Overview

The workflow is triggered whenever changes are made to any `chocolateyInstall.ps1` file in the repository. Specifically, it does the following:

1. **Checks out the repository**: The first step ensures that the repository is available for further actions.
2. **Sets up Chocolatey**: The workflow installs Chocolatey so that it can be used for packaging and pushing.
3. **Detects Changed Packages**: It checks which package folders were modified by comparing the latest commit with the previous commit. It looks for changes in the `chocolateyInstall.ps1` file within the `tools` folder of each package.
4. **Packs and Pushes the Package**: For any modified packages, the workflow:
   - Packs the `.nuspec` file into a `.nupkg` package.
   - Pushes the generated `.nupkg` file to GitHub Packages using the API key stored in GitHub Secrets.

## Workflow Trigger

The workflow is triggered when changes are pushed to the `main` branch, specifically when the `chocolateyInstall.ps1` file in the `tools` directory of any package is modified.

```yml
name: Chocolatey Package Deployment

on:
  push:
    paths:
      - "**/tools/chocolateyInstall.ps1"  # Detect changes in any software folder
    branches:
      - main  # Change if using a different branch

permissions:
  packages: write  # Allows pushing Chocolatey packages to GitHub Packages
  contents: read   # Allows reading repository contents

jobs:
  choco_package:
    runs-on: windows-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up Chocolatey
        run: choco install chocolatey

      - name: Get Changed Package Folder
        id: changed-folder
        run: |
          git fetch --prune --unshallow
          $commitSha = git rev-parse HEAD
      
          Write-Host "Commit SHA: $commitSha"
      
          $changedFiles = git diff --name-only HEAD^ HEAD
          Write-Host "Changed files: $changedFiles"
      
          $packageFolders = $changedFiles | Where-Object { $_ -match '^[^/\\]+[/\\]tools[/\\]chocolateyInstall.ps1$' } | ForEach-Object { ($_ -split '[/\\]tools[/\\]')[0] }
      
          if (-not $packageFolders) {
              Write-Host "No changed packages detected. Exiting."
              echo "PACKAGE_FOLDER=" | Out-File -FilePath $env:GITHUB_ENV -Append
              exit 0
          }
      
          Write-Host "Detected package folders: $packageFolders"
          echo "PACKAGE_FOLDER=$($packageFolders -join ' ')" | Out-File -FilePath $env:GITHUB_ENV -Append
        shell: pwsh

      - name: Pack and Push Chocolatey Packages
        env:
          CHOCO_API_KEY: ${{ secrets.APPTOKEN }}  # Store the API key in an environment variable
          OWNER: vijaikodi
        run: |
          if (-not $env:CHOCO_API_KEY) {
              Write-Host "Error: No API key found."
              exit 1
          }

          $folders = "${{ env.PACKAGE_FOLDER }}" -split ' '
          if ($folders -eq "" -or $folders.Length -eq 0) {
              Write-Host "No package folders detected. Skipping."
              exit 0
          }

          foreach ($folder in $folders) {
              $fullPath = "$env:GITHUB_WORKSPACE\$folder"
              Write-Host "Processing package in: $fullPath"

              if (!(Test-Path $fullPath)) {
                  Write-Host "Skipping $folder - Directory does not exist."
                  continue
              }

              # Find .nuspec file
              $nuspecFile = Get-ChildItem -Path $fullPath -Filter "*.nuspec" -File -ErrorAction SilentlyContinue
              if ($nuspecFile -eq $null) {
                  Write-Host "Skipping $folder - No .nuspec file found in $fullPath"
                  continue
              }

              Set-Location $fullPath
              Write-Host "Packing Chocolatey package..."
              choco pack $nuspecFile.FullName

              # Find the newly created .nupkg file
              $packageFile = Get-ChildItem -Path $fullPath -Filter "*.nupkg" -File -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
              if (-not $packageFile) {
                  Write-Host "Error: No .nupkg file was created."
                  exit 1
              }

              Write-Host "Pushing package: $packageFile"

              # Add Chocolatey source with GitHub authentication
              choco source add --name=github --source="https://nuget.pkg.github.com/$env:OWNER/index.json" --user="$env:OWNER" --password="$env:CHOCO_API_KEY" --force

              # Add API Key correctly for Chocolatey push
              choco apikey add --source="https://nuget.pkg.github.com/$env:OWNER/index.json" --key="$env:CHOCO_API_KEY" --force

              # Push using the correct GitHub Packages format
              choco push $packageFile --source "https://nuget.pkg.github.com/$env:OWNER/index.json" --api-key="$env:CHOCO_API_KEY"

              # Clean up the .nupkg file
              Remove-Item -Path $packageFile -Force
              Set-Location $env:GITHUB_WORKSPACE
          }
        shell: pwsh
```

This ensures that the workflow only runs when there are updates to the package installation scripts.

How to Modify chocolateyInstall.ps1 to Trigger the Workflow
To trigger the workflow, you only need to modify or add a new chocolateyInstall.ps1 file in one of the package folders. Here's how to do it:

1. Add or Modify the chocolateyInstall.ps1 File
The chocolateyInstall.ps1 script is used by Chocolatey to install your package. You can find this file in the tools directory inside each package folder.

Example of the package folder structure:

```php-template

/<package-name>/
  ├── tools/
      └── chocolateyInstall.ps1
  ├── <other-files>
  └── <nuspec-file>
```

If you modify the chocolateyInstall.ps1 file or add a new one to any package folder, the workflow will automatically detect the change. You don't need to manually trigger anything.

2. Commit the Changes
Once you've made your modifications, commit the changes to your repository.

Example Git commands:

```bash

git add .
git commit -m "Modified chocolateyInstall.ps1 for <package-name>"
git push origin main
```

This will trigger the GitHub Actions workflow as long as the chocolateyInstall.ps1 file was modified or added.

## Environment Variables
The workflow uses a few environment variables, which you should ensure are set correctly in your GitHub repository's settings:

 - `APPTOKEN`: This is the GitHub API token with the necessary permissions to push packages to GitHub Packages. It should be stored as a GitHub Secret.
 - `OWNER`: The username or organization name on GitHub that owns the package repository. It is used for authentication and setting up the Chocolatey source.

## Modifying the Workflow

If you need to modify the workflow to suit your needs, here are the key parts you might want to change:

   1. Change the trigger: If you want to trigger the workflow on different paths or branches, modify the `on.push` section.
   2. Change the Chocolatey source URL: If you're pushing to a different repository or source, you can change the `https://nuget.pkg.github.com/vijakodi/index.json` URL to your desired source.
   3. Change package details: If you're using a different way to generate the `.nupkg` file or need to modify the packaging process, you can edit the `Pack and Push Chocolatey Packages` step.

## Conclusion
This workflow helps automate the process of packaging and pushing Chocolatey packages to GitHub Packages. By simply modifying the `chocolateyInstall.ps1` file in any package folder, you can trigger the workflow to create and deploy the updated package.

For any additional changes or further customization of the workflow, you can modify the `choco_package` job in the `chocolatey_package_deployment.yml` file.
