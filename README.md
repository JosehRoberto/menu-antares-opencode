# Menu de Comandos Antares e Mira para OpenCode

Gera automaticamente os comandos `/antares-*` e `/mira-*` no menu do OpenCode a partir dos skills instalados pelo Antares e pelo [Mira Animator](https://github.com/sandeco/mira-animator).

> **Por que este repositório cobre os dois prefixos?**
> A instalação do Antares traz o Mira junto (a pasta `.agents/skills/` contém `antares-*`, `mira-*` e `_shared/`). Um único script gera os comandos dos dois frameworks, em vez de rodar `menu-mira-opencode` e este script separadamente na mesma pasta.

A partir da versão 1, a lógica deixou de ser um script shell embutido no README e passou a ser um **núcleo Python único e portátil** (`menu-antares.py`), com wrappers finos por sistema operacional. Isso torna a execução **determinística** em Linux, macOS e Windows.

## Pré-requisitos

- **Python 3** com o módulo **PyYAML**.
- **Bash** (Linux/macOS) ou **PowerShell** (Windows 10/11).
- **git** (para o fluxo via prompt ou clonagem).

### Instalar o PyYAML por sistema

| Sistema | Comando |
|---|---|
| Ubuntu / Debian / Mint | `sudo apt install python3-yaml` ou `python3 -m pip install --user pyyaml` |
| Fedora / RHEL | `sudo dnf install python3-pyyaml` |
| Arch | `sudo pacman -S python-yaml` |
| macOS | `python3 -m pip install --user pyyaml` |
| Windows | `py -3 -m pip install pyyaml` |

Verificação: `python3 -c "import yaml"` (Linux/macOS) ou `py -3 -c "import yaml"` (Windows).

> **PEP 668:** em distribuições recentes (Ubuntu 24.04, Debian 12+, Fedora) o `pip install` global pode ser recusado. Use o pacote do sistema, `--user` ou um ambiente virtual (`venv`).

## Instalação

### Opção A — via prompt no OpenCode (recomendada)

Abra o OpenCode na raiz do projeto onde o Antares (e o Mira) foi instalado e cole este prompt:

```text
Instale os comandos Antares e Mira no menu do OpenCode usando a versão fixada v1.0.0 do repositório https://github.com/JosehRoberto/menu-antares-opencode.

Procedimento:
1. Clone a versão fixada em uma pasta temporária do sistema:
   git clone --branch v1.0.0 --depth 1 https://github.com/JosehRoberto/menu-antares-opencode.git
2. Garanta Python 3 e o módulo pyyaml no ambiente atual.
3. Na raiz do meu projeto, execute o instalador não interativo:
   - Linux/macOS: bash <pasta-clonada>/bin/install.sh --yes
   - Windows: powershell -ExecutionPolicy Bypass -File <pasta-clonada>\bin\install.ps1 --yes
4. Não apague opencode.json nem opencode.jsonc; crie backup se existirem.
5. Ao final, informe o resumo (criados, atualizados, ignorados).
```

O agente executa passos determinísticos: obtém a **versão fixada**, roda o instalador **não interativo** e reporta o resumo.

### Opção B — execução manual

Clone a versão fixada e execute o wrapper do seu sistema **dentro da raiz do projeto onde o Antares foi instalado**:

```bash
git clone --branch v1.0.0 --depth 1 https://github.com/JosehRoberto/menu-antares-opencode.git /tmp/menu-antares-opencode
```

**Linux / macOS:**

```bash
bash /tmp/menu-antares-opencode/bin/install.sh --yes
```

**Windows 10/11 (PowerShell):**

```powershell
powershell -ExecutionPolicy Bypass -File C:\caminho\menu-antares-opencode\bin\install.ps1 --yes
```

Depois feche e reabra o OpenCode.

## Atualização

Quando o Antares ou o Mira for atualizado com novos skills, sincronize os comandos (cria novos, atualiza alterados com backup e remove órfãos com confirmação):

**Linux / macOS:**

```bash
bash /tmp/menu-antares-opencode/bin/sync.sh --yes
```

**Windows 10/11 (PowerShell):**

```powershell
powershell -ExecutionPolicy Bypass -File C:\caminho\menu-antares-opencode\bin\sync.ps1 --yes
```

Ou use o núcleo direto, em qualquer sistema:

```bash
python3 menu-antares.py install --yes
python3 menu-antares.py sync --yes
```

## Arquitetura

| Arquivo | Papel |
|---|---|
| `menu-antares.py` | Núcleo único, portátil. Contém toda a lógica. Subcomandos `install` e `sync` |
| `bin/install.sh`, `bin/sync.sh` | Wrappers finos para Linux/macOS (bash 3.2+) |
| `bin/install.ps1`, `bin/sync.ps1` | Wrappers finos para Windows (PowerShell) |

Os wrappers apenas detectam o Python 3 (`python3` → `python` → `py -3`) e chamam o núcleo. Toda a lógica vive em `menu-antares.py`, garantindo o mesmo comportamento em todos os sistemas.

## Flags do `menu-antares.py`

| Flag | Descrição |
|---|---|
| `--yes` / `-y` | Não interativo: responde "sim" a todas as confirmações |
| `--dry-run` | Mostra o que seria feito, sem alterar arquivos |
| `--backup-dir DIR` | Diretório de backups (padrão: `.opencode/backups`) |
| `--keep-gitignore` | Não altera o `.gitignore` |
| `--skills-root DIR` | Raiz dos skills (padrão: `.agents/skills`) |
| `--commands-dir DIR` | Diretório de comandos (padrão: `.opencode/commands`) |

## O que este repositório faz de diferente

- Cobre **`antares-*` e `mira-*`** no mesmo script (e os skills `antares` e `mira` sem sufixo, se existirem).
- Ignora `_shared/` (não possui `SKILL.md`).
- Marca `.opencode/` no `.gitignore` (de forma idempotente), tratando os comandos como artefato local regenerável — não versionado.
- Lê a `description` do frontmatter com **PyYAML**, não com `grep`. Isso é obrigatório porque vários skills do Mira usam `description: >-` (bloco multi-linha); um `grep '^description:'` capturaria apenas o `>-` e geraria um comando com descrição vazia.
- Serializa a descrição com `json.dumps`, produzindo um escalar YAML duplamente citado válido mesmo quando o texto contém aspas, dois-pontos, `+` ou `&`.
- Não remove `opencode.json` nem `opencode.jsonc` automaticamente.
- Cria backup em `.opencode/backups/` quando o usuário desejar preservar arquivos existentes.
- Pede confirmação antes de gerar, atualizar ou remover comandos; em modo `--yes` age sem intervenção.
- Evita sobrescrever arquivos sem necessidade (compara conteúdo antes de atualizar).

## Matriz de compatibilidade

| Plataforma | Wrapper | Requisitos | Observações |
|---|---|---|---|
| Ubuntu / Debian / Fedora / Mint | `bin/*.sh` | bash, python3, pyyaml | PEP 668 no pyyaml |
| macOS | `bin/*.sh` | python3 (CLT/Homebrew), pyyaml | bash 3.2; PEP 668 |
| Windows 10/11 | `bin/*.ps1` | Python 3, pyyaml | `-ExecutionPolicy Bypass`, encoding UTF-8 |

## Comportamento esperado

O OpenCode tem um **bug conhecido** (issue [#4555](https://github.com/anomalyco/opencode/issues/4555)): o autocomplete do menu `/` exibe apenas parte dos comandos inicialmente. Isso não significa que os comandos não foram carregados; os comandos continuam utilizáveis quando digitados por nome ou quando o filtro reduz a lista.

| Ação | Comportamento |
|------|---------------|
| Digitar `/` | Pode mostrar apenas parte dos comandos |
| Digitar `/antares-` | Filtra e mostra os comandos que começam com `antares-` |
| Digitar `/mira-` | Filtra e mostra os comandos que começam com `mira-` |
| Digitar `/antares-cap` | Filtra e mostra `antares-capa` |
| Digitar o comando completo e pressionar Enter | Funciona normalmente |

Além disso, existe relato de atraso para os comandos aparecerem após alterações, por isso é recomendável fechar e reabrir o OpenCode depois de modificar `.opencode/commands/`.

## Orientações de uso

### Quando usar `install`

Use `install` quando:

- a pasta acabou de receber a instalação do Antares (com o Mira junto);
- ainda não existem comandos `antares-*` / `mira-*` em `.opencode/commands/`;
- você quer preservar qualquer configuração local já existente do OpenCode.

### Quando usar `sync`

Use `sync` quando:

- você executou a atualização do Antares ou do Mira;
- novas skills foram adicionadas, removidas ou alteradas;
- você quer revisar antes de atualizar ou remover comandos já existentes.

### Sobre `opencode.json` e `opencode.jsonc`

Se já existir `opencode.json` ou `opencode.jsonc`, a recomendação é **preservar esses arquivos**.

- Não apagar automaticamente.
- Fazer backup antes de qualquer mudança manual.
- Usar `.opencode/commands/*.md` para os comandos.
- Revisar manualmente o conteúdo de `opencode.json` apenas se houver suspeita de conflito de configuração.

### Sobre versionamento de `.opencode/`

Os comandos em `.opencode/commands/*.md` são **artefato local regenerável** (a fonte de verdade é `.agents/skills/*/SKILL.md`), por isso o padrão deste repositório é **não versionar** `.opencode/`:

- Os scripts adicionam `.opencode/` ao `.gitignore` automaticamente, sem duplicar linha.
- Quem clonar o projeto gera os comandos rodando a instalação uma vez.
- O comando `sync` mantém os comandos em dia após updates do Antares/Mira.

Se preferir versionar os comandos, use `--keep-gitignore` (para não tocar no `.gitignore`) e versiona apenas o que quiser, ignorando ao menos `.opencode/backups/`.

## Notas técnicas

- O OpenCode reconhece comandos de projeto em `.opencode/commands/*.md` (também aceita `.opencode/command/`). Este repositório usa o plural, alinhado aos repositórios `menu-mira-opencode` e `menu-reversa-opencode`.
- O OpenCode **carrega** os skills de `.agents/skills/`, mas o **menu `/` lista comandos, não skills**. Por isso é necessário gerar um arquivo de comando por skill.
- `PyYAML` é usado para ler apenas o frontmatter inicial de cada `SKILL.md`, o que trata corretamente `description: >-` e evita erro quando o corpo do arquivo contém outro `---`.
- A descrição é serializada com `json.dumps`, garantindo um escalar YAML citado válido.
- `skills.paths` não é necessário neste fluxo.
- A entrada `.opencode/` no `.gitignore` é verificada antes de escrever (idempotente) e preserva o restante do arquivo.
- `agent: build` é usado como agente padrão; ajuste o frontmatter gerado se preferir outro agente.
- Comandos órfãos (skill removido) só são excluídos no `sync` e com confirmação explícita.

### Bugs e observações conhecidas

- [#4555](https://github.com/anomalyco/opencode/issues/4555): autocomplete truncado no menu `/`.
- [#8017](https://github.com/anomalyco/opencode/issues/8017): atraso para comandos aparecerem após alterações.
- [#18987](https://github.com/anomalyco/opencode/issues/18987): presença de `opencode.json` pode interferir em alguns cenários; por isso a recomendação atual é preservar, fazer backup e revisar manualmente em vez de apagar automaticamente.

## Solução de problemas

### `pip` recusa a instalação do pyyaml (PEP 668)

Use o pacote do sistema (`python3-yaml` no apt, `python3-pyyaml` no dnf), `--user` (`python3 -m pip install --user pyyaml`) ou um `venv`.

### macOS: `python3` não encontrado

Instale as Command Line Tools (`xcode-select --install`) ou o Python via Homebrew (`brew install python`).

### Windows: script bloqueado pela ExecutionPolicy

Execute com bypass explícito:

```powershell
powershell -ExecutionPolicy Bypass -File C:\caminho\menu-antares-opencode\bin\install.ps1 --yes
```

### Acentuação/emoji estranhos no console do Windows

O núcleo grava os arquivos em UTF-8 e reconfigura a saída para UTF-8. Se o console ainda exibir caracteres estranhos, rode `chcp 65001` antes.

### `ERRO: diretório '.agents/skills' não encontrado`

Você não está na raiz do projeto onde o Antares (e o Mira) foi instalado. Rode o wrapper a partir dessa pasta.

## Legado (script shell, v0)

A versão anterior baseada em script shell embutido no README continua disponível no histórico:

- README histórico: <https://github.com/JosehRoberto/menu-antares-opencode/blob/5e45265/README.md>
- Prompt legado: `Execute o procedimento descrito em https://github.com/JosehRoberto/menu-antares-opencode/blob/5e45265/README.md ...`

O fluxo legado exige Git Bash ou WSL no Windows e não é determinístico entre sistemas; prefira as opções A ou B acima.

## Licença

MIT
