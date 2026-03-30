---
title: How to clone a GitLab repository with SSH
---

# How to clone a GitLab repository with SSH

## Prerequisites

- You have created an SSH key and added the public key to your GitLab profile

## Find the SSH URL

1. Navigate to your GitLab project in the browser
2. Click the Clone button and copy the SSH URL (it looks like `git@gitlab.com:group/project.git`)

## Clone the repository

```bash
git clone git@gitlab.com:group/project.git
```

If you have multiple keys or hosts, ensure your `~/.ssh/config` is set up appropriately.

```sshconfig
Host gitlab.com
  HostName gitlab.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```
