# Comandos IMPORT e EXPORT

Processo para realizar uma exportação de dados do banco de dados dos clientes em casos de recuperação ou execução de serviços.

## Dependencias do processo

Saber [Conectar no banco de dados](../../banco-de-dados/utilitarios/conectar-banco-de-dados.md){ data-preview } atraves do terminal Linux ou Windows.

Saber [navegar nos diretórios do Linux ou Windows](*){ data-preview } pelo terminal.

## Comando export
````bash
-- Exemplo de comando export
db2 export to <nome-arquivo> OF <formato> <"busca dos dados">

-- Exemplo prático
db2 export to ESTOQUE_SINTETICO.IXF OF IXF <"SELECT * FROM ESTOQUE_SINTETICO WHERE IDEMPRESA = 1 AND DTMOVIMENTO = '2025-01-01'">
````

### Parametros do comando export

- **TO <nome-arquivo>**: Especifica o nome do arquivo para onde os dados serão exportados (Ex: arquivo.ixf).
- **OF <formato>**: Especifica o formato do arquivo 
    - **IXF**: (Integration Exchange Format, versão para PC) é um formato binário proprietário.
    - **DEL**: (formato ASCII delimitado), pode ser usado em varios progrmas, mas normalmente é convertido em XLSX por ser semelhante ao formato CSV.

Após o Export, deve-se direcionar o arquivo ao banco de dados do cliente. 
Em caso do cliente ser DataCISS validar o arquivo [Importar arquivo para OCI](*){ data-preview } para imrpotar o arquivo de uma base para outra.

