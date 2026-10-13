# Security policy

## Reporting a vulnerability

Email vivek@fortunaterra.com with the subject line "SECURITY: <repository name>". Please do not open a public issue for a suspected vulnerability.

Include:

- the repository and the file or section affected
- what you observed and what you expected
- steps to reproduce, or the prompt or input that triggers it
- your assessment of the impact

You may also use GitHub's private vulnerability reporting for this repository if it is enabled.

## Scope

In scope: content in this repository that could cause harm when a user installs or runs it. Examples: skill instructions that direct an agent to take an unsafe or unrequested action, a manifest that points somewhere other than it should, a committed secret, or a dependency or script that does something its README does not say.

Out of scope: vulnerabilities in Claude Code itself or in the model (report those to Anthropic), and findings that depend on a user having granted an agent permissions beyond what this repository asks for.

## Supported versions

Only the latest commit on the default branch is supported.
