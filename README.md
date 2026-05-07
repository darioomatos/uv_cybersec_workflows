# ⚡ uv + Cybersecurity Workflows

> **TL;DR:** Migrei meu tooling ofensivo para [`uv`](https://github.com/astral-sh/uv) e nunca mais tive `pip install` quebrando meu sistema. Se você usa Arch, Manjaro ou qualquer distro rolling-release para pentesting, leia isso.

---

## O Problema

Se você faz pentesting em Linux, provavelmente já viu isso:

```
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, either use a distribution package...
```

Ou pior — instalou com `--break-system-packages` e quebrou o `pacman`.

O ecossistema Python em cybersec é caótico por natureza:

| Problema | Exemplo real |
|----------|-------------|
| Conflito de dependências | Impacket vs NetExec vs CrackMapExec |
| AUR package quebrado pós-update | `python-impacket` no Arch após Python 3.13 |
| PyO3/Rust build failures | Qualquer tool com bindings Rust no rolling-release |
| Tool exige Python antigo | NetExec requer `<= 3.13` por causa do PyO3 |
| Sistema Python corrompido | `sudo pip install` em 2019 🪦 |

Ferramentas afetadas constantemente:

- **NetExec** / CrackMapExec forks
- **Impacket** (secretsdump, psexec, etc.)
- **Certipy** (AD CS attacks)
- **BloodyAD**
- **Pypykatz**
- **Coercer**
- **LSASSY**

---

## A Solução: uv

[`uv`](https://github.com/astral-sh/uv) é um package manager Python escrito em **Rust** pela Astral (mesma equipe do Ruff).

Substitui de vez:

```
pip + pipx + virtualenv + pyenv + poetry
```

Em um único binário. Absurdamente rápido.

### Benchmark real — instalando NetExec (95 pacotes)

```
Installed 95 packages in 118ms
```

Sim. 118 milissegundos. Com pip tradicional isso levaria 30-60 segundos.

---

## Instalação

### Arch / Manjaro

```bash
sudo pacman -S uv
```

### Universal (qualquer distro)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## Por Que uv Resolve os Problemas de Cybersec

### 1. Ambientes Completamente Isolados

Cada tool tem seu próprio ambiente. Zero contaminação do sistema.

```bash
uv tool install certipy-ad
uv tool install impacket
uv tool install bloodyad
```

Cada uma roda no seu próprio venv gerenciado automaticamente. Sem conflito.

### 2. Múltiplas Versões de Python em Paralelo

```bash
uv python install 3.11
uv python install 3.12
uv python install 3.13
```

Precisa rodar uma tool que exige Python 3.11 enquanto seu sistema está no 3.14?

```bash
uv tool install --python 3.11 git+https://github.com/tool/antigo
```

Feito. Sem pyenv, sem gambiarras.

### 3. Execução Temporária Sem Instalação (como `npx`)

Perfeito para labs e CTFs onde você não quer sujar o ambiente:

```bash
uvx impacket-smbserver . -smb2support
uvx certipy find -u user@domain -p pass -dc-ip 10.10.10.1
uvx netexec smb 10.10.10.0/24
```

Roda direto, sem instalar globalmente.

---

## Workflows Práticos

### CVE Research Lab (exemplo: cvehunter)

```bash
uv init cvehunter
cd cvehunter

uv add requests rich nvdlib aiohttp
uv run cvehunter.py --cve CVE-2025-29927 --github
```

Zero configuração de venv manual. O `uv run` resolve tudo automaticamente.

### Active Directory Tooling

```bash
# NetExec com Python específico (por compatibilidade PyO3)
uv tool install --python 3.13 \
  git+https://github.com/Pennyw0rth/NetExec

# AD CS attacks
uv tool install certipy-ad

# LDAP attacks
uv tool install bloodyad

# Pass-the-hash, secretsdump, etc.
uv tool install impacket
```

### Malware Analysis Lab

```bash
uv init malware-lab
cd malware-lab

uv add pefile yara-python capstone unicorn
uv run analyzer.py sample.exe
```

### Pentest Scripting Rápido

```bash
# Projeto temporário para um engagement
uv init engagement-2025
cd engagement-2025

uv add impacket requests pwntools
uv run exploit.py
```

---

## Comparativo: Antes vs Depois

| Situação | pip / pipx | uv |
|----------|-----------|-----|
| Instalar 95 pacotes | ~45 segundos | **118ms** |
| Múltiplas versões Python | pyenv + virtualenv | `uv python install` |
| Isolar tools | pipx (limitado) | `uv tool install` |
| Rodar sem instalar | ❌ | `uvx tool` |
| Conflito de dependências | Problema seu | Resolvido automaticamente |
| Arch rolling-release | Quebra constantemente | Nunca toca o sistema |
| Projeto novo | `python -m venv .venv && source .venv/bin/activate && pip install` | `uv init && uv add` |

---

## Estrutura de Diretórios Recomendada

```
~/
├── labs/           # CTFs e labs temporários
├── tools/          # Tools instaladas com uv tool
├── ad/             # Scripts e research de Active Directory
├── c2/             # Frameworks C2 e configs
├── research/       # CVE research, exploits, PoCs
├── malware/        # Análise de malware
└── automation/     # Scripts de automação e pipelines
```

Cada subprojeto tem seu `uv init` próprio com dependências isoladas.

---

## Comandos Essenciais

```bash
# Gerenciar Python
uv python install 3.13
uv python list

# Gerenciar tools globais
uv tool install certipy-ad
uv tool list
uv tool upgrade netexec
uv tool uninstall netexec

# Projetos
uv init projeto
uv add requests rich
uv run script.py

# Executar sem instalar
uvx impacket-smbserver
uvx certipy

# Dentro de um projeto existente (instala deps do pyproject.toml)
uv sync
```

---

## Integração com Neovim (Bônus)

`uv` funciona perfeitamente com LSP no Neovim:

- **Pyright / BasedPyright** detecta o `.venv` do `uv` automaticamente
- **Ruff** (também da Astral) para linting/formatting
- Autocomplete e análise estática funcionam out-of-the-box

```bash
# No projeto
uv init meu-tool
uv add requests

# Neovim já detecta o .venv e oferece autocomplete correto
nvim script.py
```

---

## Por Que Parei de Usar AUR para Ferramentas Ofensivas

AUR é ótimo para muita coisa. Para offensive tooling em Python, é uma dor constante:

- Updates do Python no Arch quebram bindings PyO3
- PKGBUILDs desatualizados
- Dependências conflitantes entre tools
- Qualquer `pacman -Syu` pode ser um surpresa

Com `uv`, essas tools não tocam o sistema. O `pacman -Syu` fica sem drama.

---

## Referências

- [uv — GitHub](https://github.com/astral-sh/uv)
- [uv Docs](https://docs.astral.sh/uv/)
- [Ruff](https://github.com/astral-sh/ruff) — linter Python da mesma equipe
- [NetExec](https://github.com/Pennyw0rth/NetExec)
- [Certipy](https://github.com/ly4k/Certipy)
- [Impacket](https://github.com/fortra/impacket)
- [BloodyAD](https://github.com/CravateRouge/bloodyAD)
- [BasedPyright](https://github.com/DetachHead/basedpyright)

---

*Testado em Arch Linux com kernel 6.x e Python 3.13/3.14. Pull requests e discussões bem-vindos.*
