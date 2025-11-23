# 🥋 FIUAPI - Sistema de Gestão de Eventos Esportivos

API desenvolvida como projeto final para a disciplina de **Banco de Dados**. O sistema gerencia atletas, centros de treinamento (CTs), eventos e competições, integrando conceitos avançados de banco de dados relacional com uma arquitetura moderna de backend.

## 🚀 Tecnologias Utilizadas

* **Linguagem:** C# (.NET 8)
* **Banco de Dados:** PostgreSQL
* **ORM:** Entity Framework Core (Code First)
* **Driver:** Npgsql
* **Documentação:** Swagger / OpenAPI

## 🏛️ Arquitetura e Padrões

O projeto foi construído seguindo boas práticas de desenvolvimento e design patterns:

* **Repository Pattern:** Implementação genérica (`Repository<T>`) para operações CRUD padrão e repositórios específicos (`AtletaRepository`, `EventoRepository`) para consultas complexas.
* **DTOs (Data Transfer Objects):** Separação entre as entidades do banco e os dados trafegados na API.
* **Service Layer:** Camada de serviço (`EventoService`, `CTService`) para regras de negócio e controle de transações (Unit of Work).
* **Dependency Injection:** Injeção de dependência para desacoplamento de componentes.

## 💾 Destaques do Banco de Dados (PL/pgSQL)

Como foco da disciplina, foram implementadas diversas rotinas diretamente no banco de dados para performance e integridade:

### ⚡ Procedures
* `registrar_presenca`: Realiza a inscrição de um atleta em um evento, validando se ele já está inscrito para evitar duplicidade.

### 🔍 Functions (Relatórios e Consultas)
* `calendario_eventos`: Filtra eventos por Dia, Semana ou Mês, aplicando tratamento de Fuso Horário (UTC) para garantir precisão nas datas.
* `historico_atleta`: Retorna o histórico completo de competições passadas de um atleta.
* `avaliacoes_eventos`: Relatório analítico que calcula a média de notas e concatena múltiplos comentários de avaliações em uma linha.
* `exibir_detalhes_encontro`: Gera um "Raio-X" do evento usando múltiplos `LEFT JOINS` (Evento, Local, CT, Atletas), exibindo dados mesmo se parciais.
* `filtrar_encontros_por_modalidade`: Lista eventos de uma modalidade específica, utilizando `DISTINCT` para evitar linhas duplicadas.
* `exibir_participantes_evento`: Gera a lista de chamada com os nomes dos atletas inscritos em um evento específico.
* `deletar_avaliacoes_equipe`: Função auxiliar executada pela Trigger para limpar avaliações órfãs.

### 🔫 Triggers
* `trg_delete_avaliacoes_equipe`: Gatilho `BEFORE DELETE` que limpa automaticamente as avaliações de uma equipe antes que ela seja excluída, evitando erros de chave estrangeira (*Foreign Key Violation*).

## ⚙️ Como Rodar o Projeto

### Pré-requisitos
* .NET SDK 8.0
* PostgreSQL instalado e rodando

### 1. Configurar Conexão
No arquivo `appsettings.json`, ajuste a string de conexão com suas credenciais do Postgres:

```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Port=5432;Database=FIUDb;User Id=postgres;Password=SUA_SENHA;"
}
```
### 2. Criar o Banco (Migrations)
Execute o comando abaixo no terminal para que o Entity Framework cire o banco e as tabelas:

```csharp
dotnet ef database update
```
(Nota: As procedures e triggers devem ser criadas via script SQL no banco, utilizando um SGBD de sua preferencia).

### 3. Criar Scripts SQL (Obrigatório)
O Entity Framework cria as tabelas, mas as Procedures, Functions e Triggers devem ser criadas manualmente.

Copie os scripts abaixo e execute no seu gerenciador de banco de dados (PgAdmin, DataGrip, DBeaver, etc)

<details>
  <summary>
    <strong>📄 CLIQUE AQUI PARA EXPANDIR OS SCRIPTS SQL COMPLETOS</strong>
  </summary>


