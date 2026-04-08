# Ghostty Terminal Setup Guide

A complete guide to install and configure Ghostty with Powerlevel10k, autosuggestions, syntax highlighting, and Catppuccin theme.

<img src="https://lh3.googleusercontent.com/d/1qS1hTAB0OrlRUOfbhNymAsQg5TsDs_6S" width="200" alt="0Gosha" />

Covers: **macOS**, **Linux**, and **Windows**.

---

## Step 1: Install Ghostty

### macOS

```bash
brew install --cask ghostty
```

Or download from [https://ghostty.org](https://ghostty.org).

### Linux (Ubuntu/Debian)

```bash
curl -fsSL https://ghostty.org/pubkey.gpg | sudo gpg --dearmor -o /usr/share/keyrings/ghostty-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/ghostty-keyring.gpg] https://apt.ghostty.org stable main" | sudo tee /etc/apt/sources.list.d/ghostty.list
sudo apt update
sudo apt install ghostty
```

### Linux (Fedora)

```bash
sudo dnf copr enable pgdev/ghostty
sudo dnf install ghostty
```

### Linux (Arch)

```bash
sudo pacman -S ghostty
```

### Windows

Download the installer from [https://ghostty.org](https://ghostty.org). Ghostty on Windows is still in early access — check the site for the latest availability. Alternatively, use it via WSL2 (see Windows-specific notes below).

---

## Step 2: Install Zsh (if not already default)

### macOS

Zsh is the default shell. Skip this step.

### Linux

```bash
# Ubuntu/Debian
sudo apt install zsh

# Fedora
sudo dnf install zsh

# Arch
sudo pacman -S zsh
```

Set as default:

```bash
chsh -s $(which zsh)
```

Log out and back in for the change to take effect.

### Windows (WSL2)

Install WSL2 first if you haven't:

```powershell
wsl --install
```

Then inside WSL2 (Ubuntu):

```bash
sudo apt install zsh
chsh -s $(which zsh)
```

---

## Step 3: Install Nerd Font

Required for icons (git branch, folder, OS logo, etc.).

### macOS

```bash
brew install --cask font-meslo-lg-nerd-font
```

### Linux

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts
curl -fLO https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Meslo.tar.xz
tar -xf Meslo.tar.xz
rm Meslo.tar.xz
fc-cache -fv
```

### Windows

Download `Meslo.zip` from [https://github.com/ryanoasis/nerd-fonts/releases](https://github.com/ryanoasis/nerd-fonts/releases). Extract, select all `.ttf` files, right-click → **Install for all users**.

---

## Step 4: Install Oh My Zsh

Same command on all platforms (macOS, Linux, WSL2):

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

---

## Step 5: Install Powerlevel10k Theme

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

---

## Step 6: Install Plugins

### Autosuggestions (gray command suggestions as you type)

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### Syntax Highlighting (green/red command validation)

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

---

## Step 7: Install Catppuccin Theme for Ghostty

### macOS / Linux

```bash
mkdir -p ~/.config/ghostty/themes
curl -o ~/.config/ghostty/themes/catppuccin-mocha https://raw.githubusercontent.com/catppuccin/ghostty/main/themes/catppuccin-mocha
```

### Windows

```powershell
mkdir -Force "$env:APPDATA\ghostty\themes"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/catppuccin/ghostty/main/themes/catppuccin-mocha" -OutFile "$env:APPDATA\ghostty\themes\catppuccin-mocha"
```

---

## Step 8: Configure ~/.zshrc

```bash
nano ~/.zshrc
```

**CRITICAL:** Find the line `source $ZSH/oh-my-zsh.sh`. Place the following two lines **directly ABOVE** it:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

**Delete or comment out** any other `ZSH_THEME=` or `plugins=` lines in the file. Duplicate lines cause conflicts where the first value wins and overrides your config.

Save: `Ctrl+O` → Enter → `Ctrl+X`

---

## Step 9: Configure Ghostty

### macOS / Linux

```bash
mkdir -p ~/.config/ghostty
nano ~/.config/ghostty/config
```

### Windows

Config location: `%APPDATA%\ghostty\config`

```powershell
notepad "$env:APPDATA\ghostty\config"
```

### Config content (all platforms)

```
font-family = MesloLGS NF
font-size = 14
background-opacity = 0.85
theme = catppuccin-mocha
```

#### Platform-specific keybindings

Add one of these depending on your OS:

**macOS:**

```
keybind = super+t=new_tab
```

**Linux / Windows:**

```
keybind = ctrl+shift+t=new_tab
```

---

## Step 10: Restart Ghostty

Fully quit and reopen Ghostty:

- **macOS:** `Cmd+Q`, then reopen
- **Linux:** Close the window, then reopen
- **Windows:** Close and reopen from Start menu

The Powerlevel10k configuration wizard will auto-launch.

---

## Step 11: Powerlevel10k Wizard

Follow the prompts with these recommended choices:

| Prompt | Choice |
|---|---|
| Diamond symbol visible? | Yes (y) |
| Lock symbol visible? | Yes (y) |
| Arrow symbol visible? | Yes (y) |
| Icons fit? | Yes (y) |
| Prompt Style | Rainbow (3) |
| Character Set | Unicode (1) |
| Show current time? | 12-hour format (1) |
| Prompt Separators | Round (1) |
| Prompt Heads | Sharp (1) |
| Prompt Tails | Round (1) |
| Prompt Height | Two lines (2) |
| Prompt Connection | Dotted (2) |
| Prompt Frame | Left (1) |
| Prompt Spacing | Sparse (2) |
| Icons | Many icons (2) |
| Prompt Flow | Concise (1) |
| Transient Prompt | No (n) |
| Save to ~/.p10k.zsh? | Yes (y) |

---

## Usage Tips

- **Autosuggestions:** Start typing and a gray suggestion appears based on your history. Press `→` (right arrow) to accept the full suggestion, or `Ctrl+→` to accept word by word.
- **Syntax Highlighting:** Valid commands appear green, invalid commands appear red as you type.
- **New Tab:** `Cmd+T` (macOS) or `Ctrl+Shift+T` (Linux/Windows).
- **Reconfigure theme:** Run `p10k configure` at any time to change the look.

---

## Troubleshooting

### `p10k: command not found`

Your `ZSH_THEME` line is either missing or placed **after** the `source $ZSH/oh-my-zsh.sh` line. It must be above it. Run:

```bash
grep -n "ZSH_THEME\|source.*oh-my-zsh" ~/.zshrc
```

Make sure the `ZSH_THEME="powerlevel10k/powerlevel10k"` line number is **lower** than the `source` line number.

### Duplicate `plugins=` or `ZSH_THEME=` lines

Only one of each should exist. Comment out extras with `#`. The first uncommented match wins and blocks the rest.

### Icons showing as boxes or question marks

Your terminal is not using the Nerd Font. Set the font in Ghostty config:

```
font-family = MesloLGS NF
```

### Autosuggestions not appearing

Verify the plugin folder exists:

```bash
ls ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions
```

If empty or missing, re-clone (see Step 6). Then verify only one `plugins=` line exists before the `source` line.

### Catppuccin theme not found

The theme file must exist at the correct path:

- macOS/Linux: `~/.config/ghostty/themes/catppuccin-mocha`
- Windows: `%APPDATA%\ghostty\themes\catppuccin-mocha`

Re-run Step 7 if missing.

---

## Disable Temporarily (Keep Everything Installed)

```bash
# Switch to default look
mv ~/.zshrc ~/.zshrc.custom
cp ~/.zshrc.pre-oh-my-zsh ~/.zshrc
```

Restart Ghostty.

## Re-enable

```bash
mv ~/.zshrc.custom ~/.zshrc
```

Restart Ghostty.

---

## Full Uninstall (Remove Everything)

### macOS

```bash
uninstall_oh_my_zsh
rm ~/.config/ghostty/config
rm -rf ~/.config/ghostty/themes
```

### Linux

```bash
uninstall_oh_my_zsh
rm ~/.config/ghostty/config
rm -rf ~/.config/ghostty/themes
```

### Windows (WSL2)

```bash
uninstall_oh_my_zsh
```

```powershell
Remove-Item "$env:APPDATA\ghostty\config"
Remove-Item -Recurse "$env:APPDATA\ghostty\themes"
```

Restart Ghostty. Back to stock defaults.
