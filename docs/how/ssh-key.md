---
title: How to generate an SSH key pair
---

# How to generate an SSH key pair

Follow these steps to create a new SSH key and add it to your SSH agent.

## Generate key

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Accept the defaults and set a passphrase when prompted.

## Start the agent and add your key

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

## Copy the public key

```bash
pbcopy < ~/.ssh/id_ed25519.pub   # macOS
# xclip -sel clip < ~/.ssh/id_ed25519.pub  # Linux
```

Paste the key into your Git hosting provider (e.g., GitHub or GitLab).
