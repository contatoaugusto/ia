<!--
================================================================================
  SKILL: dotnet-git-azuredevops-publicação
  Versão: 1.0
  Repositório: https://github.com/contatoaugusto/ia/blob/main/Claude/Skills/dotnet-git-azuredevops-publicação.md
================================================================================

MINI MANUAL DE USO
==================

1. PRÉ-REQUISITOS
-----------------
  - Claude Code instalado (CLI): https://claude.ai/code
  - Azure CLI com extensão azure-devops:
      az extension add --name azure-devops
  - Autenticado no Azure DevOps:
      az devops login   (ou)   az login
  - Git instalado e acessível no terminal
  - O terminal deve estar em um repositório Git cujo remote aponta para Azure DevOps

2. ONDE COLOCAR O ARQUIVO DESTA SKILL
--------------------------------------
  Windows:   C:\Users\<SeuUsuario>\.claude\commands\dotnet-git-azuredevops-publicação.md
  Linux/Mac: ~/.claude/commands/dotnet-git-azuredevops-publicação.md

  Se a pasta "commands" não existir, crie-a manualmente.

3. COMO CHAMAR A SKILL
-----------------------
  Dentro do Claude Code, digite:

    /dotnet-git-azuredevops-publicação <pr1> [<pr2> ...] <pasta-destino>

  Onde:
    <pr1>, <pr2>, ...  = Nomes (títulos) ou IDs numéricos dos Pull Requests,
                         separados por espaço. Use aspas para nomes com espaço.
    <pasta-destino>    = Caminho da pasta onde o HTML será salvo.
                         Deve ser o último argumento, separado por espaço.

  EXEMPLOS:

  a) Um PR por ID:
     /dotnet-git-azuredevops-publicação 142 D:\Relatorios

  b) Um PR por nome:
     /dotnet-git-azuredevops-publicação "Feature Boleto Fatura" D:\Relatorios

  c) Vários PRs por nome e ID:
     /dotnet-git-azuredevops-publicação "Feature XPTO" 137 "Fix Bug Login" D:\Relatorios

4. O QUE É GERADO
------------------
  Um arquivo HTML autocontido salvo em:
    <pasta-destino>\relatorio-prs-YYYYMMDD-HHmm.html

  O HTML contém:
    - Cards de resumo: total de PRs, autores, artefatos
    - Blocos por Autor > Projeto > PR
    - Tabela de artefatos por PR com tipo (Alteração/Inclusão/Exclusão)
    - Comentário curto gerado a partir dos commits e título do PR
    - Seções colapsáveis com botões Expandir/Recolher todos
    - Funciona offline — basta abrir no navegador

  Após o relatório, a skill conduz o merge de Dev → Main com
  confirmação obrigatória a cada passo.

5. COMO USAR SEM INSTALAR O ARQUIVO (qualquer máquina)
--------------------------------------------------------
  Em qualquer conversa com Claude Code, cole o seguinte prompt:

  "Busque a skill de publicação Azure DevOps em
   https://raw.githubusercontent.com/contatoaugusto/ia/main/Claude/Skills/dotnet-git-azuredevops-publicação.md
   e execute-a com os PRs <nomes/ids> e destino <pasta>"

6. ATUALIZAR PARA A VERSÃO MAIS RECENTE
-----------------------------------------
  https://github.com/contatoaugusto/ia/blob/main/Claude/Skills/dotnet-git-azuredevops-publicação.md

================================================================================
  FIM DO MANUAL — O conteúdo abaixo é a instrução executada pelo Claude
================================================================================
-->

Siga exatamente estas etapas para os argumentos fornecidos: $ARGUMENTS

## 1. Extrair argumentos

- O **último token** (separado por espaço, fora de aspas) é a `<pasta-destino>`
- Todos os tokens anteriores são os **nomes ou IDs dos Pull Requests**
  - Token puramente numérico → ID de PR
  - Token textual (com ou sem aspas) → título (ou parte do título) do PR
