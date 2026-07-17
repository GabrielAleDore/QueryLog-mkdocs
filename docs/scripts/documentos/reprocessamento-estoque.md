---
tags:
  - Script
  - Estoque
---
# Reprocessamento de Estoque
Este documento descreve o **Processo de reprocessamento de estoque via script**, implementado por meio de tabales de controle e um conjunto de procedure. O Objetivo principal é **Recalcular saldo de estoque (Analitico, sentetico e saldo atual)** em cenários de divergencia, ajuste de balanços, exclusões indevidas de moviemntos e falhas de processamento anteriores.

!!! tip "Pré-requisito"
    - Sistemas na versão 22.0.2.300 ou superior.

## Visão Geral

O processo de Reprocessamento de estoque desenvolvido pelo time de script tem como Objetivo:

- Reprocessar o saldo de estoque em cenários de divergencia, considerando balanços de estoque;

- Ajustar o saldo das tabelas `ESTOQUE_SINTETICO` e `ESTOQUE_SALDO_ATUAL`;

- Garantir a integridade do estoque, evitando divergencias;

- Evitar locks e impactos de desempenho em produção.

## Estrutura de Controle

Todo o controle do reprocessamento é feito por meio de tabelas de controle e um conjunto das procedures.

### Tabelas

??? note "SCRIPT.CONTROLE_REPROEST"
    Responsavel pro controlar quais produtos, apartir de qual data, quais empresas e locais de estoque precisam ser reprocessados.

    ```sql title="Principais campos"
    -- Compoe a chave primaria da tabela, controla o que vai ser reprocessado
    IDEMRPESA       INTEGER NOT NULL,
    IDPRODUTO       INTEGER NOT NULL,
    IDLOCALESTOQUE  INTEGER NOT NULL,

    -- Definine o periodo que vai ser reprocessado
    DTMOVIMENTO     DATE    NOT NULL,

    -- Define o status do processamento (F - Pendente, B - Balanços, T - Reprocessado)
    FLAGPROCESSADO  CHAR(1) NOT NULL DEFAULT 'F',

    -- Campos para gerar uma estimativa de termino do reprocessamento
    QTDLINHAS       INTEGER,
    DTINICIO        TIMESTAMP,
    DTFIM           TIMESTAMP
    ```

??? note "SCRIPT.REPROEST"
    Tabela Intermediaria que armazena os movimentos analiticos que serão efetivamente reprocessados.

    **Função principal:**

    - Ordernar corretamente os movimentos analiticos para que o reprocessamento seja feito na ordem correta;

    - Permirtir processamento em blocos(Exemplo: 1000 registros por vez);

    - Identificar se o movimento é Balanço ou não

### Procedures
??? note "SCRIPT.SP_REPROEST_AUT"
    Atua como o cerebro de todo o processo, a SCRIPT.SP_REPROEST_AUT gerencia o loop de reprocessamento para garantir que cada item seja tratado com precisão. O processamento é realizado de forma individualizada, tratando um produto e um local por vez, o que evita sobrecargas e erros de conciliação. Para assegurar a estabilidade do banco de dados e a segurança das informações, a procedure realiza commits intermediários durante a execução, mantendo simultaneamente a atualização constante das informações de progresso para que o status da operação possa ser monitorado em tempo real.

??? note "SCRIPT.SP_REPROEST"
    A procedure SCRIPT.SP_REPROEST atua no coração da operação, sendo a responsável direta pelo reprocessamento efetivo do saldo de estoque e pela garantia de que todos os movimentos respeitem a ordem cronológica e lógica dos eventos. Para otimizar a performance e a organização dos dados, ela utiliza a estratégia técnica de tabelas temporárias globais, especificamente a SESSION.TMP_EST2, limitando o processamento a lotes de até 2000 movimentos por execução para manter a estabilidade do sistema.

??? note "SCRIPT.SP_REPROCESSA_BALANCO_NORMAL"
    A procedure SP_REPROCESSA_BALANCO_NORMAL exerce o papel fundamental de recalcular os balanços de estoque, sendo especificamente aplicada a produtos que não possuem controle de lote.

