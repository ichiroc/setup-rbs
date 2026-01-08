# Rails RBS/Steep Setup Guide for Claude Code

A Claude Code command for introducing RBS / Steep to Rails applications.

## Overview

This provides a detailed guide as a Claude Code skill to help you gradually introduce type definitions (RBS) and type checker (Steep) to your Rails projects.

## Installation

```sh
curl -sSL https://raw.githubusercontent.com/ichiroc/setup-rbs/main/install.sh | sh
```

## Usage

After installation, you can use this skill in Claude Code:

```sh
claude
```

Then execute the command:

```
/rails:setup-rbs
```

This will guide you through the entire process of setting up RBS and Steep for your Rails application, focusing on the `app/models` directory.