- Se a pasta de destino não existir, crie-a
- Nome do arquivo: `relatorio-prs-<YYYYMMDD-HHmm>.html`

---

## 2. Detectar configuração do Azure DevOps

Execute no diretório de trabalho atual:

```bash
git remote get-url origin
```

Identifique o formato da URL:
- `https://dev.azure.com/<organization>/<project>/_git/<repository>`
- `https://<organization>.visualstudio.com/<project>/_git/<repository>`

Extraia: `ORG`, `PROJECT`, `REPO`.

Configure o padrão da CLI:

```bash
az devops configure --defaults organization="https://dev.azure.com/<ORG>" project="<PROJECT>"
```

---

## 3. Para cada Pull Request — coletar dados

Repita os passos 3a–3e para cada PR informado.

### 3a. Localizar o PR

```bash
# Por ID numérico
az repos pr show --id <id> --output json

# Por título (busca parcial, todos os status)
az repos pr list --repository "<REPO>" --status all --output json \
  --query "[?contains(title, '<nome>')]"
```

Se vários PRs retornados pelo mesmo nome, usar o de maior `pullRequestId`.

### 3b. Extrair campos do PR

Do JSON retornado, extrair:
- `pullRequestId`
- `title`
- `createdBy.displayName` — **autor**
- `description`
- `sourceRefName` — branch origem (remover prefixo `refs/heads/`)
- `targetRefName` — branch destino
- `status` (`active`, `completed`, `abandoned`)
- `closedDate` ou `creationDate`

### 3c. Listar commits do PR

```bash
az repos pr list-commits --id <pr_id> --output json
```

Extrair por commit: `commitId`, `comment`, `author.name`, `author.date`.

### 3d. Listar arquivos alterados por commit

Para cada `commitId`:

```bash
# Opção 1 — CLI Azure DevOps
az repos git commit show --id <commitId> --repository "<REPO>" --output json

# Opção 2 — REST API (usar se CLI não retornar changes)
az rest --method get \
  --url "https://dev.azure.com/<ORG>/<PROJECT>/_apis/git/repositories/<REPO>/commits/<commitId>/changes?api-version=7.0"
```

De `changes[]`, extrair:
- `item.path` — caminho relativo do arquivo
- `changeType` — `edit` → Alteração | `add` → Inclusão | `delete` → Exclusão

**Deduplicar** arquivos que apareçam em múltiplos commits do mesmo PR (manter o `changeType` mais recente).

### 3e. Derivar projeto

O **projeto** de cada arquivo é o **primeiro segmento** do caminho (ex.: `Br.UniCeub.Software.BusinessObject` ou `UI.Web.SGI`).

### 3f. Gerar comentário curto

Com base em title, description e mensagens de commit, gere **uma linha** (≤ 120 chars) descrevendo o que foi implementado. Ex.:
- `"Adiciona campo icSemCobranca na entidade Contrato e ajusta SP de gravação"`
- `"Corrige validação de campos obrigatórios no formulário de matrícula"`

---

## 4. Gerar o Relatório HTML

Gere um arquivo HTML **completo e autocontido** (CSS e JS inline, sem CDN).

### Layout geral

- Font: system-ui / sans-serif
- Background: `#f0f2f5`
- Conteúdo: max-width 1200px, centralizado

### Cabeçalho

- Título: **"Relatório de Pull Requests — Azure DevOps"**
- Subtítulo: organização + projeto
- Data/hora de geração
- Cards de resumo global:
  - Total de PRs processados (azul `#1d4ed8`)
  - Total de Autores distintos (roxo `#7c3aed`)
  - Total de Artefatos únicos (cinza `#374151`)

### Agrupamento por Autor

Para cada **autor** distinto, um bloco com:
- Cabeçalho do bloco: nome do autor em destaque + total de PRs e artefatos
- Botões "Expandir todos" / "Recolher todos" no topo do bloco

Dentro do bloco, agrupado por **Projeto**:
- Título do projeto

Para cada PR do autor naquele projeto:
- Cabeçalho collapsible com: **título do PR**, número `#id`, badge de status, branch origem → destino
- Linha de comentário curto (em itálico)
- Tabela de artefatos com colunas: `#`, Arquivo, Caminho, Tipo

