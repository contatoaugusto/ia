# Skill: Gerar Scripts de Atualização de Banco por Branch

## Parâmetros

Este skill aceita até dois parâmetros posicionais opcionais:

```
/skill_bancoGeraScriptAtualizacao [branch_origem] [branch_destino]
```

| Parâmetro | Descrição | Default |
|---|---|---|
| `branch_origem` | Branch que contém as alterações a serem deployadas | `dev` |
| `branch_destino` | Branch de referência (base de comparação) | `main` |

**Exemplos de uso:**
```
/skill_bancoGeraScriptAtualizacao
/skill_bancoGeraScriptAtualizacao feature/minha-branch
/skill_bancoGeraScriptAtualizacao feature/minha-branch dev
```

Quando invocado, execute **exatamente** os passos abaixo sem pedir confirmação intermediária.

---

## 1. Resolver os parâmetros

Extraia `$BRANCH_ORIGEM` e `$BRANCH_DESTINO` do que foi passado pelo usuário.  
Se não informados, use os defaults:

- `$BRANCH_ORIGEM` = `dev`
- `$BRANCH_DESTINO` = `main`

Informe ao usuário as branches que serão comparadas antes de prosseguir:

```
Comparando: origin/<$BRANCH_ORIGEM>  →  origin/<$BRANCH_DESTINO>
```

---

## 2. Identificar arquivos alterados

Execute:

```bash
git log origin/<$BRANCH_DESTINO>..origin/<$BRANCH_ORIGEM> --name-only --format="AUTHOR:%an" | awk '
/^AUTHOR:/ { author=$0; sub(/^AUTHOR:/, "", author) }
/\.sql$|\.cs$/ { print author ": " $0 }
' | sort
```

Se o resultado estiver vazio, informe: _"Nenhum arquivo encontrado entre as branches informadas."_ e encerre.

Liste os resultados agrupados por autor.

---

## 3. Agrupar arquivos por autor

Para cada autor distinto, monte a lista dos seus arquivos.  
Ignore arquivos não deployáveis como `.sqlproj`, `Model.bim` e arquivos `.cs` sem SQL correspondente.

---

## 4. Gerar o script de atualização para cada autor

Para cada autor, crie o arquivo:

```
D:\Arquivos\Database\Atualizacao_<yyyy-mm-dd>_<nome do autor>.sql
```

Onde `<yyyy-mm-dd>` é a data de hoje.

Salve o PowerShell abaixo em `/tmp/gerar_scripts_<autor>.ps1` e execute com `powershell -File`:

```powershell
$baseDir = 'D:\tfs\Databases_git'
$outDir  = 'D:\Arquivos\Database'
$hoje    = (Get-Date).ToString('yyyy-MM-dd')

function Build-AlterScript {
    param([string]$file)
    $content = Get-Content $file -Raw -Encoding UTF8
    # CREATE PROC / CREATE PROCEDURE -> CREATE OR ALTER PROCEDURE
    $content = $content -replace '(?i)\bCREATE\s+PROC(EDURE)?\b', 'CREATE OR ALTER PROCEDURE'
    # CREATE FUNCTION / TRIGGER / VIEW -> CREATE OR ALTER <tipo>
    $content = $content -replace '(?i)\bCREATE\s+(FUNCTION|TRIGGER|VIEW)\b', 'CREATE OR ALTER $1'
    return "$content`r`nGO`r`n"
}

function Build-NewTableScript {
    param([string]$file, [string]$schema, [string]$name)
    $content = Get-Content $file -Raw -Encoding UTF8
    $block  = "IF OBJECT_ID('[$schema].[$name]', 'U') IS NULL`r`nBEGIN`r`n"
    $block += $content
    $block += "`r`nEND`r`nGO`r`n"
    return $block
}
```

### Regras de transformação por tipo de objeto

| Tipo de objeto | Transformação |
|---|---|
| `CREATE PROCEDURE` ou `CREATE PROC` | `CREATE OR ALTER PROCEDURE` |
| `CREATE FUNCTION` | `CREATE OR ALTER FUNCTION` |
| `CREATE TRIGGER` | `CREATE OR ALTER TRIGGER` |
| `CREATE VIEW` | `CREATE OR ALTER VIEW` |
| `CREATE TABLE` (tabela nova) | `IF OBJECT_ID(..., 'U') IS NULL BEGIN CREATE TABLE ... END` |
| Funções CLR (`EXTERNAL NAME`) | `DROP FUNCTION IF EXISTS` + `CREATE FUNCTION` (CLR não suporta ALTER) |

### Estrutura do arquivo gerado

O arquivo deve conter **apenas o SQL necessário**, sem comentários de cabeçalho, sem linha de identificação do objeto e sem qualquer texto adicional. Exemplo:

```sql
USE [SGICEUB]
GO

CREATE OR ALTER PROCEDURE [schema].[nome]
...
GO
```

Quando o autor tiver objetos em mais de um banco, inserir `USE [banco]` + `GO` antes de cada grupo, sem nenhum comentário ao redor.

### Mapeamento de projeto para banco de dados

| Pasta raiz do arquivo | USE |
|---|---|
| `SGICEUB.Database/` | `USE [SGICEUB]` |
| `SGIBROKER.Database/` | `USE [SGIBROKER]` |
| `DWC/DWC.Database/` | `USE [DWC]` |

---

## 5. Validação após geração

Execute para confirmar que todos os objetos foram transformados corretamente:

```bash
grep -h "CREATE OR ALTER\|IF OBJECT_ID\|DROP FUNCTION IF EXISTS\|USE \[" \
  "D:/Arquivos/Database/Atualizacao_<data>_*.sql" \
  | grep -v "^--" | sort -u
```

Exiba o resultado para o usuário.

---

## 6. Relatório final

Exiba uma tabela com:

| Arquivo gerado | Tamanho | Objetos incluídos |
|---|---|---|
| Atualizacao_..._Autor1.sql | X KB | lista resumida dos objetos |
| Atualizacao_..._Autor2.sql | X KB | lista resumida dos objetos |

---

## Notas importantes

- **Nunca usar DROP + CREATE** para objetos existentes — use sempre `CREATE OR ALTER`.
- **Funções CLR** são exceção: não suportam `ALTER`, usar `DROP FUNCTION IF EXISTS` + `CREATE FUNCTION`.
- **Tabelas novas**: proteger com `IF OBJECT_ID(..., 'U') IS NULL` antes do `CREATE TABLE`.
- Arquivos `.cs` de assemblies CLR **não podem ser deployados como SQL** — incluir comentário no script avisando que o assembly precisa ser atualizado separadamente antes de executar as funções CLR.
- Respeitar a **ordem de dependências** ao criar tabelas (tabelas pai antes das filhas com FK).
- O script gerado deve ser executável diretamente no **SQL Server Management Studio** ou via `sqlcmd`.
- **Não adicionar nenhum comentário** no arquivo gerado — nem cabeçalho, nem identificação de objeto, nem separadores. Apenas o SQL puro.
