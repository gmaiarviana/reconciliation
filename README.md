# Reconciliation

Repositório do projeto de conciliação de benefícios.

## Resumo Executivo

Este repositório consolida a Fase 1 da conciliação de benefícios, com foco no cruzamento entre fatura do fornecedor e referência interna do DP por matrícula do colaborador.

Pontos-chave:

1. A conciliação é feita por `matricula` com `join outer`, capturando diferenças e ausências em ambos os lados.
2. O campo prioritário de validação é o desconto do colaborador (valor que impacta a folha).
3. Há tolerância de R$ 0,05 para absorver variações de arredondamento.
4. A arquitetura é extensível por parser, permitindo incluir novos fornecedores sem alterar o fluxo principal.
5. O núcleo de negócio (`conciliar`, `parse_fatura_*`, classificação de divergências) está preparado para reaproveitamento nas próximas fases.

As decisões, contratos e regras completas permanecem registradas integralmente na seção técnica abaixo.

---

## Estrutura do Repositório

```
reconciliation/
  logic.py                        ← lógica de negócio (parsers, conciliação, utilitários)
  conciliacao_beneficios.ipynb    ← notebook do Colab (interface do analista)
  README.md
  .gitignore
```

### Separação de responsabilidades

| Arquivo | O que contém | Quem edita |
|---|---|---|
| `logic.py` | Parsers, `conciliar()`, `PARSERS`, constantes | Desenvolvedor — versionado aqui |
| `notebook` | Células de UI (widgets, upload, display) | Desenvolvedor — atualizado manualmente após sync |

O notebook nunca muda a lógica diretamente. A Célula 2 contém o conteúdo do `logic.py` colado manualmente.

### Fluxo de atualização

Quando a lógica muda:

1. Editar `logic.py` no repositório
2. Colar o conteúdo atualizado na **Célula 2** do notebook no Colab
3. Atualizar o comentário de sync no topo da célula: `# logic.py · sync: AAAA-MM-DD`

Esse marcador é a forma de confirmar que o notebook está rodando a versão correta da lógica.

---

## Registro Técnico — Notebook de Conciliação de Benefícios

Instituto Atlântico | Março 2026

Este documento registra as decisões de arquitetura, estrutura de dados e lógica de negócio implementadas no notebook de conciliação (Fase 1 da Plataforma de Gestão de Benefícios). O objetivo é preservar o conhecimento necessário para que o código e a lógica sejam reaproveitados nas fases seguintes, em especial na migração para a interface web (Fase 2) e na integração com banco de dados.

## 1. Contexto

O notebook foi construído para conciliar valores de benefícios entre:

- a fatura recebida do fornecedor;
- a referência interna do Departamento Pessoal (DP).

## 2. Arquivos de entrada

O notebook opera com dois arquivos por ciclo de conciliação.

### 2.1 Fatura do fornecedor

Arquivo Excel enviado mensalmente pelo fornecedor. Cada fornecedor tem seu próprio formato, por isso existe um parser dedicado para cada um (ver seção 4).

Exemplo: Bradesco Dental.

| Campo | Localização | Observação |
|---|---|---|
| Matricula Titular | Aba `bradesco`, coluna Q | Chave de cruzamento. Presente no titular e nos dependentes (todos com a matrícula do titular). |
| DESCONTO COLABORADOR | Aba `bradesco`, coluna V | Valor descontado do colaborador por vida (titular ou dependente). |
| Custo (coluna X) | Aba `bradesco`, coluna T | Custo para a empresa por vida. |
| Certif. | Aba `bradesco`, coluna B | Identifica o grupo familiar (ex.: `0000019/00` = titular, `0000019/01` = dependente). |

O arquivo possui uma linha por pessoa (titular ou dependente). Como a conciliação é por colaborador, os valores são agregados por Matricula Titular antes do cruzamento.

### 2.2 Referência interna do DP

Arquivo Excel mantido pelo Departamento Pessoal com os valores esperados por colaborador. Estrutura atual: `descontos_proventos_beneficios_MMAAAA.xlsx`, aba `total`.

| Campo | Localização | Observação |
|---|---|---|
| Mat | Aba `total`, cabeçalho linha 2 | Matrícula do colaborador, chave de cruzamento. |
| Nome | Aba `total`, cabeçalho linha 2 | Nome do colaborador. |
| Bradesco Dental > Fatura | Aba `total`, sub-cabeçalho linha 3 | Valor total esperado da fatura. |
| Bradesco Dental > Desconto | Aba `total`, sub-cabeçalho linha 3 | Desconto esperado do colaborador, campo usado na conciliação. |
| Bradesco Dental > Custo | Aba `total`, sub-cabeçalho linha 3 | Custo esperado para a empresa. |

O cabeçalho é em duas linhas: a primeira com o nome do fornecedor (Bradesco Dental) e a segunda com os subcampos (Fatura, Desconto, Custo). O parser localiza os índices dinamicamente pela leitura desses cabeçalhos, sem posição fixa.

## 3. Lógica de conciliação

### 3.1 Chave de cruzamento