??? note "SCRIPT.SP_REPROCESSA_BALANCO_LOTE"
    A procedure SP_REPROCESSA_BALANCO_LOTE é a responsável por recalcular os balanços de estoque especificamente para produtos que possuem o controle de lote ativado. Diferente do processo simplificado, esta rotina realiza uma varredura detalhada em todos os lotes ativos do produto, garantindo que cada subdivisão do estoque seja devidamente validada.

## Reprocessamento

Passao a passo para realizar o reprocessamento de estoque:

### Carga de produtos

Temos duas opções para carga de produtos:

??? note "Carga geral de produto, definindo apenas a data inicial, empresa e local de estoque" 
    ```sql linenums="1"
    INSERT INTO SCRIPT.CONTROLE_REPROEST ( IDEMPRESA, IDPRODUTO, IDLOCALESTOQUE, DTMOVIMENTO, FLAGPROCESSADO )
    SELECT
        EA.IDEMPRESABAIXAEST,
        EA.IDPRODUTO,
        EA.IDLOCALESTOQUE,
        :RA_DTINI,
        'F'
    FROM
        DBA.ESTOQUE_ANALITICO AS EA
        JOIN DBA.OPERACAO_INTERNA AS OP ON
            EA.IDOPERACAO = OP.IDOPERACAO
        JOIN DBA.PRODUTO AS P ON
            EA.IDPRODUTO = P.IDPRODUTO
    WHERE
        NOT ( EA.FLAGMOVSALDOPRO = 'F' AND OP.TIPOITEMCATEGORIA NOT IN ('H8','P1','D11','A7', 'A5') ) AND -- Regra da SP_INSERT_ESTOQUE
        EA.DTMOVIMENTO >= :RA_DTINI AND
        EA.IDEMPRESA = :RA_IDEMPRESA
    GROUP BY
        EA.IDEMPRESABAIXAEST,
        EA.IDPRODUTO,
        EA.IDLOCALESTOQUE
    UNION
    -- Tratado novamente questão de item mestre, pois podem existir linhas de itens filhos na estoque_sintetico ( casos de suporte por exemplo, updates e inserts incorretos)
    SELECT
        ES.IDEMPRESA,
        ES.IDPRODUTO,
        ES.IDLOCALESTOQUE,
        :RA_DTINI,
        'F'
    FROM
        DBA.ESTOQUE_SINTETICO AS ES
        JOIN DBA.PRODUTO AS P ON
            ES.IDPRODUTO = P.IDPRODUTO    
    WHERE
        ES.DTMOVIMENTO >= :RA_DTINI AND
        ES.IDEMPRESA = :RA_IDEMPRESA
    GROUP BY
        ES.IDEMPRESA,
        ES.IDPRODUTO,
        ES.IDLOCALESTOQUE
        
    UNION
    -- Tratar para inserir contagens de balanço caso não tenham registro na estoque_analitico
    SELECT
        ESTOQUE_BALANCO_ENCERRADO.IDEMPRESA,
        ESTOQUE_BALANCO_ENCERRADO.IDPRODUTO,
        ESTOQUE_BALANCO_ENCERRADO.IDLOCALESTOQUE,
        :RA_DTINI,
        'F'
    FROM
        DBA.ESTOQUE_BALANCO_ENCERRADO AS ESTOQUE_BALANCO_ENCERRADO
    WHERE
        ESTOQUE_BALANCO_ENCERRADO.DTBALANCO >= :RA_DTINI AND
        ESTOQUE_BALANCO_ENCERRADO.IDEMPRESA = :RA_IDEMPRESA
    GROUP BY
        ESTOQUE_BALANCO_ENCERRADO.IDEMPRESA,
        ESTOQUE_BALANCO_ENCERRADO.IDPRODUTO,
        ESTOQUE_BALANCO_ENCERRADO.IDLOCALESTOQUE;
    ```

