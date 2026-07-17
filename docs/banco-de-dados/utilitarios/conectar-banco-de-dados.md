# Conectar no banco de dados

Conectar ao IBM DB2 via linha de comando exige que você entenda a hierarquia do banco: primeiro o sistema operacional, depois a instância (o processo que gerencia os bancos) e, por fim, o banco de dados em si.

## Guia Pratico

### No linux (Terminal)

No linux, o DB2 utiliza usuário do sistema operacional para gerenciar instancias. O administrador padrão geralmente é o usuário `db2inst1`.

#### Acessar usuário ROOT

Ao abrir o terminal, é preciso mudar para o superusuário ROOT 

```bash
sudo su -
```

#### Acessar usuário da instância

Você não deve rodar comandos do DB2 como root. É necessário alternar para o usuário que "é" a instância:

```bash
su - db2inst1
```
Isso carrega automaticamente as variáveis de ambiente necessárias (como o db2profile).

#### Conectar no banco de dados

Para se conectar no banco de dados, utilize o comando `db2 connect`.

```bash
db2 connect to <nome-banco-de-dados> user dba
```
Ao rodar o comando acima será solicitado a senha do usuário `dba`.


### No windows

No windows, o DB2 instala um atalho específico que já configura o ambiente. O uso do CMD comum não funciona pois não carrega as variáveis de ambiente necessárias.

#### Abrir DB2CMDADMIN

Normalmente o atalho se encontra em: `C:\Program Files\IBM\SQLLIB\BIN\db2cmdadmin`

#### Verificar se esta na instancia correta

Diferente do linux, no windows ao abrir o terminal a instancia já é carregada automaticamente, sendo necessario conferir se esta na correta.

Existem duas maneira de conferir:

1. `db2list`: Apresenta a instância atual logada

2. `db2 list db directory`: Lista os bancos disponiveis na instancia atual, assim você verifica se o banco que deseja esta disponivel.

#### Conectar no banco de dados

Nessa etapa o windows segue o mesmo processo que o linux.

Para se conectar no banco de dados, utilize o comando `db2 connect`.

```bash
db2 connect to <nome-banco-de-dados> user dba
```

Ao rodar o comando acima será solicitado a senha do usuário `dba`.

!!! tip "Comandos Úteis tanto no windows quato no linux"
    - `db2 lis db directory`: Lista todos os bancos de dados disponiveis na instancia atual.
    - `db2list`: Lista as instancias
    - `db2 get connection state`: Verifica o estado da conexão atual.