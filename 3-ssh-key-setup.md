## Appendix: SSH key setup

> [!NOTE]
> Most users don't need this. The `gh auth login` command handles authentication for typical workflows.

### When SSH keys are useful

- **Multiple GitHub accounts** (e.g. personal + work). SSH config lets you specify which key to use per repo.
- **Servers without `gh` CLI.** SSH works anywhere once configured.
- **Tokens keep expiring.** SSH keys don't expire unless you revoke them.

### Setup steps

#### 1. Create key

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

#### 2. Add to ssh-agent

**macOS:**
```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

**Windows:** Start the OpenSSH Agent service first, then:
```bash
ssh-add ~/.ssh/id_ed25519
```

#### 3. Add to GitHub

Copy your public key to clipboard:

**macOS:**
```bash
cat ~/.ssh/id_ed25519.pub | pbcopy
```

**Windows:**
```bash
cat ~/.ssh/id_ed25519.pub | clip
```

Then paste at [GitHub SSH settings](https://github.com/settings/ssh/new).

#### 4. Test connection

```bash
ssh -T git@github.com
```

You should see: "Hi username! You've successfully authenticated..."

### Multiple accounts with SSH config

Create `~/.ssh/config` to manage multiple GitHub accounts:

```
# Personal account
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

# Work account
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

Then clone work repos using the custom host:

```bash
git clone git@github-work:company/repo.git
```

**Docs:** [Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
