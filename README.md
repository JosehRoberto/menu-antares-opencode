# Menu de Comandos Antares e Mira para OpenCode

Gera automaticamente os comandos `/antares-*` e `/mira-*` no menu do OpenCode a partir dos skills instalados pelo [Antares](https://github.com/sandeco/antares) e pelo [Mira Animator](https://github.com/sandeco/mira-animator).

> **Por que este repositório cobre os dois prefixos?**
> A instalação do Antares traz o Mira junto (a pasta `.agents/skills/` contém `antares-*`, `mira-*` e `_shared/`). Um único script gera os comandos dos dois frameworks, em vez de rodar `menu-mira-opencode` e este script separadamente na mesma pasta.

## Pré-requisitos

- **Python 3** com o módulo `pyyaml`.
  - Instalação: `pip install pyyaml` ou `pip3 install pyyaml`.
  - Verificação: `python3 -c "import yaml"`.
- **Bash** (Linux/macOS).
- No Windows, use Git Bash ou WSL.

## Instalação

Após instalar o Antares (e o Mira) em uma nova pasta, abra o OpenCode e cole este prompt na primeira mensagem:

```text
Execute o procedimento descrito em https://github.com/JosehRoberto/menu-antares-opencode para instalar os comandos Antares e Mira no menu do OpenCode com segurança, preservando qualquer configuração existente do OpenCode.
```

O agente acessará este README via `webfetch` e executará o procedimento descrito aqui.

> **Compatibilidade:** os scripts usam Python 3 e Bash (Linux/macOS). Quando executados via prompt no OpenCode, o agente de IA traduz os comandos para o shell do seu sistema automaticamente (PowerShell no Windows, zsh no macOS, bash no Linux). Para execução manual no Windows, use Git Bash ou WSL.

## O que este repositório faz de diferente

- Cobre **`antares-*` e `mira-*`** no mesmo script (e os skills `antares` e `mira` sem sufixo, se existirem).
- Ignora `_shared/` (não possui `SKILL.md`).
- Marca `.opencode/` no `.gitignore` (de forma idempotente), tratando os comandos como artefato local regenerável — não versionado.
- Lê a `description` do frontmatter com **PyYAML**, não com `grep`. Isso é obrigatório porque vários skills do Mira usam `description: >-` (bloco multi-linha); um `grep '^description:'` capturaria apenas o `>-` e geraria um comando com descrição vazia.
- Serializa a descrição com `json.dumps`, produzindo um escalar YAML duplamente citado válido mesmo quando o texto contém aspas, dois-pontos, `+` ou `&`.
- Não remove `opencode.json` nem `opencode.jsonc` automaticamente.
- Cria backup em `.opencode/backups/` quando o usuário desejar preservar arquivos existentes.
- Pede confirmação antes de gerar, atualizar ou remover comandos.
- Evita sobrescrever arquivos sem necessidade.

## Instalação segura

Use este script para uma instalação inicial mais segura. Ele cria os comandos Antares e Mira em `.opencode/commands/`, preserva a configuração existente do usuário e oferece backup quando encontrar `opencode.json` ou `opencode.jsonc`.

```bash
#!/usr/bin/env bash
set -euo pipefail

COMMANDS_DIR=".opencode/commands"
SKILLS_ROOT=".agents/skills"
BACKUP_DIR=".opencode/backups"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
CREATED_COUNT=0
SKIPPED_COUNT=0
UPDATED_COUNT=0

echo "==> Instalação segura dos comandos Antares e Mira no OpenCode"
echo

if [ ! -d "$SKILLS_ROOT" ]; then
  echo "ERRO: diretório '$SKILLS_ROOT' não encontrado."
  echo "Abra este script na raiz do projeto onde o Antares (e o Mira) foi instalado."
  exit 1
fi

mkdir -p "$COMMANDS_DIR"
mkdir -p "$BACKUP_DIR"

backup_file() {
  local file="$1"
  if [ -f "$file" ]; then
    local base
    base="$(basename "$file")"
    cp -p "$file" "$BACKUP_DIR/${base}.${TIMESTAMP}.bak"
    echo "Backup criado: $BACKUP_DIR/${base}.${TIMESTAMP}.bak"
  fi
}

confirm() {
  local prompt="$1"
  local response
  read -r -p "$prompt [y/N]: " response
  case "$response" in
    y|Y|yes|YES|s|S|sim|SIM) return 0 ;;
    *) return 1 ;;
  esac
}

# Garante uma entrada no .gitignore sem duplicar nem quebrar o arquivo.
ensure_gitignore() {
  local entry="$1"
  [ -f ".gitignore" ] || : > ".gitignore"
  if ! grep -qxF "$entry" ".gitignore"; then
    if [ -s ".gitignore" ]; then
      printf '\n' >> ".gitignore"
    fi
    printf '%s\n' "$entry" >> ".gitignore"
    echo "Adicionado ao .gitignore: $entry"
  fi
}

# Extrai apenas o frontmatter inicial do SKILL.md e escreve o comando.
# Recebe: nome do skill, caminho do SKILL.md, caminho do comando de saída.
build_command() {
  python3 - "$1" "$2" "$3" <<'PYEOF'
import json
import sys
from pathlib import Path
import yaml

skill_name = sys.argv[1]
skill_file = Path(sys.argv[2])
cmd_file = Path(sys.argv[3])

text = skill_file.read_text(encoding="utf-8")
desc = ""
if text.startswith("---"):
    end = text.find("\n---", 3)
    if end != -1:
        frontmatter = text[3:end]
        try:
            data = yaml.safe_load(frontmatter) or {}
            if isinstance(data, dict):
                desc = str(data.get("description", "")).strip()
        except Exception:
            desc = ""

if not desc:
    desc = f"Comando Antares/Mira: {skill_name}"

# json.dumps produz um escalar YAML duplamente citado válido.
desc_literal = json.dumps(desc, ensure_ascii=False)

content = (
    "---\n"
    f"description: {desc_literal}\n"
    "agent: build\n"
    "---\n"
    "\n"
    f"# Comando `{skill_name}` do Antares/Mira\n"
    "\n"
    f"Ative este comando para carregar o skill `{skill_name}` e siga as instruções do SKILL.md.\n"
    "\n"
    f"/{skill_name}\n"
)
cmd_file.write_text(content, encoding="utf-8")
PYEOF
}

echo "Diretório de skills: $SKILLS_ROOT"
echo "Diretório de comandos: $COMMANDS_DIR"
echo

if [ -f "opencode.json" ] || [ -f "opencode.jsonc" ]; then
  echo "ATENÇÃO: foi encontrado arquivo de configuração do OpenCode neste projeto."
  [ -f "opencode.json" ] && echo " - opencode.json"
  [ -f "opencode.jsonc" ] && echo " - opencode.jsonc"
  echo
  echo "Esse arquivo NÃO será apagado."
  echo "O OpenCode suporta comandos por arquivos .md e também por configuração JSON/JSONC."
  echo "Apagar automaticamente pode remover configurações válidas do usuário."
  echo
  if confirm "Deseja criar backup desses arquivos agora?"; then
    [ -f "opencode.json" ] && backup_file "opencode.json"
    [ -f "opencode.jsonc" ] && backup_file "opencode.jsonc"
  fi
  echo
fi

if ! confirm "Deseja gerar os comandos Antares e Mira em '$COMMANDS_DIR'?"; then
  echo "Operação cancelada pelo usuário."
  exit 0
fi

# Estratégia adotada: .opencode/ é artefato local regenerável, então fica fora do git.
ensure_gitignore ".opencode/"

for root in "$SKILLS_ROOT"; do
  [ -d "$root" ] || continue

  for skill_dir in "$root"/antares-*/ "$root"/mira-*/ "$root"/antares/ "$root"/mira/; do
    [ -d "$skill_dir" ] || continue

    skill_name="$(basename "$skill_dir")"
    skill_file="$skill_dir/SKILL.md"
    cmd_file="$COMMANDS_DIR/$skill_name.md"

    if [ ! -f "$skill_file" ]; then
      echo "Ignorado: $skill_name (sem SKILL.md)"
      SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
      continue
    fi

    tmp_file="$(mktemp)"
    build_command "$skill_name" "$skill_file" "$tmp_file"

    if [ -f "$cmd_file" ]; then
      if cmp -s "$tmp_file" "$cmd_file"; then
        echo "Sem alteração: $skill_name"
        SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
        rm -f "$tmp_file"
        continue
      fi

      echo
      echo "Comando existente detectado: $cmd_file"
      if confirm "Deseja atualizar este comando mantendo backup?"; then
        backup_file "$cmd_file"
        mv "$tmp_file" "$cmd_file"
        echo "Atualizado: $skill_name"
        UPDATED_COUNT=$((UPDATED_COUNT + 1))
      else
        echo "Mantido sem alteração: $skill_name"
        SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
        rm -f "$tmp_file"
      fi
    else
      mv "$tmp_file" "$cmd_file"
      echo "Criado: $skill_name"
      CREATED_COUNT=$((CREATED_COUNT + 1))
    fi
  done
done

echo
echo "Resumo:"
echo " - Criados: $CREATED_COUNT"
echo " - Atualizados: $UPDATED_COUNT"
echo " - Ignorados: $SKIPPED_COUNT"
echo
echo "Concluído com segurança."
echo "Feche e reabra o OpenCode para recarregar os comandos."
```

Depois feche e reabra o OpenCode.

## Atualização segura

Quando o Antares ou o Mira for atualizado com novos skills, abra o OpenCode e cole este prompt:

```text
Sincronize os comandos Antares e Mira no menu do OpenCode conforme o procedimento seguro em https://github.com/JosehRoberto/menu-antares-opencode, preservando comandos existentes, criando backups e pedindo confirmação antes de remover comandos órfãos.
```

O script abaixo cria comandos novos, atualiza comandos alterados com backup, preserva `opencode.json` e pergunta antes de remover comandos órfãos.

```bash
#!/usr/bin/env bash
set -euo pipefail

COMMANDS_DIR=".opencode/commands"
SKILLS_ROOT=".agents/skills"
BACKUP_DIR=".opencode/backups"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
CREATED_COUNT=0
UPDATED_COUNT=0
REMOVED_COUNT=0
SKIPPED_COUNT=0

if [ ! -d "$SKILLS_ROOT" ]; then
  echo "ERRO: diretório '$SKILLS_ROOT' não encontrado."
  echo "Abra este script na raiz do projeto onde o Antares (e o Mira) foi instalado."
  exit 1
fi

mkdir -p "$COMMANDS_DIR"
mkdir -p "$BACKUP_DIR"

backup_file() {
  local file="$1"
  if [ -f "$file" ]; then
    local base
    base="$(basename "$file")"
    cp -p "$file" "$BACKUP_DIR/${base}.${TIMESTAMP}.bak"
    echo "Backup criado: $BACKUP_DIR/${base}.${TIMESTAMP}.bak"
  fi
}

confirm() {
  local prompt="$1"
  local response
  read -r -p "$prompt [y/N]: " response
  case "$response" in
    y|Y|yes|YES|s|S|sim|SIM) return 0 ;;
    *) return 1 ;;
  esac
}

# Garante uma entrada no .gitignore sem duplicar nem quebrar o arquivo.
ensure_gitignore() {
  local entry="$1"
  [ -f ".gitignore" ] || : > ".gitignore"
  if ! grep -qxF "$entry" ".gitignore"; then
    if [ -s ".gitignore" ]; then
      printf '\n' >> ".gitignore"
    fi
    printf '%s\n' "$entry" >> ".gitignore"
    echo "Adicionado ao .gitignore: $entry"
  fi
}

build_command() {
  python3 - "$1" "$2" "$3" <<'PYEOF'
import json
import sys
from pathlib import Path
import yaml

skill_name = sys.argv[1]
skill_file = Path(sys.argv[2])
cmd_file = Path(sys.argv[3])

text = skill_file.read_text(encoding="utf-8")
desc = ""
if text.startswith("---"):
    end = text.find("\n---", 3)
    if end != -1:
        frontmatter = text[3:end]
        try:
            data = yaml.safe_load(frontmatter) or {}
            if isinstance(data, dict):
                desc = str(data.get("description", "")).strip()
        except Exception:
            desc = ""

if not desc:
    desc = f"Comando Antares/Mira: {skill_name}"

desc_literal = json.dumps(desc, ensure_ascii=False)

content = (
    "---\n"
    f"description: {desc_literal}\n"
    "agent: build\n"
    "---\n"
    "\n"
    f"# Comando `{skill_name}` do Antares/Mira\n"
    "\n"
    f"Ative este comando para carregar o skill `{skill_name}` e siga as instruções do SKILL.md.\n"
    "\n"
    f"/{skill_name}\n"
)
cmd_file.write_text(content, encoding="utf-8")
PYEOF
}

echo "==> Sincronização segura dos comandos Antares e Mira"
echo

if [ -f "opencode.json" ] || [ -f "opencode.jsonc" ]; then
  echo "Configuração existente do OpenCode detectada neste projeto."
  [ -f "opencode.json" ] && echo " - opencode.json"
  [ -f "opencode.jsonc" ] && echo " - opencode.jsonc"
  echo "Esses arquivos serão preservados."
  echo
fi

# Estratégia adotada: .opencode/ é artefato local regenerável, então fica fora do git.
ensure_gitignore ".opencode/"

for root in "$SKILLS_ROOT"; do
  [ -d "$root" ] || continue

  for skill_dir in "$root"/antares-*/ "$root"/mira-*/ "$root"/antares/ "$root"/mira/; do
    [ -d "$skill_dir" ] || continue

    skill_name="$(basename "$skill_dir")"
    skill_file="$skill_dir/SKILL.md"
    cmd_file="$COMMANDS_DIR/$skill_name.md"

    if [ ! -f "$skill_file" ]; then
      echo "Ignorado: $skill_name (sem SKILL.md)"
      SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
      continue
    fi

    tmp_file="$(mktemp)"
    build_command "$skill_name" "$skill_file" "$tmp_file"

    if [ -f "$cmd_file" ]; then
      if cmp -s "$tmp_file" "$cmd_file"; then
        echo "Sem alteração: $skill_name"
        SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
        rm -f "$tmp_file"
      else
        echo
        echo "Comando alterado detectado: $cmd_file"
        if confirm "Deseja atualizar este comando mantendo backup?"; then
          backup_file "$cmd_file"
          mv "$tmp_file" "$cmd_file"
          echo "Atualizado: $skill_name"
          UPDATED_COUNT=$((UPDATED_COUNT + 1))
        else
          echo "Mantido sem alteração: $skill_name"
          SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
          rm -f "$tmp_file"
        fi
      fi
    else
      mv "$tmp_file" "$cmd_file"
      echo "Criado: $skill_name"
      CREATED_COUNT=$((CREATED_COUNT + 1))
    fi
  done
done

for cmd_file in "$COMMANDS_DIR"/antares*.md "$COMMANDS_DIR"/antares.md \
                "$COMMANDS_DIR"/mira*.md "$COMMANDS_DIR"/mira.md; do
  [ -f "$cmd_file" ] || continue
  skill_name="$(basename "$cmd_file" .md)"

  if [ ! -d "$SKILLS_ROOT/$skill_name" ]; then
    echo
    echo "Comando órfão detectado: $skill_name"
    if confirm "Deseja remover este comando órfão mantendo backup?"; then
      backup_file "$cmd_file"
      rm -f "$cmd_file"
      echo "Removido: $skill_name"
      REMOVED_COUNT=$((REMOVED_COUNT + 1))
    else
      echo "Preservado: $skill_name"
      SKIPPED_COUNT=$((SKIPPED_COUNT + 1))
    fi
  fi
done

echo
echo "Resumo:"
echo " - Criados: $CREATED_COUNT"
echo " - Atualizados: $UPDATED_COUNT"
echo " - Removidos: $REMOVED_COUNT"
echo " - Ignorados: $SKIPPED_COUNT"
echo
echo "Sincronização concluída com segurança."
echo "Feche e reabra o OpenCode após alterações em .opencode/commands/."
```

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

### Quando usar o script de instalação

Use o script de **Instalação segura** quando:

- a pasta acabou de receber a instalação do Antares (com o Mira junto);
- ainda não existem comandos `antares-*` / `mira-*` em `.opencode/commands/`;
- você quer preservar qualquer configuração local já existente do OpenCode.

### Quando usar o script de sincronização

Use o script de **Atualização segura** quando:

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
- O script de sincronização mantém os comandos em dia após updates do Antares/Mira.

Se preferir versionar os comandos, remova a chamada `ensure_gitignore ".opencode/"` e ignore ao menos `.opencode/backups/`.

## Notas técnicas

- O OpenCode reconhece comandos de projeto em `.opencode/commands/*.md` (também aceita `.opencode/command/`). Este repositório usa o plural, alinhado aos repositórios `menu-mira-opencode` e `menu-reversa-opencode`.
- O OpenCode **carrega** os skills de `.agents/skills/`, mas o **menu `/` lista comandos, não skills**. Por isso é necessário gerar um arquivo de comando por skill.
- `PyYAML` é usado para ler apenas o frontmatter inicial de cada `SKILL.md`, o que trata corretamente `description: >-` e evita erro quando o corpo do arquivo contém outro `---`.
- A descrição é serializada com `json.dumps`, garantindo um escalar YAML citado válido.
- `skills.paths` não é necessário neste fluxo.
- Os scripts mantêm `.opencode/` no `.gitignore` via `ensure_gitignore`, que verifica a entrada antes de escrever (idempotente) e preserva o restante do arquivo.
- `agent: build` é usado como agente padrão; ajuste o frontmatter gerado se preferir outro agente.
- Comandos órfãos (skill removido) só são excluídos na sincronização e com confirmação explícita.

### Bugs e observações conhecidas

- [#4555](https://github.com/anomalyco/opencode/issues/4555): autocomplete truncado no menu `/`.
- [#8017](https://github.com/anomalyco/opencode/issues/8017): atraso para comandos aparecerem após alterações.
- [#18987](https://github.com/anomalyco/opencode/issues/18987): presença de `opencode.json` pode interferir em alguns cenários; por isso a recomendação atual é preservar, fazer backup e revisar manualmente em vez de apagar automaticamente.

## Licença

MIT
