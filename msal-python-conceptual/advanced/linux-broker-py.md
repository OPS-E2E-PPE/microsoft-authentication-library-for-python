---
title: Using MSAL Python with an Authentication Broker on Linux
description: MSAL is able to call Microsoft Single Sign-on for Linux, which is a component that ships as a dependency of Intune Portal. This component acts as an authentication broker allowing the users of your app to benefit from integration with accounts known to the broker.
author: ploegert
ms.author: jploegert
ms.service: msal
ms.topic: how-to
ms.date: 06/03/2025
---

# Enable SSSO in native Linux using MSAL Python

Microsoft Authentication Library (MSAL) is a Software Development Kit (SDK) that enables apps to call the Microsoft Single Sign-on to Linux broker, a Linux component that is shipped independent of the Linux Distribution, however it gets installed using a package manager using `sudo apt install microsoft-identity-broker` or `sudo dnf install microsoft-identity-broker`.

This component acts as an authentication broker, allowing the users of your app to benefit from integration with accounts known to Linux - such as the account you signed into your Linux sessions for apps that consume from the broker.

The broker is also bundled as a dependency of applications developed by Microsoft (such as [Company Portal](/mem/intune-service/user-help/enroll-device-linux))). An example of installation of the broker being installed is when a Linux computer is enrolled into a company's device fleet via an endpoint management solution like [Microsoft Intune](/mem/intune/fundamentals/what-is-intune).

## What is a broker

An authentication broker is an application that runs on a user’s machine that manages the authentication handshakes and token maintenance for connected accounts. The Linux operating system uses the Microsoft single sign-on for Linux as its authentication broker. It has many benefits for developers and customers alike, including:

- **Enables Single Sign-On**: enables apps to simplify how users authenticate with Microsoft Entra ID and protects Microsoft Entra ID refresh tokens from exfiltration and misuse
- **Enhanced security.** Many security enhancements are delivered with the broker, without needing to update the application logic.
- **Feature support.** With the help of the broker developers can access rich OS and service capabilities.
- **System integration.** Applications that use the broker plug-and-play with the built-in account picker, allowing the user to quickly pick an existing account instead of reentering the same credentials over and over.
- **Token Protection.** Microsoft single sign-on for Linux ensures that the refresh tokens are device bound and [enables apps](../../advanced/proof-of-possession-tokens.md) to acquire device bound access tokens. See [Token Protection](/azure/active-directory/conditional-access/concept-token-protection).

## How to opt in to use broker?

1. In the MSAL Python library, we've introduced the `enable_broker_on_linux` flag, which enables the broker on both WSL and standalone Linux.
    - If your goal is to enable broker support solely on WSL for Azure CLI, you can consider modifying the Azure CLI app code to activate the `enable_broker_on_wsl` flag exclusively on WSL.
    - If you are writing a cross-platform application, you will also need to use `enable_broker_on_windows`, as outlined in the [Using MSAL Python with Web Account Manager](wam.md) article.
    - You can set any combination of the following opt-in parameters to true:

| Opt-in flag              | If app will run on                | App has registered this as a Desktop platform redirect URI in Azure Portal       |
| ------------------------ | --------------------------------- | -------------------------------------------------------------------------------- |
| enable_broker_on_windows | Windows 10+                       | ms-appx-web://Microsoft.AAD.BrokerPlugin/your_client_id                          |
| enable_broker_on_wsl     | WSL                               | ms-appx-web://Microsoft.AAD.BrokerPlugin/your_client_id                          |
| enable_broker_on_mac     | Mac with Company Portal installed | msauth.com.msauth.unsignedapp://auth                                             |
| enable_broker_on_linux   | Linux with Intune installed       | `https://login.microsoftonline.com/common/oauth2/nativeclient` (MUST be enabled) |

2. As shown in the table above, your application needs to support broker-specific redirect URIs. For Linux, the URL must be:

    ```text
    https://login.microsoftonline.com/common/oauth2/nativeclient
    ```

3. To use the broker, you will need to install the broker-related packages in addition to the core MSAL from PyPI:

    ```python
    pip install msal[broker]>=1.31,<2
    ```

4. Once configured, you can call `acquire_token_interactive` to acquire a token.

    ```python
    result = app.acquire_token_interactive(["User.ReadBasic.All"],
                        parent_window_handle=app.CONSOLE_WINDOW_HANDLE)
    ```

## Parameters for broker support

The following parameters are available to configure broker support in MSAL Python. These parameters can be passed to the `PublicClientApplication` constructor or to the `acquire_token_interactive` method.

| Parameters: | Type | Description |
|-------------|------|-------|
| enable_broker_on_windows | `boolean` | This setting is only effective if your app is running on Windows 10+. This parameter defaults to None, which means MSAL will not utilize a broker. </br></br>`New in MSAL Python 1.25.0.` |
| enable_broker_on_wsl | `boolean` | This setting is only effective if your app is running on WSL. This parameter defaults to None, which means MSAL will not utilize a broker. </br></br>`New in MSAL Python 1.25.0`. |
| enable_broker_on_mac | `boolean` | This setting is only effective if your app is running on Mac with Company Portal installed. This parameter defaults to None, which means MSAL will not utilize a broker. </br></br>`New in MSAL Python 1.31.0`.|
| enable_broker_on_linux | `boolean` | This setting is only effective if your app is running on Linux with Intune installed. This parameter defaults to None, which means MSAL will not utilize a broker. </br></br>`New in MSAL Python 1.33.0`. |
| parent_window_handle | `int` | <i>OPTIONAL</i></br> - If your app does not opt in to use broker, you do not need to provide a parent_window_handle here.</br>- If your app opts in to use broker, parent_window_handle is required.</br>If your app is a GUI app running on Windows or Mac system, you are required to also provide its window handle, so that the sign-in window will pop up on top of your window.</br>- If your app is a console app running on Windows or Mac system, you can use a placeholder `PublicClientApplication.CONSOLE_WINDOW_HANDLE`|

## The fallback behaviors of MSAL Python’s broker support

MSAL will either error out, or silently fallback to non-broker flows.

1. MSAL will ignore the enable_broker_… and bypass broker on those auth flows that are known to be NOT supported by broker. This includes ADFS, B2C, etc.. For other “could-use-broker” scenarios, please see below.

2. MSAL errors out when app developer opted-in to use broker but a direct dependency “mid-tier” package is not installed. Error message guides app developer to declare the correct dependency msal[broker]. We error out here because the error is actionable to app developers.

3. MSAL silently “deactivates” the broker and fallback to non-broker, when opted-in, dependency installed yet failed to initialize. We anticipate this would happen on a device whose OS is too old or the underlying broker component is somehow unavailable. There is not much an app developer or the end user can do here. Eventually, the conditional access policy shall force the user to switch to a different device.

4. MSAL errors out when broker is opted in, installed, initialized, but subsequent token request(s) failed.

>[!IMPORTANT]
>If broker-related packages are not installed and you will try to use the authentication broker, you will get an error: `ImportError: You need to install dependency by: pip install "msal[broker]>=1.31,<2"`.

>[!IMPORTANT]
>If you are writing a cross-platform application, you will also need to use `enable_broker_on_windows`, as outlined in the [Using MSAL Python with Web Account Manager](wam.md) article.

>[!NOTE]
>The `parent_window_handle` parameter is required even though on Linux it is not used. For GUI applications, the login prompt location will be determined ad-hoc and currently cannot be bound to a specific window. In a future update, this parameter will be used to determine the _actual_ parent window.

## Token caching

The authentication broker handles refresh and access token caching. You do not need to set up custom caching.