### 3.1 filtrar_encontros_por_modalidade:
```SQL
DROP FUNCTION IF EXISTS filtrar_encontros_por_modalidade(INT);
DROP FUNCTION IF EXISTS filtrar_encontros_por_modalidade(BIGINT);

CREATE OR REPLACE FUNCTION filtrar_encontros_por_modalidade(p_mod_id BIGINT)
RETURNS TABLE (
    evento_id BIGINT,
    evento_nome VARCHAR,
    evento_data TIMESTAMPTZ,
    modalidade VARCHAR,
    cidade VARCHAR,
    bairro VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT DISTINCT -- DISTINCT evita duplicatas se tiver vários atletas da mesma modalidade
        e.pk_eventoid AS evento_id,
        e.nome AS evento_nome,
        e.data AS evento_data,
        m.nome AS modalidade,
        l.cidade,
        l.bairro
    FROM evento e
    INNER JOIN participacao_evento pe ON pe.fk_eventoid = e.pk_eventoid
    INNER JOIN atleta a ON a.pk_atletaid = pe.fk_atletaid
    INNER JOIN faixa f ON f.pk_faixaid = a.fk_faixaid
    INNER JOIN modalidade m ON m.pk_modalidadeid = f.fk_modalidadeid
    INNER JOIN local l ON l.fk_eventoid = e.pk_eventoid
    WHERE m.pk_modalidadeid = p_mod_id
    ORDER BY e.data;
END;
$$;
```
### 3.2 exibir_detalhes_encontro:
```SQL
DROP FUNCTION IF EXISTS exibir_detalhes_encontro(BIGINT);

CREATE OR REPLACE FUNCTION exibir_detalhes_encontro(p_eve_id BIGINT)
RETURNS TABLE (
    evento_nome VARCHAR,
    evento_data TIMESTAMPTZ,
    logradouro VARCHAR,
    bairro VARCHAR,
    cidade VARCHAR,
    ct_nome VARCHAR,
    atleta_nome VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.nome AS evento_nome,
        e.data AS evento_data,
        COALESCE(l.logradouro, 'Sem Local') AS logradouro,
        l.bairro,
        l.cidade,
        COALESCE(ct.nome, 'Sem CT vinculado') AS ct_nome,
        COALESCE(a.nome, 'Nenhum inscrito') AS atleta_nome
    FROM "Evento" e
    LEFT JOIN local l ON l.fk_eventoid = e.pk_eventoid
    LEFT JOIN ct_evento ce ON ce.fk_eventoid = e.pk_eventoid
    LEFT JOIN ct ct ON ct.pk_ctid = ce.fk_ctid
    LEFT JOIN participacao_evento pe ON pe.fk_eventoid = e.pk_eventoid
    LEFT JOIN atleta a ON a.pk_atletaid = pe.fk_atletaid
    
    WHERE e.pk_eventoid = p_eve_id
    ORDER BY ct.nome, a.nome;
END;
$$;
```
### 3.3 deletar_avaliacoes_equipe:
```SQL
CREATE OR REPLACE FUNCTION deletar_avaliacoes_equipe()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    DELETE FROM "avaliacao"
    WHERE fk_equipeid = OLD.pk_equipeid;

    RETURN OLD;
END;
$$;
```
### 3.4 trg_delete_avaliacoes_equipe:

