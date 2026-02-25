# Boas Práticas Desenvolvimento SQL

As operações `DELETE`, `UPDATE` e `INSERT` são fundamentais para a manipulação de dados. Mas também representam riscos significativos quando executadas sem critérios adequados, podendo comprometer:

- Integridade dos dados
- Consistencia das informações
- Disponibilidade do sistema
- Confiabilidade das operações

Esta documentação aborda boas práticas para garantir a **integridade**, **segurança** e **disponibilidade** dos dados durante essas operações.

## Riscos associados ás operações

| Operação | Principais Riscos |
| --- | -------------------------------------------------------------------------------------------------------------------- |
| DELETE | Perda irreversível de dados, quebra de integridade referencial, impacto em relatórios e processos dependentes. |
| UPDATE | Sobrescrita indevida de registros, inconsistência lógica, corrupção de regras de negócio. |
| INSERT | Inserção de dados inválidos, duplicidade, violação de chaves e restrições. |

## Diretrizes para operações seguras

Antes de executar qualquer operação, recomenda-se

1. Validar previamente os dados a serem modificados
2. Avaliar o impacto das operações no sistema
3. Realizar backups dos dados antes de realizar operações
4. Executar preferencialmente dentro de transações controladas, permitindo ROLLBACK em caso de inconsistências.
5. Garantir rastreabilidade, registrando evidências da execução.

## Operações:

### DELETE
	
!!! info "A opração `DELETE` remove registros permanentemente da tabela. Deve ser executada com cautela, pois não pode ser desfeita."

#### Validação právia
```SQL linenums="1"
-- Validação dos registros que serão removidos
SELECT 
	*
FROM
	NOME_TABELA
WHERE
	PLANIREGISTRO = 123;
```

#### Backup pontual
```SQL linenums="1"
-- Criação de uma tablea temporária para armazenar os registros que serão removidos
CREATE TABLE TMP.NOME_TABELA_BACKUP

-- Inserção dos registros que serão removidos na tabela temporária
INSERT INTO TMP.NOME_TABELA_BACKUP
SELECT 
	*
FROM
	NOME_TABELA
WHERE
	PLANIREGISTRO = 123;
```

```SQL linenums="1"
-- Remoção dos registros da tabela
DELETE FROM
	NOME_TABELA
WHERE
	PLANIREGISTRO = 123;
```

#### Observações:
- Não executar `DELETE` sem a cláusula `WHERE` em ambientes de produção
- Validar possiveis vínculos referenciados FK ou PK
- Confirmar impacto no sistema


