---
layout: default
title:  "Development Environment Setup: My Checklist"
date:   2025-11-15 00:00:00
categories: Development DevOps Productivity
---

Setting up a new development machine takes me about 2 hours now. It used to take 2 days. The difference: a checklist and automation scripts.

Here's my complete development environment setup.

## Essential Tools

### Package Manager

**macOS:**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**Windows:**
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
```

### Terminal

**macOS/Linux:**
- iTerm2 or Alacritty
- Zsh with Oh My Zsh
- Powerlevel10k theme

```bash
# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install Powerlevel10k
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

**Windows:**
- Windows Terminal
- PowerShell 7

### Version Control

```bash
# Git
brew install git  # macOS
choco install git  # Windows

# Configuration
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

### Code Editor

**VS Code:**
```bash
brew install --cask visual-studio-code  # macOS
choco install vscode  # Windows
```

**Essential Extensions:**
- ESLint
- Prettier
- GitLens
- Remote - SSH
- Docker
- Language-specific (Python, Go, etc.)

### Programming Languages

**Node.js:**
```bash
# Use nvm for version management
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install --lts
nvm use --lts
```

**Python:**
```bash
brew install python3  # macOS
choco install python  # Windows

# pyenv for version management
curl https://pyenv.run | bash
```

**Go:**
```bash
brew install go  # macOS
choco install golang  # Windows
```

### Database Tools

```bash
# PostgreSQL
brew install postgresql@15  # macOS
choco install postgresql  # Windows

# MySQL
brew install mysql  # macOS
choco install mysql  # Windows

# Redis
brew install redis  # macOS
choco install redis  # Windows

# Database GUI
brew install --cask dbeaver-community  # macOS
```

### Docker

```bash
brew install --cask docker  # macOS
choco install docker-desktop  # Windows
```

### API Development

```bash
# Postman or Insomnia
brew install --cask postman  # macOS
choco install postman  # Windows
```

### Cloud CLI Tools

```bash
# AWS CLI
brew install awscli  # macOS
choco install awscli  # Windows

# Azure CLI
brew install azure-cli  # macOS
choco install azure-cli  # Windows

# Google Cloud
brew install --cask google-cloud-sdk  # macOS
```

## My .zshrc Configuration

```bash
# Aliases
alias ll='ls -alh'
alias g='git'
alias gst='git status'
alias gc='git commit'
alias gp='git push'
alias gl='git pull'
alias gco='git checkout'
alias gcb='git checkout -b'

# Docker aliases
alias d='docker'
alias dc='docker compose'
alias dcu='docker compose up -d'
alias dcd='docker compose down'

# Kubernetes
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'

# Quick navigation
alias projects='cd ~/Projects'
alias dotfiles='cd ~/dotfiles'

# Custom functions
function mkcd() {
    mkdir -p "$@" && cd "$@"
}

function gac() {
    git add . && git commit -m "$1"
}
```

## Development Directories

```bash
mkdir -p ~/Projects
mkdir -p ~/Projects/personal
mkdir -p ~/Projects/work
mkdir -p ~/dotfiles
mkdir -p ~/bin
```

## SSH Keys

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"

# Start ssh-agent
eval "$(ssh-agent -s)"

# Add key
ssh-add ~/.ssh/id_ed25519

# Copy public key
cat ~/.ssh/id_ed25519.pub | pbcopy  # macOS
cat ~/.ssh/id_ed25519.pub | clip  # Windows
```

## Dotfiles Management

```bash
# Initialize git repo for dotfiles
cd ~
mkdir dotfiles
cd dotfiles
git init

# Add files
ln -s ~/dotfiles/.zshrc ~/.zshrc
ln -s ~/dotfiles/.gitconfig ~/.gitconfig
ln -s ~/dotfiles/.vimrc ~/.vimrc
```

## Productivity Tools

```bash
# tmux (terminal multiplexer)
brew install tmux

# fzf (fuzzy finder)
brew install fzf

# ripgrep (fast grep)
brew install ripgrep

# jq (JSON processor)
brew install jq

# htop (system monitor)
brew install htop
```

## My Complete Setup Script (macOS)

```bash
#!/bin/bash

# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Development tools
brew install git
brew install node
brew install python3
brew install go
brew install docker
brew install postgresql@15
brew install redis
brew install tmux
brew install fzf
brew install ripgrep
brew install jq
brew install htop

# Applications
brew install --cask visual-studio-code
brew install --cask iterm2
brew install --cask postman
brew install --cask dbeaver-community

# Configure git
git config --global user.name "Your Name"
git config --global user.email "your@example.com"
git config --global init.defaultBranch main

# Create directories
mkdir -p ~/Projects/{personal,work}
mkdir -p ~/dotfiles
mkdir -p ~/bin

echo "Setup complete!"
```

## Quick Reference

**Check versions:**
```bash
node --version
python3 --version
go version
docker --version
git --version
```

**Update everything:**
```bash
# macOS
brew update && brew upgrade

# Windows
choco upgrade all
```

## Resources

- [Homebrew](https://brew.sh/)
- [Chocolatey](https://chocolatey.org/)
- [Oh My Zsh](https://ohmyz.sh/)
- [VS Code](https://code.visualstudio.com/)

---

*Questions about dev environment setup? [Let me know](mailto:jordan@jordananderson.us).*
