
# Ansible Install Role

Ansible role used to quickly install various tools and packages with a variety of different package managers.

## Requirements

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

## Role Variables

### General

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_reboot` | bool | no | `false` | If true and the system requires a reboot (i.e., `/var/run/reboot-required` exists), the role will trigger an automatic reboot. If false, it will only notify the user that a reboot is required. |
| `install_path` | str | no | `"{{ ansible_facts['env']['PATH'] }}"` | Base PATH environment variable to use for `install_full_path`. |

### APT

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_apt_autoremove` | bool | no | `false` | If true, runs `apt autoremove` to remove unnecessary packages after package operations. |
| `install_apt_packages` | list | no | — | A list of APT packages to install. |
| `install_apt_update` | bool | no | `false` | If true, runs `apt update` to refresh the package index before installing packages. |
| `install_apt_upgrade` | str | no | `"no"` | Specifies whether or not to perform apt-upgrade and what type (choices: `dist`, `full`, `no`, `safe`, `yes`). |

### Git

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_git` | bool | no | `false` | If true, installs the Git version control system using apt. |
| `install_git_dir` | str | no | `"{{ ansible_facts['env']['HOME'] ~ '/Tools' }}"` | The base directory where tools and git repositories will be installed. |
| `install_git_binaries_dir` | str | no | `"{{ install_git_dir ~ '/Binaries' }}"` | The directory where binaries downloaded from GitHub releases will be placed. |
| `install_git_repos` | dict | no | — | A dictionary of Git repositories to clone, keyed by repository name with values containing `url`, `dest` (optional), and additional `git` module parameters. |
| `install_git_releases` | dict | no | — | A dictionary of GitHub release binaries to download and install, keyed by package name. |
| `install_git_pause` | int | no | `5` | Time in seconds to pause between git clones and downloading binaries from git repos. |

### GoLang

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_golang` | bool | no | `false` | If true, installs the Go programming language from the official upstream source. |
| `install_golang_path` | str | no | `"{{ ansible_facts['env']['HOME'] ~ '/go/bin:/usr/local/go/bin' }}"` | PATH environment variable where Go binaries will be available. |
| `install_golang_packages` | dict | no | — | A dictionary of Go packages to install, keyed by package name with optional `version` values. |
| `install_golang_batch_size` | int | no | `1` | Number of golang packages to install concurrently. If set to 1, packages are installed one at a time. |

### Pipx

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_pipx` | bool | no | `false` | If true, installs pipx for managing isolated Python application environments. |
| `install_pipx_packages` | dict | no | — | A dictionary of Python packages to install via pipx, keyed by package name with optional `source` values. |

### Ruby

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_ruby` | bool | no | `false` | If true, installs the Ruby programming language using apt. |
| `install_ruby_packages` | list | no | — | A list of Ruby gems to install via `gem`. |

### Rust

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_rust` | bool | no | `false` | If true, installs the Rust programming language via rustup and removes any apt-installed versions. |
| `install_rust_path` | str | no | `"{{ ansible_facts['env']['HOME'] ~ '/.cargo/bin' }}"` | PATH directory where Rust/Cargo binaries will be available. |
| `install_rust_packages` | list | no | — | A list of Rust crates to install via `cargo`. |

### Combined Paths

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `install_full_path` | str | no | `"{{ install_path ~ ':' ~ install_golang_path ~ ':' ~ install_rust_path }}"` | Combined PATH environment variable that prepends all other custom PATH variables. |

## Examples

### Install a minimal set of development tools

Installs Git, Go, and Ruby:

```yaml
- hosts: all
  roles:
    - role: ansible-install-role
      vars:
        install_git: true
        install_golang: true
        install_ruby: true
```

### Run apt-update, apt-upgrade, autoremove, and install specific APT packages

Refresh the package index, upgrade the system, install additional packages, and autoremove:

```yaml
- hosts: all
  roles:
    - role: ansible-install-role
      vars:
        install_apt_update: true
        install_apt_upgrade: "full"
        # Reboot if needed after updates
        install_reboot: true
        install_apt_autoremove: true
        install_apt_packages:
          - curl
          - wget
          - vim
          - htop
```

### Install Git repositories and GitHub release binaries

Clone repos and download prebuilt binaries from GitHub releases:

```yaml
- hosts: all
  roles:
    - role: ansible-install-role
      vars:
        install_git: true
        install_git_repos:
          nishang:
            # Use Github Personal Access Token (PAT) and no_log to protect it.
            repo: "{{ 'https://' ~ github_access_token ~ '@github.com/samratashok/nishang.git' }}"
            no_log: true
          trufflehog:
            # Clone beta branch
            repo: "https://github.com/trufflesecurity/trufflehog.git"
            version: "beta"
          seclists:
            # Custom depth, destination, and write using elevated privileges 
            repo: "https://github.com/danielmiessler/SecLists.git"
            depth: 1
            dest: "/usr/share/"
            become: true
            

        install_git_releases:
          chisel:
            repo: "https://github.com/jpillora/chisel.git"
            # Regex filter to download .deb and windows releases
            filter: ".*\\.deb|.*window.*"
          ligolo-ng:
            repo: "https://github.com/nicocha30/ligolo-ng.git"
            # Regex filter to download windows and linux releases
            filter: ".*windows.*|.*linux.*"
          PEASS-ng:
            repo: "https://github.com/peass-ng/PEASS-ng.git"
            # Regex filter to download all releases
            filter: ".*"
```

### Install Go packages with batching

Install Go packages concurrently using batch size of 3:

```yaml
- hosts: all
  roles:
    - role: ansible-install-role
      vars:
        install_golang: true
        install_golang_packages:
          gobuster: "github.com/OJ/gobuster/v3@latest"
          httpx: "github.com/projectdiscovery/httpx/cmd/httpx@latest"
          katana: "github.com/projectdiscovery/katana/cmd/katana@latest"
        # go install 3 packages at a time
        install_golang_batch_size: 3
```

### Install pipx, Ruby gems, and Rust crates

```yaml
- hosts: all
  roles:
    - role: ansible-install-role
      vars:
        install_pipx: true
        install_pipx_packages:
          # Use package shortname
          arjun: "arjun"
          # Use full repo url
          git-dumper: "git+https://github.com/arthaud/git-dumper"
        install_ruby: true
        install_ruby_packages:
          - bundler
          - rails
        install_rust: true
        install_rust_packages:
          - cargo-edit
          - cargo-audit
```

### Custom Path Variables

Custom Path Variables

```yaml
- hosts: all
  become: true
  roles:
    - role: ansible-install-role
      vars:
        # Custom base path to build install_full_path from
        install_path: "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        # Custom directory to clone git repos.
        install_git_dir: "/opt/Tools"
        # Custom directory to download Github binaries to
        install_git_binaries_dir: "{{ ansible_env.HOME }}/dev/bin"
        # Custom GoLang Path
        install_golang_path: "/opt/go/bin"
        # Custom Rust Path
        install_rust_path: "{{ ansible_env.HOME }}/.cargo/bin"
```

## License

MIT
