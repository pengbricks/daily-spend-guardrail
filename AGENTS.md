# Declarative Automation Bundles Project

This project uses Declarative Automation Bundles for deployment.

## Prerequisites

Install the Databricks CLI (>= v0.292.0) and verify it with `databricks -v`.

## For AI Agents

Read the `databricks-core`, `databricks-dabs`, `databricks-dbsql`, and `databricks-data-discovery` skills before changing this project. Always use an explicitly selected profile with Databricks CLI commands.

After configuration changes, validate every target with `databricks bundle validate --strict --profile <name>` and the required bundle variables.
