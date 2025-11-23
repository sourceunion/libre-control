# Configuration Assets

> [!WARNING]
> Never commit production secrets (API Keys, Private Keys, Real Passwords) to this directory. Use environment variables (`.env` files) or a Secret Manager for sensitive data. The files here should serve only as **templates** (e.g., `config.example.yaml`).

This directory holds the externalized configuration templates and environment definitions for the LibreControl ecosystem.

## Purpose

Following the **Twelve-Factor App** methodology, configuration (which varies between deploys) is strictly separated from code. This directory contains the blueprints for:

- **C2 Profiles**: Defining how traffic should look (e.g., imitating Amazon or Google traffic).
- **Server Configs**: Database connection strings and listening ports (usually overridden by Environment Variables in production).