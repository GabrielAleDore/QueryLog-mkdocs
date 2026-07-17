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
| DELETE | Perda irreversível de dados, quebra de integridade referencial, impacto em relatórios e processos dependentes. 	 |
| UPDATE | Sobrescrita indevida de registros, inconsistência lógica, corrupção de regras de negócio. 						 |
| INSERT | Inserção de dados inválidos, duplicidade, violação de chaves e restrições. 									 	 |

## Diretrizes para operações seguras

Antes de executar qualquer operação, recomenda-se

1. Validar previamente os dados a serem modificados
2. Avaliar o impacto das operações no sistema
3. Realizar backups dos dados antes de realizar operações
4. Executar preferencialmente dentro de transações controladas, permitindo ROLLBACK em caso de inconsistências.
5. Garantir rastreabilidade, registrando evidências da execução.

## Operações:

### DELETE
	
!!! info " "
	A opração `DELETE` remove registros permanentemente da tabela. Deve ser executada com cautela, pois não pode ser desfeita.

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

!!! warning " "
    - Não executar `DELETE` sem a cláusula `WHERE` em ambientes de produção
    - Validar possiveis vínculos referenciados FK ou PK
    - Confirmar impacto no sistema

### UPDATE

!!! info " "
	A operação `UPDATE` altera dados existentes e pode gerar inconsistência nos dados se aplicada incorretamente.

#### Validação prévia
```SQL linenums="1"
-- Validação dos registros que serão alterados
SELECT 
	*
FROM
	NOME_TABELA
WHERE
	PLANIREGISTRO = 456;
```

#### Backup pontual
```SQL linenums="1"
CREATE TABLE TMP.NOME_TABELA_BACKUP

INSERT INTO TMP.NOME_TABELA_BACKUP
SELECT 
	*
FROM
	NOME_TABELA
WHERE
	PLANIREGISTRO = 456;
```

#### Execução do UPDATE
```SQL linenums="1"
UPDATE
	NOME_TABELA
SET
	IDREGISTRO = 456
WHERE
	PLANIREGISTRO = 456;
```

!!! warning " "
    - Não executar `UPDATE` sem a cláusula `WHERE` em ambientes de produção
    - Validar possiveis vínculos referenciados FK ou PK
    - Confirmar impacto no sistema

### INSERT

!!! info "A operação `INSERT` insere registros na tabela. Deve ser executada com cautela, pois pode gerar inconsistência nos dados se aplicada incorretamente."

#### Insersão controlada(Log)
```SQL linenums="1"
-- Criação de uma tabela de log
CREATE TABLE TMP.NOME_TABELA_LOG LIKE NOME_TABELA;

-- Inserção dos registros para validação 
INSERT INTO TMP.NOME_TABELA_LOG
VALUES (1, 'Dado 1', 'Dado 2', 'Dado 3');

-- Insertção definitiva na tabala principal
INSERT INTO NOME_TABELA
SELECT 
	*
FROM
	TMP.NOME_TABELA_LOG;
```

#### Insert com multiplos valores
```SQL linenums="1"
-- Insertção com multiplos valores
INSERT INTO TMP.NOME_TABELA_LOG
VALUES (1, 'Dado 1', 'Dado 2'),
	   (2, 'Dado 1', 'Dado 2'),
	   (3, 'Dado 1', 'Dado 2');

INSERT INTO NOME_TABELA (ID, DATA, VALOR)
SELECT 
	ID,
	DATA,
	VALOR
FROM
	TMP.NOME_TABELA_LOG;
```

!!! warning "Atenção"
    - Validar existência prévia do registro (ex.: chave primária).
    - Conferir constraints e regras de integridade.
    - Preferir uso de tabelas de staging para grandes volumes.
    - Monitorar logs e erros de execução.



## Considerações

!!! danger "Importante"
	Importante reforçar que os exemplos apresentados acima são meramente ilustrativos.

	Em ambientes de produção, as operações são consideravelmente mais complexas e técnicas, e a realização de um backup completo de uma tabela geralmente não é viável, seja por questões de desempenho, volume de dados ou políticas de governança. Por isso, é fundamental sempre realizar backups pontuais e direcionados apenas aos dados que efetivamente serão manipulados.

	Além disso, é imprescindível adotar uma postura de extrema cautela ao executar qualquer alteração em ambientes de produção, especialmente considerando que se trata de sistemas críticos e de responsabilidade direta com o cliente.
	A ausência desses cuidados se caracteriza como um incidente de segurança, podendo comprometer a integridade dos dados e a confiança no ambiente.

??? tip "Dica"
	Em caso de dúvidas ou necessidades específicas, é possível acionar o time de script, que está disponível para fornecer apoio técnico e garantir a correta execução das operações.

	Time de script. 
	Gabriel Alessandro Doré 
	Adriana Sarturi 
	Otavio Augusto Colares Cutrim