??? note "Carga apenas para produtos que possuem divergencia entre `ESTOQUE_ANALITICO` e `ESTOQUE_SINTETICO`" 
    ```sql linenums="1"
    INSERT INTO SCRIPT.CONTROLE_REPROEST ( IDEMPRESA, IDPRODUTO, IDLOCALESTOQUE, DTMOVIMENTO, FLAGPROCESSADO )
    WITH ANALITICO AS (
    SELECT
        DTMOVIMENTO,
        IDEMPRESA,
        IDLOCALESTOQUE,
        IDPRODUTO,
        IDSUBPRODUTO,
        SUM(QTDENTRADA) AS QTDENTRAESTOQUE,
        SUM(QTDSAIDA)   AS QTDSAIDAESTOQUE,
        SUM(QTDBALANCO) AS QTDAJUSTEBALANCO,
        SUM(QTDVENDA)   AS QTDVENDA,
        SUM(QTDGIRO)    AS QTDGIRO
    FROM
        (
        SELECT
            EA.DTMOVIMENTO,
            EA.IDEMPRESABAIXAEST AS IDEMPRESA,
            EA.IDLOCALESTOQUE,
            EA.IDPRODUTO,
            (CASE WHEN P.TIPOBAIXAMESTRE = 'M' THEN EA.IDPRODUTO ELSE EA.IDSUBPRODUTO END) AS IDSUBPRODUTO,
            EA.IDOPERACAO,
            EA.FLAGMOVSALDOPRO,
            EA.FLAGESTOQUEPROCESSADO,
            COALESCE( CASE WHEN EA.IDOPERACAO > 1000 AND EA.IDOPERACAO <> 2000 AND EA.FLAGMOVSALDOPRO = 'T' THEN
                EA.QTDPRODUTO
            END,0) AS QTDSAIDA,
            COALESCE( CASE WHEN EA.IDOPERACAO < 1000 AND EA.FLAGMOVSALDOPRO = 'T' THEN
                EA.QTDPRODUTO
            END,0) AS QTDENTRADA,
            COALESCE( CASE WHEN EA.IDOPERACAO = 2000 AND EA.FLAGMOVSALDOPRO = 'T' THEN
                EA.QTDPRODUTO
            END,0) AS QTDBALANCO,
            -- BASEADO EM CALCULOS DA SP_PROCESSA_SALDO
            COALESCE( CASE
                        WHEN EA.FLAGMOVSALDOPRO = 'F' AND TIPOITEMCATEGORIA NOT IN ('A11','A14','D11') THEN
                            EA.QTDPRODUTO
                        WHEN EA.FLAGMOVSALDOPRO = 'T' AND EA.IDOPERACAO > 1000 AND
                            (
                                (
                                    (
                                        (
                                            (
                                                OP.TIPOCATEGORIA = 'A' OR
                                                ( OP.TIPOITEMCATEGORIA = 'B2' AND (SELECT FLAGVENDAANTECIPADA FROM DBA.NOTAS WHERE IDEMPRESA = EA.IDEMPRESA AND IDPLANILHA = EA.IDPLANILHA) = 'F' )
                                            ) AND
                                            EA.IDOPERACAO NOT IN (1090,3001)
                                        ) OR
                                        EA.IDOPERACAO = 1095
                                    ) AND
                                    OP.TIPOITEMCATEGORIA NOT IN ('A11','A14')
                                ) OR
                                (
                                    EA.TIPOCATEGORIA = 'D' AND
                                    OP.FLAGESTORNONFE = 'T'
                                )
                            ) THEN
                            EA.QTDPRODUTO
                        ELSE
                            0
                        END,0) AS QTDVENDA,
            -- BASEADO EM CALCULOS DA SP_PROCESSA_SALDO
            COALESCE( CASE WHEN EA.FLAGCALCULOGIRO = 'T' THEN
                        CASE
                            WHEN EA.FLAGMOVSALDOPRO = 'T' AND EA.IDOPERACAO < 1000 AND OP.TIPOCATEGORIA = 'D' AND OP.TIPOITEMCATEGORIA <> 'D6' AND
                                COALESCE((SELECT DEVOLUCAO_LOGISTICA_MOVIMENTO.FLAGGERARREENTREGA
                                        FROM   DEVOLUCAO_LOGISTICA_MOVIMENTO,
                                                NOTAS_DEVOLUCAO,
                                                NOTAS_ENTRADA_SAIDA
                                        WHERE  DEVOLUCAO_LOGISTICA_MOVIMENTO.IDEMPRESA         = NOTAS_DEVOLUCAO.IDEMPRESA             AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDPLANILHA        = NOTAS_DEVOLUCAO.IDPLANILHA            AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.NUMSEQUENCIADEV   = NOTAS_DEVOLUCAO.NUMSEQUENCIADEVOLUCAO AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDPRODUTO         = NOTAS_DEVOLUCAO.IDPRODUTO             AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDSUBPRODUTO      = NOTAS_DEVOLUCAO.IDSUBPRODUTO          AND
                                                NOTAS_ENTRADA_SAIDA.IDEMPRESA                   = NOTAS_DEVOLUCAO.IDEMPRESA             AND
                                                NOTAS_ENTRADA_SAIDA.IDPLANILHA                  = NOTAS_DEVOLUCAO.IDPLANILHADEVOLUCAO   AND
                                                NOTAS_ENTRADA_SAIDA.IDOPERACAO                  = 3001                                  AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDEMPRESA         = EA.IDEMPRESA      AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDPLANILHA        = EA.IDPLANILHA     AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.NUMSEQUENCIADEV   = EA.NUMSEQUENCIA   AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDPRODUTO         = EA.IDPRODUTO      AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.IDSUBPRODUTO      = EA.IDSUBPRODUTO   AND
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.FLAGGERARREENTREGA= 'T'
                                        GROUP BY
                                                DEVOLUCAO_LOGISTICA_MOVIMENTO.FLAGGERARREENTREGA ), 'F' ) = 'F' THEN
                                EA.QTDPRODUTO *-1
                            WHEN EA.FLAGMOVSALDOPRO = 'F' AND OP.TIPOITEMCATEGORIA = 'D11' THEN
                                EA.QTDPRODUTO *-1
                            WHEN EA.IDOPERACAO > 1000 THEN
                                EA.QTDPRODUTO
                            ELSE
                                0
                        END
                    ELSE
                        0
            END,0) AS QTDGIRO
        FROM
            DBA.ESTOQUE_ANALITICO EA
        JOIN
            DBA.PRODUTO P
                ON ( EA.IDPRODUTO = P.IDPRODUTO )
        JOIN
            DBA.OPERACAO_INTERNA AS OP ON
                    EA.IDOPERACAO = OP.IDOPERACAO
        WHERE
            NOT ( EA.FLAGMOVSALDOPRO = 'F' AND OP.TIPOITEMCATEGORIA NOT IN ('H8','P1','D11','A7', 'A5') ) AND -- REGRA DA SP_INSERT_ESTOQUE - AVALIANDO SE SOMENTE ELA SERVE
            DTMOVIMENTO >= :RA_DTINI
        ) AS TMP
    WHERE
        NOT( QTDENTRADA IS NULL AND QTDSAIDA IS NULL AND QTDBALANCO IS NULL AND QTDVENDA IS NULL AND QTDGIRO IS NULL )
    GROUP BY
        TMP.DTMOVIMENTO,
        TMP.IDEMPRESA,
        TMP.IDLOCALESTOQUE,
        TMP.IDPRODUTO,
        TMP.IDSUBPRODUTO
    ),
    SINTETICO AS
    (
    SELECT
        IDEMPRESA,
        IDLOCALESTOQUE,
        DTMOVIMENTO,
        IDPRODUTO,
        IDSUBPRODUTO,
        QTDATUALESTOQUE,
        QTDENTRAESTOQUE,
        QTDSAIDAESTOQUE,
        QTDAJUSTEBALANCO,
        QTDVENDA,
        QTDGIRO
    FROM
        ESTOQUE_SINTETICO EA
    WHERE
        DTMOVIMENTO >= :RA_DTINI
    )

    SELECT
        COALESCE( A.IDEMPRESA, S.IDEMPRESA) AS IDEMPRESA,
        COALESCE( A.IDPRODUTO, S.IDPRODUTO) AS IDPRODUTO,
        COALESCE( A.IDLOCALESTOQUE, S.IDLOCALESTOQUE) AS IDLOCALESTOQUE,    
        MIN( COALESCE( A.DTMOVIMENTO, S.DTMOVIMENTO) ) AS DTMOVIMENTO,
        'F' AS FLAGPROCESSADO     
    FROM
        ANALITICO A
    FULL JOIN
        SINTETICO S
            ON A.IDEMPRESA      = S.IDEMPRESA AND
            A.IDLOCALESTOQUE = S.IDLOCALESTOQUE AND
            A.IDPRODUTO      = S.IDPRODUTO AND
            A.IDSUBPRODUTO   = S.IDSUBPRODUTO AND
            A.DTMOVIMENTO    = S.DTMOVIMENTO
    WHERE
        (
        COALESCE( A.QTDENTRAESTOQUE,0) <> COALESCE( S.QTDENTRAESTOQUE, 0) OR
        COALESCE( A.QTDSAIDAESTOQUE,0) <> COALESCE( S.QTDSAIDAESTOQUE, 0) OR
        COALESCE( A.QTDAJUSTEBALANCO,0) <> COALESCE( S.QTDAJUSTEBALANCO, 0) OR
        COALESCE( A.QTDVENDA,0) <> COALESCE( S.QTDVENDA, 0) OR
        COALESCE( A.QTDGIRO,0) <> COALESCE( S.QTDGIRO, 0)
        )
    GROUP BY
        COALESCE( A.IDEMPRESA, S.IDEMPRESA),
        COALESCE( A.IDPRODUTO, S.IDPRODUTO),
        COALESCE( A.IDLOCALESTOQUE, S.IDLOCALESTOQUE)
    ```

