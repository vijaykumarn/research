Yes — but **`gh` does not currently auto-select between multiple accounts on the same `github.com` host based on the repository**. It can detect the GitHub _host_ from the repository, but when several accounts are authenticated to the same host, it uses the currently active account. GitHub's own docs still recommend `gh auth switch` for that case.  GitHub Docs+1

 There is an open feature request specifically for repository-based account selection, so you're running into a known limitation.  GitHub

 ### The approach I'd recommend: `GH_CONFIG_DIR` \+ `direnv`

 Instead of switching the global `gh` account, give each account its **own `gh` configuration directory**, then automatically select that directory when you enter a repository.

 `gh` supports `GH_CONFIG_DIR`, which controls where it stores its authentication/configuration.  GitHub CLI

 For example:

```
~/.config/gh-personal/
~/.config/gh-work/
~/.config/gh-client/
```

 Log each account into its own config directory:

```
GH_CONFIG_DIR="$HOME/.config/gh-personal" gh auth login
GH_CONFIG_DIR="$HOME/.config/gh-work" gh auth login
GH_CONFIG_DIR="$HOME/.config/gh-client" gh auth login
```

 Check them:

```
GH_CONFIG_DIR="$HOME/.config/gh-personal" gh auth status
GH_CONFIG_DIR="$HOME/.config/gh-work" gh auth status
GH_CONFIG_DIR="$HOME/.config/gh-client" gh auth status
```

 Now there's no concept of a globally "active" account. Each terminal/repository can use the appropriate config.

 ### Automatically select it per repository

 If you use `direnv`, you can put an `.envrc` in each repository:

 For repo-a:

```
export GH_CONFIG_DIR="$HOME/.config/gh-work"
```

 For repo-b:

```
export GH_CONFIG_DIR="$HOME/.config/gh-personal"
```

 For repo-c:

```
export GH_CONFIG_DIR="$HOME/.config/gh-client"
```

 Then when you:

```
cd ~/projects/repo-a
```

 `direnv` automatically sets:

```
GH_CONFIG_DIR=~/.config/gh-work
```

 and:

```
gh pr list
gh pr create
gh issue list
gh repo view
```

 all use the work account.

 When you move to:

```
cd ~/projects/repo-b
```

 the environment changes automatically to the personal account.

 This has a **major advantage over automatically running `gh auth switch`**: `gh auth switch` modifies the shared/global active-account state. With multiple terminals open simultaneously, that can cause one terminal to change the account underneath another terminal. `GH_CONFIG_DIR` avoids that entirely.

 ### Even better: organize repositories by account

 For example:

```
~/src/
├── personal/
│   ├── repo-a/
│   └── repo-b/
├── work/
│   ├── project-x/
│   └── project-y/
└── client/
    ├── project-z/
    └── project-w/
```

 Then you can put a single `.envrc` in each directory:

```
~/src/personal/.envrc
```

```
export GH_CONFIG_DIR="$HOME/.config/gh-personal"
```

 and:

```
~/src/work/.envrc
```

```
export GH_CONFIG_DIR="$HOME/.config/gh-work"
```

 Every repository underneath inherits the correct account.

 ### If you don't want `direnv`

 You can also make your shell automatically determine the account from the Git remote. For example, you could have a wrapper around `gh` that does roughly:

```
remote=$(git remote get-url origin 2>/dev/null)

case "$remote" in
    *github.com/personal-org/*)
        export GH_CONFIG_DIR="$HOME/.config/gh-personal"
        ;;
    *github.com/work-org/*)
        export GH_CONFIG_DIR="$HOME/.config/gh-work"
        ;;
    *github.com/client-org/*)
        export GH_CONFIG_DIR="$HOME/.config/gh-client"
        ;;
esac

command gh "$@"
```

 But I'd choose **`direnv` \+ `GH_CONFIG_DIR`** over this. It's cleaner, doesn't run detection on every `gh` invocation, and works naturally with your terminal's current directory.

 ### One important distinction

 This solves the **GitHub CLI (`gh`) account** problem. Git itself is a separate issue.

 If you're also using:

```
git pull
git push
```

 you should configure Git authentication separately so that the correct identity is selected per repository/account. GitHub documents approaches for multiple accounts using HTTPS credentials or separate SSH keys.  GitHub Docs

 If you're currently using **SSH**, I would actually recommend setting up:

```
github-personal
github-work
github-client
```

 as SSH hosts and configuring each repository's `origin` accordingly. Then Git and `gh` can both be made completely account-independent from each other.

 **For your situation, I'd use:**

```
Repository
    ↓
direnv
    ↓
GH_CONFIG_DIR
    ↓
correct gh account
```

 rather than:

```
Repository
    ↓
gh auth switch
    ↓
global active account
```

 The former is also safe when you have **3 terminals open simultaneously on 3 different repositories**.

 If you tell me whether your repositories use **SSH or HTTPS**, and how you distinguish the three accounts (e.g. `github.com/personal-org/*`, `github.com/company/*`), I can give you a clean Arch/Zsh or Bash setup that automatically handles **both `gh` and `git`** with zero manual switching.