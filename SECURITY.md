# Security Policy

## Supported versions
Only the [latest release](../../releases/latest) receives fixes. Please update before reporting.

## Reporting a vulnerability
Please **do not open a public issue** for a security problem.

Use GitHub's private reporting instead: go to the repository's **Security** tab and choose **Report a vulnerability**. Include the control version, what you observed, and steps to reproduce.

Please never include real subscription keys, SAS tokens, tenant or subscription IDs, or Function App URLs in a report. Redact them.

You can expect an acknowledgment, and a fix or a decision on the report, as time allows. This is a community project maintained by an individual, so there is no guaranteed response time.

## Scope
In scope: the code in this repository (the PCF control and its solution package).

Out of scope: your own Azure resources (Azure Maps account, token-broker Function App, identities, CORS, network settings) and your Power Platform environment. See [Securing the token broker](README.MD#securing-the-token-broker) in the README for guidance on those.