!!! info " "
    Impotante resaltar que em ambas as opções acima, não é necessário passar a data final do reprocessamento, pois o mesmo será feito até a data atual que o reprocessamento começou. 
    As consultas tambem já consideram os balanços de estoque e permite separa por empresa, local de estoque.

### Runstats

Após realizar a carga na tabela de controle, é necessário realizar o runstats na tabela para que o banco de dados tenha as estatísticas atualizadas.

```sql linenums="1"
CALL ADMIN_CMD ( 'RUNSTATS ON TABLE SCRIPT.CONTROLE_REPROEST ON ALL COLUMNS WITH DISTRIBUTION AND SAMPLED DETAILED INDEXES ALL ALLOW WRITE ACCESS');
```

### Chamada da procedure

Não é necessário passar parametros para a chamada da procedure, pois ela vai trabalhar com base o dados inseridos na tabela de controle `SCRIPT.CONTROLE_REPROEST`

```sql linenums="1"
CALL SCRIPT.SP_REPROEST_AUT( );
```

## Fluxo do reprocessamento

O reprocessamento é feito em etapas

=== "SP_REPROEST_AUT"
    ```mermaid
    flowchart TD

        %% =========================
        %% FLUXO DA SP_REPROEST_AUT
        %% =========================
        
        A([🚀 Inicio SP_REPROEST_AUT]) --> B{FLAGPROCESSADO = 'F'}

        B -- Sim --> C[Busca o Primeiro Registro]
        B -- Não --> Z[✅ Fim do Reprocessamento]

        C --> D[Limpa SCRIPT.REPROEST]

        D --> E[Busca os balanços de estoque zerados]

        E --> F{Existe balanços de estoque zerado?}
        F -- Sim --> G[Insere os balanços na ESTOQUE_ANALITICO]
        F -- Não --> H[Limpa a SALDO_PROCESSAR ]
        G --> H

        H --> I[Insere os Registros na REPROEST]

        I --> J{Registro é do tipo balanço?}
        J -- Sim --> K[Ajusta o flag FLAGESTOQUEPROCESSADO para 'F' na estoque_analitico]
        J -- Não --> L[Remove as linhas de saldo]
        K --> L

        L --> M[Limpa a ESTOQUE_SINTETICO]
        M --> N[Limpa a ESTOQUE_SINTETICO_LOTE]
        N --> O[Limpa a ESTOQUE_SALDO_ATUAL]

        O --> P[Limpa os registros já processados SALDO_PROCESSADO]

        P --> Q[Roda o RUNSTATS para atulizar as estatisticas da REPROEST]

        Q --> R[Conta a quantidade de registros da REPROEST]

            R -- > 0 --> S[[Chama a SP_REPROEST]]
            S --> T[Retorno reprocessamento]
            T --> R

            R -- = 0 --> U[Atualiza o FLAGPROCESSADO para 'T']

        U --> A

        %% =========================
        %% Corres personalizadas
        %% =========================

        classDef inicio   fill:#F5A623,stroke:#C47D0E,color:#000
        classDef processo fill:#1A73E8,stroke:#1558B0,color:#fff
        classDef decisao  fill:#F5820A,stroke:#C45E00,color:#000
        classDef erro     fill:#D32F2F,stroke:#B71C1C,color:#fff
        classDef sucesso  fill:#2E7D32,stroke:#1B5E20,color:#fff

        class A inicio
        class C,D,E,G,H,I,K,L,M,N,O,P,Q,R,S,T,U processo
        class B,F,J decisao
        class Z sucesso
    ```
