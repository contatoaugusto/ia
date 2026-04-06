Gere um relatório HTML de alterações Git com base nos argumentos fornecidos: $ARGUMENTS

O formato dos argumentos é: <caminho-do-projeto> <pasta-de-destino>
Exemplo: D:\TFS\SGI_GIT D:\Relatorios

Siga exatamente estes passos:

## 1. Extrair argumentos
- Primeiro argumento: caminho do projeto (onde está o repositório Git)
- Segundo argumento: pasta de destino onde o HTML será salvo
- Se a pasta de destino não existir, crie-a
- O nome do arquivo gerado deve ser: `relatorio-alteracoes-<YYYYMMDD-HHmm>.html`

## 2. Coletar dados do Git
Execute os seguintes comandos Bash:

```
git -C "<caminho-projeto>" status --porcelain
git -C "<caminho-projeto>" log -1 --pretty=format:"%H|%an|%ae|%ad|%s" --date=format:"%d/%m/%Y %H:%M"
git -C "<caminho-projeto>" rev-parse --abbrev-ref HEAD
git -C "<caminho-projeto>" remote get-url origin 2>/dev/null || echo "sem-remoto"
```

## 3. Classificar arquivos

Com base na primeira(s) letra(s) de cada linha do `git status --porcelain`:

| Código | Categoria     |
|--------|---------------|
| M, MM, AM | Alterações |
| R, RM  | Alterações (renomeado) |
| A      | Inclusões    |
| ??     | Inclusões (não rastreado) |
| D, MD  | Exclusões    |
| C      | Alterações (copiado) |

Para cada arquivo extraia:
- Nome do arquivo (sem o caminho)
- Caminho relativo completo
- Extensão
- Status exato (código git)

## 4. Gerar o HTML

Gere um arquivo HTML completo e autocontido com as seguintes características:

### Estrutura e layout
- Sem dependências externas (CSS e JS inline, sem CDN)
- Design leve, moderno e responsivo
- Fonte: system-ui / sans-serif
- Fundo da página: #f0f2f5
- Largura máxima do conteúdo: 1100px, centralizado

### Cabeçalho da página
- Título: "Relatório de Alterações Git"
- Nome do projeto: último segmento do caminho informado
- Branch atual
- Último commit: hash curto (7 chars), autor, data, mensagem
- Remoto (se disponível)
- Data/hora de geração do relatório
- Cards de resumo: total de Alterações, Inclusões, Exclusões e Total Geral — cada card com a cor da respectiva seção

### Seções collapsible (3 seções independentes)

Cada seção deve:
- Ter um cabeçalho clicável que expande/colapsa o corpo
- Iniciar **expandida** por padrão
- Exibir no cabeçalho: ícone, nome da seção e contador de itens (badge)
- Ter animação suave de expand/collapse (transition CSS)
- Ter cor distinta:
  - **Alterações**: tom âmbar/laranja — header `#b45309`, badge `#fef3c7`, texto `#92400e`, borda esquerda das linhas `#f59e0b`
  - **Inclusões**: tom verde — header `#15803d`, badge `#dcfce7`, texto `#14532d`, borda esquerda das linhas `#22c55e`
  - **Exclusões**: tom vermelho — header `#b91c1c`, badge `#fee2e2`, texto `#7f1d1d`, borda esquerda das linhas `#ef4444`

Cada seção contém uma tabela com colunas:
- **#** (número sequencial)
- **Arquivo** (nome do arquivo com ícone de extensão simples em texto)
- **Caminho** (caminho relativo, em fonte mono menor)
- **Status** (código git, pequeno badge colorido)

Se a seção estiver vazia: exibir mensagem centralizada "Nenhum item nesta categoria" com ícone ✓.

### Rodapé
- Texto: "Gerado automaticamente por Claude Code"
- Data/hora de geração

### JavaScript inline
- Função de toggle para cada seção (click no header)
- Rotação do ícone de seta ao expandir/colapsar
- Sem jQuery, apenas vanilla JS

## 5. Salvar e reportar

- Salve o arquivo HTML no caminho: `<pasta-de-destino>\relatorio-alteracoes-<YYYYMMDD-HHmm>.html`
- Após salvar, informe ao usuário:
  - Caminho completo do arquivo gerado
  - Resumo: X alteração(ões), Y inclusão(ões), Z exclusão(ões)
  - Total de artefatos impactados
