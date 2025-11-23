# Managing Multiple GitHub Accounts on macOS: Complete Setup Guide

## Overview
This guide will help you maintain complete separation between your personal and work GitHub accounts, preventing commits from being attributed to the wrong account.

**The Strategy:**
- Use separate SSH keys for each GitHub account
- Configure SSH to automatically use the correct key based on your repository location
- Set up local Git configurations for each repository type
- Use a directory-based approach to enforce separation

---

## Step 1: Generate SSH Keys for Each Account

SSH keys authenticate you to GitHub without needing to enter passwords. You'll create one key per account.

### 1.1 Open Terminal

Press `Command + Space` to open Spotlight Search, type "Terminal", and press Enter.

### 1.2 Generate SSH Key for Personal Account

Replace `your.personal.email@gmail.com` with your actual personal email:

```bash
ssh-keygen -t ed25519 -C "your.personal.email@gmail.com" -f ~/.ssh/id_ed25519_personal
```

When prompted to "Enter passphrase", you can press Enter to skip it (or add one for extra security). Press Enter twice if skipping.

### 1.3 Generate SSH Key for Work Account

Replace `your.work.email@company.com` with your actual work email:

```bash
ssh-keygen -t ed25519 -C "your.work.email@company.com" -f ~/.ssh/id_ed25519_work
```

Again, press Enter twice for no passphrase (or add one).

### 1.4 Verify Both Keys Were Created

```bash
ls -la ~/.ssh/id_ed25519*
```

You should see four files listed:
- `id_ed25519_personal` (private key)
- `id_ed25519_personal.pub` (public key)
- `id_ed25519_work` (private key)
- `id_ed25519_work.pub` (public key)

---

## Step 2: Add SSH Keys to SSH Agent

The SSH agent manages your keys. This step loads them into memory so Git can use them.

### 2.1 Start SSH Agent (if not already running)

```bash
eval "$(ssh-agent -s)"
```

You should see output like: `Agent pid 12345`

### 2.2 Create or Edit SSH Config File

We'll create a configuration file that tells SSH which key to use for which GitHub account.

First, check if the file exists:

```bash
cat ~/.ssh/config
```

If you see "No such file or directory", that's fine. We'll create it.

### 2.3 Open SSH Config in a Text Editor

```bash
nano ~/.ssh/config
```

This opens the Nano text editor. If the file is empty, that's expected.

### 2.4 Add the SSH Configuration

Copy and paste the following configuration into Nano:

```
# Personal GitHub Account
Host github.com-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  AddKeysToAgent yes
  IdentitiesOnly yes

# Work GitHub Account
Host github.com-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  AddKeysToAgent yes
  IdentitiesOnly yes
```

### 2.5 Save and Exit Nano

Press `Control + O` (the letter O, not zero), then press Enter to save.
Then press `Control + X` to exit.

---

## Step 3: Add Public Keys to GitHub

Now you need to tell GitHub about your public keys so it can recognize your machine.

### 3.1 Get Your Personal Public Key

```bash
cat ~/.ssh/id_ed25519_personal.pub
```

This will display your public key (a long string). Select and copy it (Command + C).

### 3.2 Add to Personal GitHub Account

1. Go to https://github.com/settings/keys
2. Click "New SSH key"
3. Title: "MacBook Personal" (or any descriptive name)
4. Key type: "Authentication Key"
5. Paste your public key in the "Key" field
6. Click "Add SSH key"

### 3.3 Get Your Work Public Key

```bash
cat ~/.ssh/id_ed25519_work.pub
```

Copy this key (Command + C).

### 3.4 Add to Work GitHub Account

1. Log in to your work GitHub account at https://github.com/settings/keys
2. Click "New SSH key"
3. Title: "MacBook Work" (or any descriptive name)
4. Key type: "Authentication Key"
5. Paste your public key
6. Click "Add SSH key"

---

## Step 4: Set Up Directory Structure

Create a clear separation by putting repositories in different folders.

```bash
mkdir -p ~/Projects/Personal
mkdir -p ~/Projects/Work
```

This creates two directories where you'll keep your repositories organized.

---

## Step 5: Configure Git for Each Repository

This is the critical step. Each repository will have its own Git configuration for user name and email.

### 5.1 Clone or Initialize a Personal Repository

When cloning, use the custom SSH host we configured. For example:

```bash
cd ~/Projects/Personal
git clone git@github.com-personal:your-personal-username/your-personal-repo.git
```

Notice we use `github.com-personal` instead of `github.com`.

### 5.2 Configure the Personal Repository

Navigate into the cloned repository:

```bash
cd your-personal-repo
```

Set the local Git configuration for this repository:

```bash
git config user.name "Your Personal Name"
git config user.email "your.personal.email@gmail.com"
```

### 5.3 Verify Configuration (Personal)

```bash
git config user.name
git config user.email
```

You should see your personal name and email.

### 5.4 Clone or Initialize a Work Repository

```bash
cd ~/Projects/Work
git clone git@github.com-work:your-work-username/your-work-repo.git
```

Notice we use `github.com-work` instead of `github.com`.

### 5.5 Configure the Work Repository

Navigate into the cloned repository:

```bash
cd your-work-repo
```

Set the local Git configuration:

```bash
git config user.name "Your Work Name"
git config user.email "your.work.email@company.com"
```

### 5.6 Verify Configuration (Work)

```bash
git config user.name
git config user.email
```

You should see your work name and email.

---

## Step 6: Automate Configuration with Conditional Includes (Optional but Recommended)