=== "SP_REPROEST"
    ```mermaid
    flowchart TD

        %% =========================
        %% FLUXO DA SP_REPROEST
        %% =========================
        
        A([🚀 Inicio da SP_REPROEST]) --> B[Declara a session tabela TMP_EST2]

        B --> C[Busca os 2000 primeiros registros da REPROEST e Insere na SESSION.TMP_EST2]

        C --> D[Entra no loop FOR para buscar o registros e fazer o reprocessamento]

        D --> E[Valida se o registro é do tipo balanço]

            E -- FLAGBALANCO = 'T' --> F{O registro possui IDLOTE?}

                F -- Sim --> G[/Chama a SP_REPROCESSA_BALANCO_LOTE\]
                F -- Não --> H[/Chama a SP_REPROCESSA_BALANCO_NORMAL\]
            
            E -- FLAGBALANCO = 'F' --> I([Chama a SP_INSERT_ESTOQUE])

        G --> J[Remove os registros reprocessados da REPROEST]
        H --> J
        I --> J

        J --> K[Dropa a session TMP_EST2]

        K --> L([Retorna para a SP_REPROEST_AUT])

        %% =========================
        %% Corres personalizadas
        %% =========================

        classDef inicio   fill:#F5A623,stroke:#C47D0E,color:#000
        classDef processo fill:#1A73E8,stroke:#1558B0,color:#fff
        classDef decisao  fill:#F5820A,stroke:#C45E00,color:#000
        classDef erro     fill:#D32F2F,stroke:#B71C1C,color:#fff
        classDef sucesso  fill:#2E7D32,stroke:#1B5E20,color:#fff

        class A inicio
        class C,D,E,G,H,I,K,L,M,N,O,P,Q,R,S,T,U processo
        class B,F,J decisao
        class L sucesso
    ```
