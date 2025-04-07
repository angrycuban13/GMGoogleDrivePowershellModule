# GMGoogleDrive Powershell Module

[![PowerShell Gallery](https://img.shields.io/powershellgallery/v/GMGoogleDrive.svg)](https://www.powershellgallery.com/packages/GMGoogleDrive/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Google Drive REST API module for Powershell with Google Sheets API support.

## Table of Contents

- [GMGoogleDrive Powershell Module](#gmgoogledrive-powershell-module)
  - [Table of Contents](#table-of-contents)
  - [Google Project Setup](#google-project-setup)
    - [Project Creation](#project-creation)
    - [API Setup](#api-setup)
    - [Credentials Setup](#credentials-setup)
      - [OAuth 2.0](#oauth-20)
        - [OAuth 2.0 Consent Screen](#oauth-20-consent-screen)
        - [OAuth 2.0 Credentials](#oauth-20-credentials)
      - [Service Account](#service-account)
  - [Authorization Code and Refresh Token Retrieval](#authorization-code-and-refresh-token-retrieval)
    - [Using Powershell](#using-powershell)
      - [Authorization Code](#authorization-code)
      - [Refresh Token](#refresh-token)
    - [Manually](#manually)
  - [Access Token Retrieval](#access-token-retrieval)
  - [Usage](#usage)
  - [Error Handling](#error-handling)
  - [Automation](#automation)

---

## Google Project Setup

Google Drive is a free service for file storage files. In order to use this storage you need a Google user which will own the files, and a Google API client.

### Project Creation

You need to create a Google project and enable the Google Drive API and Google Sheets API for it to use this module.

1. Navigate to the [Google Cloud Console](https://console.cloud.google.com/projectcreate), create a new project and  give it a name of your choosing.
   - You should be redirected to the [Project Dashboard](https://console.cloud.google.com/home/dashboard) for your new project.

### API Setup

1. Go to [APIs & Services](https://console.cloud.google.com/apis/dashboard)
   - Click on **Enable APIs and Services**.
   - Search for `Google Drive API`, click on it and then click **Enable**.
   - Search for `Google Sheets API`, click on it and then click **Enable**.

### Credentials Setup

#### OAuth 2.0

##### OAuth 2.0 Consent Screen

Before setting up OAuth 2.0 credentials, you need to set up the OAuth consent screen. This is a one-time setup for your project. The consent screen is what users will see when they are asked to authorize your application to access their Google Drive files.

1. Navigate to [APIs & Services](https://console.cloud.google.com/apis/dashboard)
1. Navigate to [OAuth consent screen](https://console.cloud.google.com/auth/overview)
   - Click on `Get Started`
   - Under `App Information` fill in the following fields:
      - `App name` - be aware of the [naming restrictions](https://support.google.com/cloud/answer/15549049#zippy=%2Capp-name)
      - `User support email` - normally your email address
   - Under `Audience`:
      - Select `Internal` if you are using a Google Workspace account
      - Select `External` if you are using a personal account
   - Under `Contact Information` fill in the following fields:
      - `Email address` - same as `User support email`
1. Click on [Audience](https://console.cloud.google.com/auth/audience) and under `Test Users` add the email addresses of the users who will be testing/using your application.

> [!NOTE]
> Be aware of the limitations of an `External` application that is under **`Testing`** mode. The refresh token will expire after 7 days. If you want to have a non-expiring refresh token, you need to publish your application or use an `Internal` application.

##### OAuth 2.0 Credentials

1. Navigate to [APIs & Services](https://console.cloud.google.com/apis/dashboard)
1. Navigate to [Credentials](https://console.cloud.google.com/apis/credentials)
   - Click on `Create Credentials` and select `OAuth client ID`
   - Under `Application type` select `Web application`
   - Under `Authorized redirect URIs` add the following URI `https://developers.google.com/oauthplayground`
   - Click on `Create`
   - Click on `Download JSON` to download the credentials file. This file contains your `Client ID` and `Client Secret`.

> [!WARNING]
>
> Make sure that the redirect URI does not have a trailing `/` at the end.

Here's how you can reference the credentials in your code:

```powershell
$oAuthJson = Get-Content -Path 'C:\path\to\your\credentials.json' |
    ConvertFrom-Json
```

#### Service Account

Using a service account allows you to upload data to folders that are shared with the service account.

In Google Workspace enterprise environments, it is also possible to grant impersonation rights to the service account. With these rights, the service account can act as a user (without OAuth consent screen).

Please check the Google documentation:

- [Create a service account](https://developers.google.com/workspace/guides/create-credentials#create_a_service_account)
- [Assign impersonation rights (domain-wide delegation)](https://developers.google.com/workspace/guides/create-credentials#optional_set_up_domain-wide_delegation_for_a_service_account)

Google offers two types of service user files `.json` and `.p12`. Both types are implemented in this module.

```powershell
Get-GDriveAccessToken -Path D:\service_account.json `
-JsonServiceAccount -ImpersonationUser "user@domain.com"
```

```powershell
$keyData = Get-Content -AsByteStream -Path D:\service_account.p12

Get-GDriveAccessToken -KeyData $KeyData `
    -KeyId 'd41d8cd98f0b24e980998ecf8427e' `
    -ServiceAccountMail test-account@980998ecf8427e.iam.gserviceaccount.com `
    -ImpersonationUser "user@domain.com"
```

## Authorization Code and Refresh Token Retrieval

### Using Powershell

#### Authorization Code

```powershell
$authorizationCode = Request-GDriveAuthorizationCode -ClientID $oAuthJson.web.client_id `
-ClientSecret $oAuthJson.web.client_secret
```

#### Refresh Token

``` powershell
$refreshToken = Request-GDriveRefreshToken -ClientID $oAuthJson.web.client_id `
-ClientSecret $oAuthJson.web.client_secret `
-AuthorizationCode $authorizationCode
```

### Manually

1. Browse to [OAuth 2.0 Playground](https://developers.google.com/oauthplayground)
1. Click the gear in the right-hand corner and place a checkmark next to "Use your own OAuth credentials"
   - Fill in the `OAuth Client ID` and `OAuth Client Secret` fields with the values from your credentials file.
1. Under `Select & authorize APIs` search and select the following scopes:
    1.Under `Drive API v3` select the following scopes:
      - `https://www.googleapis.com/auth/drive`
      - `https://www.googleapis.com/auth/drive.file`
1. Under `Google Sheets API v4` select the following scopes:
      - `https://www.googleapis.com/auth/spreadsheets`
1. Click on `Authorize APIs` and follow the prompts to authorize the application.
1. Click on `Exchange authorization code for tokens` to get the your `Refresh token`.

> [!CAUTION]
> The `Refresh token` can only be retrieved once. If you lose it, you will need to repeat the authorization process to get a new one.

## Access Token Retrieval

> [!TIP]
> The `Access Token` is a mandatory parameter for almost every `GDrive` cmdlet. It is valid for 1 hour and needs to be refreshed every hour.

```powershell
$accessToken = Get-GDriveAccessToken -ClientID $oAuthJson.web.client_id `
    -ClientSecret $oAuthJson.web.client_secret `
    -RefreshToken $refreshToken.refresh_token
```

## Usage

``` powershell
# Upload new file
Add-GDriveItem -AccessToken $access.access_token -InFile D:\SomeDocument.doc -Name SomeDocument.doc

# Search existing file
Find-GDriveItem -AccessToken $access.access_token -Query 'name="test.txt"'

# Update existing file contents
Set-GDriveItemContent -AccessToken $access.access_token -ID $file.id -StringContent 'test file'

# Get ParentFolderID and Modified Time for file
Get-GDriveItemProperty -AccessToken $access.access_token -ID $file.id -Property parents, modifiedTime

# and so on :)
```

## Error Handling

Cmdlets will exit at the first error. However, for instance if "Metadata Upload" succeeded but content upload failed, `UploadID` is returned as `ResumeID` to resume operations later.

You can further leverage `Get-GDriveError` to get more information about the error by catching it in a try/catch block.

``` powershell
# Catch error
try {
    Get-GDriveItemProperty -AccessToken 'error token' -id 'error id'
    }
    catch {
        $err = $_
    }

# Decode error message
 Get-GDriveError $err
```

## Automation

In order to automate operations, you can use the `Get-GDriveAccessToken` cmdlet to retrieve an access token and then use that token.

The functions below are meant to be used as an example only. It is outside of the scope of this module to provide a secure way to store your credentials. You can use the `Protect-String` and `Unprotect-String` functions to convert a string to a secure string and back.

> [!IMPORTANT]
> `ConvertTo-SecureString` and `ConvertFrom-SecureString` use the Windows Data Protection API (DPAPI) to encrypt and decrypt the string.
> This means that the encrypted string can only be decrypted on the **same machine** that it was encrypted on and by the **user account** that it was encrypted by.

``` powershell
function Protect-String {
    <#
        .SYNOPSIS
            Convert String to textual form of SecureString
        .PARAMETER String
            String to convert
        .OUTPUTS
            String
        .NOTES
            Author: MVKozlov
    #>
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [string]
        $String
    )

    PROCESS {
        $String | ConvertTo-SecureString -AsPlainText -Force |
            ConvertFrom-SecureString
    }
}

function Unprotect-String {
    <#
        .SYNOPSIS
            Convert SecureString to string
        .PARAMETER String
            String to convert (textual form of SecureString)
        .PARAMETER SecureString
            SecureString to convert
        .OUTPUTS
            String
        .NOTES
            Author: MVKozlov
    #>
    [CmdletBinding(DefaultParameterSetName = 'String')]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, Position = 0, ParameterSetName = 'String')]
        [string]
        $String,
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, Position = 0, ParameterSetName = 'SecureString')]
        [securestring]
        $SecureString
    )

    PROCESS {
        if ($String) {
            $SecureString = $String | ConvertTo-SecureString
        }

        if ($SecureString) {
        (New-Object System.Net.NetworkCredential '', ($SecureString)).Password
        }
    }
}
```

1. Open Powershell on the machine that will run the script using the specified user account.

    ```powershell
    # Method 1: Using runas command
    runas /user:DOMAIN\username powershell.exe

    # Method 2: Using the `-Credential` parameter of Start-Process
    Start-Process powershell.exe -Credential (Get-Credential)
    ```

1. Use the `Protect-String` function to convert your credentials to a secure string and save it to a file.

    ```powershell
    $credentials = [PSCustomObject]@{
        ClientID = 'clientid'
        ClientSecret = 'clientsecret'
        RefreshToken = 'refreshtoken'
    } | ConvertTo-Json | Protect-String |
        Set-Content -Path C:\Path\somefile -Value $credentials
    ```

1. In your automated script, use the `Unprotect-String` function to retrieve the credentials and use them to get the access token.

    ```powershell
    $credentials = Get-Content -Path C:\path\somefile |
        Unprotect-String | ConvertFrom-Json

    try {
        Write-Host "Getting access token..."

        $token = Get-GDriveAccessToken -ClientID $credentials.ClientID `
            -ClientSecret $credentials.ClientSecret `
            -RefreshToken $credentials.RefreshToken

        Write-Host "Token expires $((Get-Date).AddSeconds($token.expires_in))"
    }
    catch {
        Write-Warning "Error retrieving access token $_"
        Get-GDriveError $_
    }

    if ($token) {
        $Summary = Get-GDriveSummary -AccessToken $token.access_token `
        -ErrorAction Stop
        # [rest of your code here]
    }
    ```