- `matricula` (inteiro), presente nos dois arquivos.
- O `join` é `outer` para capturar ausências em qualquer lado.

### 3.2 Campo conciliado

- `DESCONTO COLABORADOR` (fatura) versus `Bradesco Dental > Desconto` (referência interna).
- A escolha pelo desconto reflete o processo do DP: é o valor que precisa bater na folha.

### 3.3 Tolerância

- `0.05` (R$ 0,05).
- Diferenças dentro desse intervalo são classificadas como `OK` para absorver variações de arredondamento de ponto flutuante.

### 3.4 Classificação de divergências

| Status | Condição |
|---|---|
| OK | `abs(diferenca) <= tolerancia` |
| Só na fatura | Presente na fatura e ausente na referência (`desconto_esperado == 0`). |
| Só na referência | Presente na referência e ausente na fatura (`desconto_fatura == 0`). |
| Divergência de valor | Presente nos dois lados, mas diferença fora da tolerância. |

## 4. Arquitetura de parsers

### 4.1 Registro

Cada fornecedor é representado por uma função `parse_fatura_<nome>(conteudo: bytes) -> DataFrame` e registrada no dicionário `PARSERS`:

```python
PARSERS = {
    ("bradesco", "dental"): parse_fatura_bradesco_dental,
    # ("unimed",):           parse_fatura_unimed,
    # ("uniodonto",):        parse_fatura_uniodonto,
}
```

A chave é uma tupla de substrings que devem estar presentes no nome do arquivo (em lowercase). A identificação é automática no momento do upload.

### 4.2 Contrato da função de parse

Toda função de parse deve:

1. Receber `conteudo: bytes`.
2. Retornar um DataFrame com as colunas:

| Coluna | Tipo | Descrição |
|---|---|---|
| matricula | int | Matrícula do colaborador |
| desconto_fatura | float | Desconto total do colaborador (soma de todas as vidas) |
| custo_fatura | float | Custo total para a empresa |
| qtd_vidas | int | Número de vidas cobertas (titular + dependentes) |

3. Agregar por `matricula` quando o arquivo tiver uma linha por vida (como no Bradesco Dental).
4. Lançar exceção com mensagem descritiva em caso de estrutura inesperada.

### 4.3 Identificação da referência interna

A referência interna é identificada pelo nome do arquivo com presença de `descontos` e `proventos` (em lowercase). Esse critério está na constante `REFERENCIA_KEYWORDS` e pode ser ajustado sem alterar o fluxo principal.

### 4.4 Adicionando um novo fornecedor

1. Criar a função `parse_fatura_<nome>(conteudo)` respeitando o contrato.
2. Registrar em `PARSERS` com a chave de identificação pelo nome do arquivo.
3. Atualizar a tabela de fornecedores suportados no cabeçalho do notebook.

O fluxo de upload, identificação, conciliação e exibição de resultado não precisa de alteração.

## 5. Decisões registradas

| Decisão | Escolha | Motivo |
|---|---|---|
| Chave de cruzamento | Matrícula | Identificador único e estável; nome do colaborador pode variar em grafia. |
| Campo de conciliação | Desconto do colaborador | É o valor que impacta a folha, prioridade do processo do DP. |
| Agregação na fatura | Por Matricula Titular | Faturas com dependentes têm uma linha por vida; conciliação é por colaborador. |
| Localização de colunas | Dinâmica (por cabeçalho) | Posições variam entre fornecedores e podem mudar com atualizações. |
| Tolerância numérica | R$ 0,05 | Cobre arredondamento de ponto flutuante sem mascarar divergências reais. |
| Join | `outer` | Captura os casos "só na fatura" e "só na referência". |
| Código oculto no Colab | `#@title` + `display-mode: form` | Reduz ruído e risco de edição acidental pelo analista. |
| Upload por arquivo | Botão repetível + confirmação explícita | Arquivos podem estar em diretórios diferentes na máquina do analista. |
| Export opcional | `checkbox exportar_excel` | Download automático é intrusivo; o analista decide quando exportar. |
| Separação logic/notebook | `logic.py` versionado no GitHub | Permite diff legível, edição cirúrgica e reaproveitamento nas fases seguintes. |
| Sync manual | Conteúdo colado na Célula 2 | Elimina dependência de autenticação GitHub no Colab; fricção controlada por convenção de marcador de data. |

## 6. O que será reaproveitado nas próximas fases

| Componente | Reaproveitamento esperado |
|---|---|
| `parse_fatura_*` | Migração direta para a interface web (Fase 2), mantendo a lógica com nova camada de entrada. |
| `parse_referencia_interna` | Substituição gradual conforme a referência interna migra para banco (Fase 2). |
| `conciliar` | Núcleo da lógica de negócio, reaproveitado em todas as fases. |
| `PARSERS` | Expansão contínua a cada novo fornecedor incorporado. |
| Classificação de divergências | Base para status no painel web (Fase 2) e alertas automáticos (Fase 5). |

---

Documento em evolução, atualizado a cada ciclo de desenvolvimento.