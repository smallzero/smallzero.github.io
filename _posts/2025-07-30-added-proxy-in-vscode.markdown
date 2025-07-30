---
layout: post
title: "How to Configure Proxy Settings in VS Code"
categories: Programming
excerpt: Sometimes you need to configure Visual Studio Code to use a specific proxy, especially to bypass your system's default proxy settings. This can be useful for development, testing, or accessing specific networks. Here is a sample configuration to set up a local proxy directly within VS Code.
---
# How to Configure Proxy Settings in VS Code

Sometimes you need to configure Visual Studio Code to use a specific proxy, especially to bypass your system's default proxy settings. This can be useful for development, testing, or accessing specific networks.

Here is a sample configuration to set up a local proxy directly within VS Code.

---

## Configuration Steps

You can add these settings to your user settings or your workspace's `settings.json` file. For a project-specific setup, create or open the `.vscode/settings.json` file in your project's root directory.

**File Location:** `.vscode/settings.json`

```json
{
    "http.proxy": "http://localhost:8080",
    "http.proxySupport": "on",
    "http.proxyStrictSSL": false,
    "http.noProxy": "localhost,127.0.0.1"
}
```
