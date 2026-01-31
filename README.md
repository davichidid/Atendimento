# Monitoramento de Atendimento em Real Time

Esta Query consolida indicadores de atendimento em tempo real, fornecendo uma visão detalhada de cada interação. Essa visão foi desenvolvida para dar suporte ao time de Control Desk e Planejamento Estratégico, permitindo a identificação imediata de desvios operacionais, instabilidades de telefonia e a validação de manobras estratégicas no discador para mensurar a eficácia da estratégia.

Informações que são trazidas quando o código é executado:

1 - Hora de início do atendimento (momento da conexão e vínculo com o discador)
2 - Segmento a qual aquele atendimento pertence (campanha discador ou 0800)
3 - Telefone 
4 - Classificação do telefone (se é o telefone tem marcação de Excelente ou ainda como Indefinido)
5 - Id do contrato acionado
6 - Valor referente ao contrato atendido 
7 - Operador responsável pelo o atendimento
8 - Tabulação lançada pelo operador (finalização)
9 - Origem desligamento (se o desconect se deu pelo operador ou pelo cliente)
10 - Tempo falado na ligação (em minutos e segundos)
11 - Hora a qual o atendimento foi executado

```

SELECT DISTINCT
    CHAM.INICIO_CHAMADA,
    CHAM.CARTEIRA_EXEMPLO,
    CHAM.NUMERO_CONTATO,
    CHAM.USUARIO_OPERADOR,
    CHAM.STATUS_FINALIZACAO,
    CHAM.MOTIVO_DESLIGAMENTO,
    CHAM.CAMPANHA,
    CHAM.OPERADORA,
    CHAM.ID_CONTRATO,

    -- Formatação de horários
    CONVERT(varchar(8), CHAM.INICIO_ATENDIMENTO, 108) AS INICIO_ATENDIMENTO,
    CONVERT(varchar(8), CHAM.FIM_CHAMADA, 108)        AS FIM_CHAMADA,

    -- Extração da hora com 2 dígitos
    RIGHT('0' + CAST(DATEPART(HOUR, CHAM.INICIO_ATENDIMENTO) AS VARCHAR), 2) AS HORA_ATENDIMENTO,

    -- Cálculo de tempo falado formatado em HH:MM:SS
    CONVERT(varchar(8), DATEADD(second, DATEDIFF(second, CHAM.INICIO_ATENDIMENTO, CHAM.FIM_CHAMADA), 0), 108) AS TEMPO_FALADO,

    SCR.CLASSIFICACAO_TELEFONE AS CLASSIFICACAO_CONTATO,
    RISCO.VALOR_CONTRATO       
    
FROM
    DB_DISCADOR.SCHEMA_OPERACIONAL.VW_LIGACOES_OUTBOUND CHAM WITH (NOLOCK)

LEFT JOIN
    DB_ANALITICO.SCHEMA_SCORE.VW_SCORE_CONTATOS SCR WITH (NOLOCK)
        ON CHAM.ID_CONTATO = SCR.ID_CONTATO

OUTER APPLY (
    SELECT TOP 1 -- SELECT DISTINCT em subqueries de relação 1:1 pode ser lento; TOP 1 é mais seguro se houver duplicidade
        F.VALOR_DIVIDA AS VALOR_CONTRATO
    FROM
        [SRV_REMOTO].DB_FINANCEIRO.SCHEMA_SIMULACAO.VW_CALCULO_RISCO F WITH (NOLOCK)
    WHERE
        F.ID_CONTRATO = CHAM.ID_CONTRATO
) RISCO

WHERE
    CHAM.CAMPANHA IN ('CAMPANHA_A', 'CAMPANHA_B', 'CAMPANHA_C')
    AND CONVERT(varchar(8), CHAM.INICIO_ATENDIMENTO, 108) >= '08:00:00'
    AND CHAM.STATUS_FINALIZACAO IS NOT NULL

ORDER BY
    CHAM.INICIO_CHAMADA DESC;
