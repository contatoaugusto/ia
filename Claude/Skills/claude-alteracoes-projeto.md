Gere um relatório HTML de alterações Git com base nos argumentos fornecidos: $ARGUMENTS

O formato dos argumentos é: <caminho-projeto-1>,<caminho-projeto-2>,...,<caminho-projeto-N> <pasta-de-destino>
- Os caminhos dos projetos são separados por vírgula (sem espaço entre eles)
- O último argumento (separado por espaço após os caminhos) é a pasta de destino do HTML
- Exemplos:
  - Um projeto:   D:\TFS\SGI_GIT D:\Relatorios
  - Vários:       D:\TFS\SGI_GIT,D:\TFS\OUTRO_PROJETO,C:\Dev\App D:\Relatorios

Siga exatamente estes passos:

## 1. Extrair argumentos

- Separe o último token (separado por espaço) como `<pasta-de-destino>`
- Todo o restante antes do último espaço é a lista de projetos — divida pelo caractere `,` para obter cada `<caminho-projeto>`
- Se a pasta de destino não existir, crie-a
- O nome do arquivo gerado deve ser: `relatorio-alteracoes-<YYYYMMDD-HHmm>.html`

## 2. Coletar dados do Git — repetir para CADA projeto

Para cada `<caminho-projeto>` da lista, execute os seguintes comandos:

```
git -C "<caminho-projeto>" status --porcelain
git -C "<caminho-projeto>" log -1 --pretty=format:"%H|%an|%ae|%ad|%s" --date=format:"%d/%m/%Y %H:%M"
git -C "<caminho-projeto>" rev-parse --abbrev-ref HEAD
git -C "<caminho-projeto>" remote get-url origin 2>/dev/null || echo "sem-remoto"
```

Agrupe os resultados por projeto para exibição separada no HTML.

## 3. Classificar arquivos

Com base na primeira(s) letra(s) de cada linha do `git status --porcelain`:

| Código     | Categoria                  |
|------------|----------------------------|
| M, MM, AM  | Alterações                 |
| R, RM      | Alterações (renomeado)     |
| A          | Inclusões                  |
| ??         | Inclusões (não rastreado)  |
| D, MD      | Exclusões                  |
| C          | Alterações (copiado)       |

Para cada arquivo extraia:
- Nome do arquivo (sem o caminho)
- Caminho relativo completo
- Extensão
- Status exato (código git)
- Projeto de origem (nome do projeto — último segmento do caminho)

## 4. Gerar o HTML

Gere um arquivo HTML completo e autocontido com as seguintes características:

### Estrutura e layout
- Sem dependências externas (CSS e JS inline, sem CDN)
- Design leve, moderno e responsivo
- Fonte: system-ui / sans-serif
- Fundo da página: #f0f2f5
- Largura máxima do conteúdo: 1200px, centralizado

### Cabeçalho da página
- Título: "Relatório de Alterações Git"
- Data/hora de geração do relatório
- Cards de resumo global (consolidado de todos os projetos): total de Alterações, Inclusões, Exclusões e Total Geral — cada card com a cor da respectiva seção

### Bloco por projeto (repetir para cada projeto)

Cada projeto deve ter seu próprio bloco com:
- Título do bloco: nome do projeto (último segmento do caminho) em destaque
- Informações: branch atual, último commit (hash 7 chars, autor, data, mensagem), remoto
- Cards de resumo individuais do projeto (Alterações, Inclusões, Exclusões)
- As 3 seções collapsible independentes (Alterações, Inclusões, Exclusões) do projeto

Se houver apenas um projeto, omitir o agrupamento por bloco e exibir diretamente as seções.

### Seções collapsible (3 por projeto, independentes)

Cada seção deve:
- Ter um cabeçalho clicável que expande/colapsa o corpo
- Iniciar **expandida** por padrão
- Exibir no cabeçalho: ícone ▸/▾, nome da seção e contador de itens (badge)
- Ter animação suave de expand/collapse (max-height transition CSS)
- Ter cor distinta:
  - **Alterações**: tom âmbar/laranja — header `#b45309`, badge `#fef3c7`, texto `#92400e`, borda esquerda das linhas `#f59e0b`
  - **Inclusões**: tom verde — header `#15803d`, badge `#dcfce7`, texto `#14532d`, borda esquerda das linhas `#22c55e`
  - **Exclusões**: tom vermelho — header `#b91c1c`, badge `#fee2e2`, texto `#7f1d1d`, borda esquerda das linhas `#ef4444`

Cada seção contém uma tabela com colunas:
- **#** (número sequencial)
- **Arquivo** (nome do arquivo)
- **Caminho** (caminho relativo, fonte mono tamanho menor)
- **Status** (código git, pequeno badge colorido)

Se a seção estiver vazia: exibir mensagem centralizada "Nenhum item nesta categoria" com ícone ✓.

### Rodapé
- Texto: "Gerado automaticamente por Claude Code"
- Data/hora de geração
- Lista dos projetos incluídos no relatório

### JavaScript inline (vanilla JS, sem jQuery)
- Função de toggle genérica reutilizável para qualquer seção
- IDs únicos por seção por projeto (ex: `sec-alteracoes-projeto1`)
- Rotação do ícone ▸ → ▾ ao expandir/colapsar
- Botões "Expandir todos" / "Recolher todos" no topo de cada bloco de projeto

## 5. Salvar e reportar

- Salve o arquivo HTML em: `<pasta-de-destino>\relatorio-alteracoes-<YYYYMMDD-HHmm>.html`
- Após salvar, informe ao usuário:
  - Caminho completo do arquivo gerado
  - Por projeto: X alteração(ões), Y inclusão(ões), Z exclusão(ões)
  - Total consolidado de artefatos impactados em todos os projetos