=== "SP_REPROCESSA_BALANCO_NORMAL"
    ```mermaid
    flowchart TD

        %% =========================
        %% FLUXO DA SP_REPROCESSA_BALANCO_NORMAL
        %% =========================

        A(["🚀 Inicio da SP_REPROCESSA_BALANCO_NORMAL"]) -- Parametros --> B[/IN_IDEMPRESA, IN_IDPLANILHA, IN_IDPRODUTO, IN_IDSUBPRODUTO/]

        B --> C[Busca o valores do encerramento de balanco]

        C --> D[Remove a linha do encerramento atual]

        D --> E[Zera as variaveis]

        E --> F["LDE_QTDATUALESTOQUE = 0 
                 LDE_VALATUALESTOQUE = 0 
                 LDE_QTDSALDOESTOQUE = 0 
                 LDE_VALSALDOESTOQUE = 0"]

        F --> G[Busca os dados atuais de estoque para reelizar o calculo]
        G --> H[Refaz o calculo do valor de estoque para usar no ajuste]

        H --> I{O produto possui alguma divergencia de contatgem do saldo de estoque?}

        I -- Sim --> J[Busca nunsequencia para inserir na ESTOQUE_ANALITICO]
        
            J --> K[Busca o NUNSEQUENCIA para inserir na ESTOQUE_ANALITICO]

            K --> L[Calculo do custo medio do Balanço]

            L --> M{Quantiade contada}
                M -- "> 0" --> N["Custo medio = Valor contado / Quantidade contada"]
                M -- "= 0" --> O["Custo medio = 0"]
                O --> P
            
            N --> P[Seta a quantidade do ajuste]

            P --> Q["Ajuste = Quantidade contada - Quantidade atual"]

            Q --> R[Calculo para ajustar valor do estoque]

            R --> S{"Saldo de estoque"}
            S -- "< 0" --> T["Val ajuste = (Quantiade ajsute * Custo medio) - Valor Saldo de estoque"]
            S -- ">= 0" --> U["Val ajuste = ( ( Saldo estoque + quantidade ajuste ) * custo medio ) - valor saldo estoque"]

            U --> V[Insere o ajuste de balanço]
            T --> V

            V --> W{"Quantidade ajuste ou valor ajuste <> 0"}
            W -- "Sim" --> X[Insere ajuste na ESTOQUE_ANALITICO]
            W -- "Não" --> Y[Fecha o loop]
            X --> Y

        I -- "Não" --> Z
                        
        Y --> Z([Retorna para a SP_REPROEST])
        

        %% =========================
        %% Corres personalizadas
        %% =========================

        classDef inicio   fill:#F5A623,stroke:#C47D0E,color:#000
        classDef processo fill:#1A73E8,stroke:#1558B0,color:#fff
        classDef decisao  fill:#F5820A,stroke:#C45E00,color:#000
        classDef erro     fill:#D32F2F,stroke:#B71C1C,color:#fff
        classDef sucesso  fill:#2E7D32,stroke:#1B5E20,color:#fff

        class A inicio
        class B,C,D,E,F,G,H,J,K,L,N,O,P,Q,R,T,U,V,X,Y processo
        class I,M,S,W decisao
        class Z sucesso
    ```