```SQL
DROP TRIGGER IF EXISTS trg_delete_avaliacoes_equipe ON equipe;

CREATE TRIGGER trg_delete_avaliacoes_equipe
BEFORE DELETE ON equipe
FOR EACH ROW
EXECUTE FUNCTION deletar_avaliacoes_equipe();
```
### 3.5 exibir_participantes_evento:
```SQL
DROP FUNCTION IF EXISTS exibir_participantes_evento(INT);
DROP FUNCTION IF EXISTS exibir_participantes_evento(BIGINT);

CREATE OR REPLACE FUNCTION exibir_participantes_evento(p_eve_id BIGINT)
RETURNS TABLE (
    evento VARCHAR,
    atleta_id BIGINT,
    atleta_nome VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.nome AS evento,
        a.pk_atletaid AS atleta_id,
        a.nome AS atleta_nome
    FROM participacao_evento p
    JOIN atleta a ON a.pk_atletaid = p.fk_atletaid
    JOIN evento e ON e.pk_eventoid = p.fk_eventoid
    WHERE p.fk_eventoid = p_eve_id
    ORDER BY a.nome;
END;
$$;
```
### 3.6 registrar_presenca:
```SQL
CREATE OR REPLACE PROCEDURE registrar_presenca(
    p_atl_id BIGINT,
    p_eve_id BIGINT
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 
        FROM participacao_evento
        WHERE fk_atletaid = p_atl_id
          AND fk_eventoid = p_eve_id
    ) THEN
        -- Inserindo com valor padrão na colocação para satisfazer o banco
        INSERT INTO participacao_evento (fk_atletaid, fk_eventoid, colocacao)
        VALUES (p_atl_id, p_eve_id, 'Inscrito');
    END IF;
END;
$$;
```
### 3.7 avaliacoes_eventos:
```SQL
DROP FUNCTION IF EXISTS avaliacoes_eventos(BIGINT, DATE, BIGINT);

CREATE OR REPLACE FUNCTION avaliacoes_eventos(
    p_mod_id BIGINT,
    p_data DATE,
    p_local_id BIGINT
)
RETURNS TABLE (
    evento_id BIGINT,
    evento_nome VARCHAR,
    evento_data TIMESTAMPTZ,
    logradouro VARCHAR,
    bairro VARCHAR,
    cidade VARCHAR,
    uf VARCHAR,
    modalidade VARCHAR,
    media_avaliacao DECIMAL,
    comentarios TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.pk_eventoid,
        e.nome,
        e.data,

        l.logradouro,
        l.bairro,
        l.cidade,
        l.uf,

        m.nome AS modalidade,

        COALESCE(AVG(av.nota), 0) AS media_avaliacao,

        STRING_AGG(av.descricao, ' || ') AS comentarios

    FROM evento e
    LEFT JOIN local l ON e.pk_eventoid = l.fk_eventoid
    LEFT JOIN participacao_evento pe ON e.pk_eventoid = pe.fk_eventoid
    LEFT JOIN atleta a ON pe.fk_atletaid = a.pk_atletaid
    LEFT JOIN faixa f ON a.fk_faixaid = f.pk_faixaid
    LEFT JOIN modalidade m ON f.fk_modalidadeid = m.pk_modalidadeid
    LEFT JOIN ct ct ON a.fk_ctid = ct.pk_ctid
    LEFT JOIN avaliacao av ON ct.fk_equipeid = av.fk_equipeid

    WHERE (p_mod_id IS NULL OR m.pk_modalidadeid = p_mod_id)
      AND (p_data IS NULL OR (e.data AT TIME ZONE 'UTC')::DATE = p_data)
      AND (p_local_id IS NULL OR l.pk_localid = p_local_id)

    GROUP BY 
        e.pk_eventoid,
        e.nome,
        e.data,
        l.logradouro,
        l.bairro,
        l.cidade,
        l.uf,
        m.nome
    
    ORDER BY e.data;
END;
$$;
```
### 3.8 historico_atleta:
```SQL
CREATE OR REPLACE FUNCTION historico_atleta(p_atl_id BIGINT)
RETURNS TABLE (
    atleta_nome VARCHAR,
    evento_id BIGINT,
    evento_nome VARCHAR,
    evento_data TIMESTAMPTZ,
    logradouro VARCHAR,
    bairro VARCHAR,
    cidade VARCHAR,
    uf VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        a.nome AS atleta_nome,      
        e.pk_eventoid AS evento_id,
        e.nome AS evento_nome,
        e.data AS evento_data,
        l.logradouro,
        l.bairro,
        l.cidade,
        l.uf
    FROM participacao_evento pe
    INNER JOIN evento e      ON pe.fk_eventoid = e.pk_eventoid
    LEFT JOIN local l        ON e.pk_eventoid = l.fk_eventoid
    INNER JOIN atleta a      ON pe.fk_atletaid = a.pk_atletaid
    WHERE pe.fk_atletaid = p_atl_id
    ORDER BY e.data DESC;
END;
$$;
```
### 3.9 calendario_eventos:
```SQL
DROP FUNCTION IF EXISTS calendario_eventos(VARCHAR, DATE);

CREATE OR REPLACE FUNCTION calendario_eventos(
    p_view_type VARCHAR,
    p_date DATE
)
RETURNS TABLE (
    id BIGINT,
    nome VARCHAR,
    data TIMESTAMPTZ,
    logradouro VARCHAR,
    bairro VARCHAR,
    cidade VARCHAR,
    uf VARCHAR
) 
LANGUAGE plpgsql
AS $$
DECLARE
    semana_inicio DATE;
    semana_fim DATE;
BEGIN
    semana_inicio := date_trunc('week', p_date)::DATE;
    semana_fim := (semana_inicio + INTERVAL '6 days')::DATE;

    IF p_view_type = 'mes' THEN
        RETURN QUERY
        SELECT 
            e.pk_eventoid AS id,
            e.nome,
            e.data,
            l.logradouro,
            l.bairro,
            l.cidade,
            l.uf
        FROM evento e
        LEFT JOIN local l ON e.pk_eventoid = l.fk_eventoid
        -- Força a leitura em UTC (Data crua)
        WHERE EXTRACT(YEAR FROM (e.data AT TIME ZONE 'UTC')) = EXTRACT(YEAR FROM p_date)
          AND EXTRACT(MONTH FROM (e.data AT TIME ZONE 'UTC')) = EXTRACT(MONTH FROM p_date)
        ORDER BY e.data;

    ELSIF p_view_type = 'semana' THEN
        RETURN QUERY
        SELECT 
            e.pk_eventoid AS id,
            e.nome,
            e.data,
            l.logradouro,
            l.bairro,
            l.cidade,
            l.uf
        FROM evento e
        LEFT JOIN local l ON e.pk_eventoid = l.fk_eventoid
        -- Força a leitura em UTC
        WHERE (e.data AT TIME ZONE 'UTC')::DATE >= semana_inicio 
          AND (e.data AT TIME ZONE 'UTC')::DATE <= semana_fim
        ORDER BY e.data;

    ELSIF p_view_type = 'dia' THEN
        RETURN QUERY
        SELECT 
            e.pk_eventoid AS id,
            e.nome,
            e.data,
            l.logradouro,
            l.bairro,
            l.cidade,
            l.uf
        FROM evento e
        LEFT JOIN local l ON e.pk_eventoid = l.fk_eventoid
        WHERE (e.data AT TIME ZONE 'UTC')::DATE = p_date
        ORDER BY e.data;

    ELSE
        RAISE EXCEPTION 'Tipo de visualização inválido. Use: mes, semana ou dia.';
    END IF;
END;
$$;
```
</details>

### 4. Rodar a API

```csharp
dotnet run
```
Acesse a documentação interavida em: `http://localhost:7069/swagger` (ou a porta indicada no seu terminal)

## 🧪 Exemplo de Uso (Endpoints)

### 1. Criar CT completo (Transaction)

POST `/api/v1/ct/cadastro-completo`

```json
{
  "nome": "Dojo Central",
  "equipeId": 1,
  "treinadorId": 1,
  "logradouro": "Av. Paulista",
  "numero": 1000,
  "bairro": "Bela Vista",
  "cidade": "São Paulo",
  "uf": "SP",
  "cep": "01310-100"
}
```

### 2. Agendar Evento (Transaction)

POST `/api/v1/evento/agendar`

```json
{
  "nomeBase": "Treinão Geral",
  "dataHora": "2025-11-25T14:00:00Z",
  "modalidadeId": 1,
  "logradouro": "Rua das Flores",
  "bairro": "Centro",
  "cidade": "São Paulo",
  "uf": "SP",
  "cep": "01001-000"
}
```
### 3. Consultar Calendário (Function)

GET `/api/v1/evento/calendario?viewType=mes&data=2025-11-22`