### Cores das seções de tipo

| Tipo       | Header        | Badge BG   | Borda linha |
|------------|---------------|------------|-------------|
| Alteração  | `#b45309`     | `#fef3c7`  | `#f59e0b`   |
| Inclusão   | `#15803d`     | `#dcfce7`  | `#22c55e`   |
| Exclusão   | `#b91c1c`     | `#fee2e2`  | `#ef4444`   |

### Seções colapsáveis

- Abertas por padrão
- Animação suave `max-height` via CSS transition
- Ícone ▸/▾ rotaciona ao expandir/colapsar
- IDs únicos por PR: `sec-pr-<id>-<projeto>`

### Rodapé

- "Gerado automaticamente por Claude Code"
- Data/hora e lista dos PRs incluídos (id + título)

---

## 5. Salvar e reportar

Salve o HTML em `<pasta-destino>\relatorio-prs-<YYYYMMDD-HHmm>.html`.

Informe ao usuário:
- Caminho completo do arquivo gerado
- Por PR: `#id — título — autor — N artefatos — comentário curto`
- Total consolidado de artefatos

---

## 6. Preparar e executar o Merge Dev → Main

> Cada passo exige **confirmação explícita** antes de ser executado.
> Se o usuário responder "N" ou "não", interrompa e informe o motivo da parada.

### Passo 6.1 — Verificar estado atual

Execute e exiba o resultado ao usuário:

```bash
git status --short
git branch -a | grep -E "^\*|dev|main"
git log --oneline -5
```

**Pergunta ao usuário:**
> "Deseja prosseguir com o merge da branch 'dev' para 'main'? (S/N)"

---

### Passo 6.2 — Atualizar branch Dev

Se confirmado:

```bash
git fetch origin
git checkout dev
git pull origin dev
```

Exiba resultado. Se houver erro, pare e informe.

**Pergunta ao usuário:**
> "Branch 'dev' atualizada com sucesso. Deseja prosseguir para atualizar 'main'? (S/N)"

---

### Passo 6.3 — Atualizar branch Main

Se confirmado:

```bash
git checkout main
git pull origin main
```

Exiba resultado.

**Pergunta ao usuário:**
> "Branch 'main' atualizada. Deseja executar o merge de 'dev' → 'main'? (S/N)"

---

### Passo 6.4 — Executar o Merge

Se confirmado, monte a mensagem de merge listando os PRs:

```bash
git merge dev --no-ff -m "Merge branch 'dev' into 'main' — PRs: #<id1> <título1>, #<id2> <título2>, ..."
```

**Se houver conflitos:**
- Liste os arquivos em conflito: `git diff --name-only --diff-filter=U`
- **Pare** e instrua o usuário a resolver manualmente
- Informe: para retomar após resolver, execute `git merge --continue` e depois refaça o push

**Se sem conflitos:**

Exiba resultado do merge.

**Pergunta ao usuário:**
> "Merge concluído localmente. Deseja fazer push para origin/main? (S/N)"

---

### Passo 6.5 — Push para Main

Se confirmado:

```bash
git push origin main
```

Exiba resultado.

---

## 7. Resumo Final

Ao concluir, exiba:

```
========================================
  RESUMO DA PUBLICAÇÃO
========================================
RELATÓRIO HTML
  Arquivo : <caminho-completo>
  PRs     : <quantidade>
  Autores : <quantidade>
  Artefatos: <quantidade>

PULL REQUESTS PROCESSADOS
  [✅|❌] #<id> — <título> — <autor> — <N> artefatos

MERGE Dev → Main
  [✅|❌] Passo 6.1 — Verificação do estado
  [✅|❌] Passo 6.2 — Atualização da branch Dev
  [✅|❌] Passo 6.3 — Atualização da branch Main
  [✅|❌] Passo 6.4 — Merge Dev → Main
  [✅|❌] Passo 6.5 — Push origin/main
========================================
```