=== "SP_REPROCESSA_BALANCO_LOTE"

    ```mermaid
    flowchart TD

        %% =========================
        %% FLUXO DA SP_REPROCESSA_BALANCO_LOTE
        %% =========================

        A(["🚀 Inicio da SP_REPROCESSA_BALANCO_LOTE"]) -- Parametros --> B[/IN_IDEMPRESA, IN_IDPLANILHA, IN_IDPRODUTO, IN_IDSUBPRODUTO/]

        B --> C[Contagem de itens que controlam lote]

        C --> D[Remove o ajuste anterior para todos os lotes do Item]

        D --> E[Busca os valores sinteticos do Item]

        E --> F[Busca os valores lançados para cada lote]

        F --> G[Zera as variaveis]

        G --> H["LDE_QTDATUALESTOQUE = 0 
                 LDE_QTDSALDOINICIAL = 0"]

        H --> I[Busca o saldo de estoque do lote]

        I --> J[Calculo do custo medio e valor do estoque para o lote]

            J --> K{Qunatiade atual do estoque princiapl > 0?}

                K -- Sim --> L["Custo medio = Valor atual do estoque princiapl / Quantidade atual do estoque princiapl"]
                L -- Sim --> M["Valor atual estoque = Custo medio * Quantidade atual do estoque"]

                K -- Não --> N["Custo medio = 0"]
                N --> O["Valor atual estoque = 0"]
            
            M --> P{"Quantidade contada <> Quantidade atual do estoque ou Valor contado <> Valor atual do estoque"}
            O --> P

                P -- Sim --> Q[Busca o numsequencia para inserir na ESTOQUE_ANALITICO]

                Q --> R[Calcula a quantidade e valor para o ajuste]

                R --> S{"Quantidade ajuste ou valor ajuste <> 0"}

                    S -- Sim --> T[Insere ajuste na ESTOQUE_ANALITICO]
                    S -- Não --> U[Fecha o loop]
            
        T --> V[Retorna para a SP_REPROEST]
    
    ```