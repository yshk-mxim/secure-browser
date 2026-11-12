# Security Policy

## Reporting a vulnerability

**Please do not open a public issue or pull request for a security vulnerability.** Public reports expose the problem to everyone before a fix exists.

Report it privately instead, using GitHub's private vulnerability reporting:

1. Go to the [Security tab](https://github.com/yshk-mxim/secure-browser/security) of this repository.
2. Choose **Report a vulnerability**.
3. Describe the issue, how to reproduce it, and its impact.

Only the maintainer and you can see the report. We can discuss it there, work on a fix privately, and publish a security advisory once a fix is available. You'll be credited unless you prefer not to be.

This is a hobby-scale project maintained on a best-effort basis. I aim to acknowledge reports within about a week, but can't promise fix timelines.

## Scope

In scope: flaws in the `secure-browser` script or its configuration that weaken the isolation it promises (VM escape paths, VPN kill-switch bypasses, clipboard or download leaks, unsafe defaults, supply-chain weaknesses in how it fetches or builds components).

Out of scope: vulnerabilities in Apple Container, Google Chrome, WireGuard or Debian themselves. Please report those to their vendors.

## Don't just trust it

This tool reduces risk and does not guarantee protection. Read the code and run your own security scans before relying on it. See the warning in the [README](README.md).
