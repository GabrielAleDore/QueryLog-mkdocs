# Processo de Desativação de Controle de Lote

Este documento descreve o processo do script para **Desativação do controle de lote** para subprodutos específicos.

O script foi desenvolvido para permitir a desativação segura do controle de lotes de um subproduto que já foi movimentado com algum lote, minimizando o impacto no sistema e garantindo a integridade dos dados por meio de Backups, Controle de triggers, log de execução e [Reprocessamento de estoque](../documentos/reprocessamento-estoque.md).

## Visão Geral

O processo de Desativação do controle de lote é composto pelas seguintes etapas:

1. Criação de tabelas temporárias de Backup, controle e Log
2. Backup dos dados críticos de estoque
3. Backup e remoção temporarria dos Triggers relacionados ao controle de lote
4. Desativação do controle de lote
5. Restauração dos triggers removidos
6. Registro de execução no log
7. Reprocessamento de estoque
8. Validaçòes pós-processamento

## Estrutura do processo

### Tabelas de controle e Backup

Criadas no schema `TMP`, espelham a estrutura original das tabelas de produção

- `TMP.BKP_ESTOQUE_SINTETICO`
- `TMP.BKP_ESTOQUE_ANALITICO`
- `TMP.BKP_ESTOQUE_SALDO_ATUAL`
- `TMP.BKP_ESTOQUE_BALANCO_ENCERRADO`

Com a finalidade de armarzenar cópias de segurança dos dados que serão tratados durante a execução do processo.

### Tabela backup de triggers

```SQL linenums="1"
CREATE TABLE TMP.TRIGGER_BKP (
    TRIGNAME    VARCHAR(128),
    TRIGSCHEMA  VARCHAR(128),
    TABNAME     VARCHAR(128),
    BODY        BLOB
)
```

Responsavel por armazenar as triggers relacionadas ao controle de lote que serão removidas temporariamente durante a execução do processo.

### Tabela log de execução

Responsavel por controlar quais produtos já foram processados e quais ainda precisam ser processados.

```SQL linenums="1"
CREATE TABLE TMP.LOG_DESATIVACAO_CONTROLE_LOTE (
    IDSUBPRODUTO INTEGER,
    DTEXECUCAO   TIMESTAMP DEFAULT CURRENT_TIMESTAMP(6),
    FLAGAJUSTADO VARCHAR(1)
)
```

### Procedures

??? note "TMP.SP_DESATIVA_CONTROLE_LOTE"
    Parâmetros de entrada:
    - AI_IDSUBPRODUTO

    Sua função é executar todo o processo de desativação de controle de lote para um subproduto específico passado como parametro.

## Execução

Este é todo o processo que deve ser seguido para realizar a desativação do controle de lote:

### Backup e preparação

1. Verificar a existência das tabela de backup e log, caso não existam, criar as tabelas:

```sql linenums="1"
CREATE TABLE TMP.BKP_ESTOQUE_SINTETICO          LIKE ESTOQUE_SINTETICO;
CREATE TABLE TMP.BKP_ESTOQUE_ANALITICO          LIKE ESTOQUE_ANALITICO;
CREATE TABLE TMP.BKP_ESTOQUE_SALDO_ATUAL        LIKE ESTOQUE_SALDO_ATUAL;
CREATE TABLE TMP.BKP_ESTOQUE_BALANCO_ENCERRADO  LIKE ESTOQUE_BALANCO_ENCERRADO;

CREATE TABLE TMP.TRIGGER_BKP (
    TRIGNAME    VARCHAR(128),
    TABNAME     VARCHAR(120),
    TEXT        CLOB(2097152),
    TRIGSCHEMA  VARCHAR(128),
    TABSCHEMA   VARCHAR(128)
);

CREATE TABLE TMP.LOG_DESATIVACAO_CONTROLE_LOTE (
    IDSUBPRODUTO INTEGER,
    DTEXECUCAO   TIMESTAMP DEFAULT CURRENT_TIMESTAMP(6),
    FLAGAJUSTADO VARCHAR(1)

);
```

2. Fazer a chama da procedure `CALL TMP.SP_DESATIVAR_CONTROLE_LOTE(AI_IDSUBPRODUTO)` passando como parametro o id do subproduto que deseja desativar o controle de lote.

3. na chamada a procedure irá:

    - Fazer backup dos dados das tabelas `ESTOQUE_SINTETICO` e `ESTOQUE_SALDO_ATUAL` somente para os subprodutos que entrarão no processo.
    - Identificar e armazenar as trigger relacionadas a `IDLOTE` na `TMP.TRIGGER_BKP` 
    - Remover as trigger relacionadas a `IDLOTE`

    - Todos os comandos são executados dinamicamento via SQL gerados a partir de uma string, que é executada via EXECUTE IMMEDIATE;

### Desativação do controle de lote

Apos a chamada da procedure e feito os backup o processo que se segue é

- Validar todas as tabelas da base que possuem o campo `IDLOTE` no schema `DBA`
- Atualiza em toda as tavela identificas o campo `IDLOTE` para `NULL` aplicando o filtro obrigatório do `IDSUBPRODUTO`
- Remove registros das tabelas que compoe o controle de lote 
- Desmarca o `FLAGLOTE` na tabela `PRODUTO_GRADE`

### Finalização

Apos a desativação do controle de lote, é feito a restauração das triggers e ajustado a tabela de log `FLAGAJUSTADO = 'T'`

Apos o processo finalizar é necessario realizar a execução do [Reprocessamento de estoque](../documentos/reprocessamento-estoque.md){ datapreview }