This advanced step prevents mistakes by automatically setting the correct user configuration based on which directory you're in.

### 6.1 Create Global Git Config with Conditional Include

Create a file for personal Git settings:

```bash
nano ~/.gitconfig-personal
```

Add:

```
[user]
  name = Your Personal Name
  email = your.personal.email@gmail.com
```

Press Control + O, Enter, then Control + X to save and exit.

### 6.2 Create File for Work Git Settings

```bash
nano ~/.gitconfig-work
```

Add:

```
[user]
  name = Your Work Name
  email = your.work.email@company.com
```

Press Control + O, Enter, then Control + X to save and exit.

### 6.3 Update Your Main Git Config

```bash
nano ~/.gitconfig
```

Find or add conditional includes at the end:

```
[includeIf "gitdir:~/Projects/Personal/"]
  path = ~/.gitconfig-personal

[includeIf "gitdir:~/Projects/Work/"]
  path = ~/.gitconfig-work
```

If a `[user]` section already exists in your main `.gitconfig`, remove it or comment it out (add `#` before the lines).

Press Control + O, Enter, then Control + X to save and exit.

### 6.4 Test the Conditional Includes

```bash
cd ~/Projects/Personal/your-personal-repo
git config user.email
```

This should show your personal email.

```bash
cd ~/Projects/Work/your-work-repo
git config user.email
```

This should show your work email.

---

## Step 7: Testing Your Setup

### 7.1 Test SSH Connection for Personal Account

```bash
ssh -T git@github.com-personal
```

You should see: `Hi your-personal-username! You've successfully authenticated...`

### 7.2 Test SSH Connection for Work Account

```bash
ssh -T git@github.com-work
```

You should see: `Hi your-work-username! You've successfully authenticated...`

### 7.3 Make a Test Commit

In a personal repository:

```bash
cd ~/Projects/Personal/your-personal-repo
echo "# Test" >> README.md
git add README.md
git commit -m "Test commit"
git log -1 --format="%an <%ae>"
```

This should display your personal name and email.

In a work repository:

```bash
cd ~/Projects/Work/your-work-repo
echo "# Test" >> README.md
git add README.md
git commit -m "Test commit"
git log -1 --format="%an <%ae>"
```

This should display your work name and email.

---

## Step 8: Handling Existing Repositories

If you already have repositories cloned in random locations, here's how to fix them:

### 8.1 For a Personal Repository in the Wrong Location

```bash
cd /path/to/your/personal/repo
```

Change the remote URL:

```bash
git remote set-url origin git@github.com-personal:your-personal-username/repo-name.git
```

Update the local Git config:

```bash
git config user.name "Your Personal Name"
git config user.email "your.personal.email@gmail.com"
```

### 8.2 For a Work Repository in the Wrong Location

```bash
cd /path/to/your/work/repo
```

Change the remote URL:

```bash
git remote set-url origin git@github.com-work:your-work-username/repo-name.git
```

Update the local Git config:

```bash
git config user.name "Your Work Name"
git config user.email "your.work.email@company.com"
```

---

## Step 9: Fixing Previously Misattributed Commits (Advanced)

If you have commits with the wrong author, you can rewrite the Git history.

### WARNING: Only do this if you haven't pushed to a shared repository or coordinate with your team first.

Use the following command to rewrite history:

```bash
git filter-branch --env-filter '
OLD_EMAIL="wrong.email@example.com"
CORRECT_NAME="Your Correct Name"
CORRECT_EMAIL="correct.email@example.com"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' -f -- --all
```

Then force push:

```bash
git push -f origin --all
```

**Only use this if necessary and after backing up your repository.**

---

## Step 10: Troubleshooting

### Problem: "Permission denied (publickey)"

**Solution:** Make sure SSH keys are properly added:

```bash
ssh-add -l
```

If you don't see your keys, add them:

```bash
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

### Problem: Git still uses the wrong email

**Solution:** Check which Git config is being used:

```bash
cd /path/to/repo
git config --list
```

Ensure you're in the correct directory (Personal vs Work) and that conditional includes are set up correctly.

### Problem: SSH Test Connection Shows "Bad Port" or Connection Error

**Solution:** Check your SSH config file syntax. Open it and ensure there are no typos:

```bash
cat ~/.ssh/config
```

The file should have proper spacing and no extra characters.

---

## Summary: The Separation of Concerns

Your final setup should look like this:

```
~/Projects/
├── Personal/
│   ├── personal-repo-1/
│   │   └── .git/config (user.email = personal@email.com)
│   └── personal-repo-2/
│       └── .git/config (user.email = personal@email.com)
│
└── Work/
    ├── work-repo-1/
    │   └── .git/config (user.email = work@company.com)
    └── work-repo-2/
        └── .git/config (user.email = work@company.com)

~/.ssh/
├── id_ed25519_personal (personal SSH key)
├── id_ed25519_personal.pub
├── id_ed25519_work (work SSH key)
├── id_ed25519_work.pub
└── config (routes github.com-personal and github.com-work to correct keys)

~/.gitconfig (with conditional includes based on directory)
```

---

## Key Points to Remember

1. **Always clone using the correct host alias** (`github.com-personal` or `github.com-work`)
2. **Keep repositories organized** in ~/Projects/Personal and ~/Projects/Work
3. **The conditional includes prevent mistakes** by automatically setting the correct user
4. **Test after setup** to ensure everything works
5. **When in doubt, check your location** (`pwd` command) before making commits

This system ensures you'll never accidentally commit to the wrong account!
