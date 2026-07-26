# Azure Infrastructure

> **Status: stub.** Scope is agreed; rules are not yet written. Not normative until this notice is removed.

Defines the Azure footprint the Databricks platform runs on: what resources exist, how they are organized, and who owns them. Infrastructure is provisioned as code; portal-created resources are a defect.

## Scope (planned)

- Subscription and resource group layout per environment
- Databricks workspace provisioning and Unity Catalog metastore attachment
- Storage accounts (ADLS Gen2) backing catalogs and external locations
- Networking: VNet injection, private endpoints, egress control
- Entra ID: groups, service principals, SCIM provisioning into Databricks
- Infrastructure-as-code standard (Terraform) and state management

## Rules

To be written.

## Sharp edges

To be written.

## Checklist

To be written.
