# AlertaCidade

> Global Solution FIAP 2026.2 · CitizenTech · Engenharia de Software (2º ano, fevereiro)

## Ficha CitizenTech

| Campo | Definição |
|---|---|
| **Nome** | AlertaCidade |
| **Cidade e bairro** | `<PREENCHER: cidade / bairro escolhidos pela equipe>` |
| **Cidadão beneficiado** | Moradores, trabalhadores em deslocamento e agentes comunitários de regiões com histórico de alagamento |
| **Problema em uma frase** | Falta informação acessível, localizada e no momento certo sobre a subida da água em pontos sujeitos a alagamento |
| **Como funciona** | Medições (simuladas) de nível de água e chuva → API REST Java → Oracle → classificação NORMAL / ATENÇÃO / CRÍTICO → alertas e dashboard |
| **O que cada parte faz** | Java: API, CRUD e regras · Oracle: persistência · Python: análise estatística e rotas em grafo · Web: dashboard · AR/VR: demonstração imersiva · Packet Tracer: rede do local |
| **Repositório** | `<PREENCHER: link do GitHub>` |
| **Deploy da API** | `<PREENCHER: link do Render>` |
| **Indicador de impacto (protótipo)** | Tempo entre a leitura e a exibição do estado no painel; acerto da classificação nos cenários de teste |
| **ODS** | 11 (Cidades e comunidades sustentáveis), 13 (Ação contra a mudança global do clima), 9 (Indústria, inovação e infraestrutura) |

> **Aviso:** protótipo acadêmico. Os limites de classificação (cotas de atenção e crítica) e todos os dados de exemplo são **fictícios e de demonstração**. O AlertaCidade não é um serviço oficial de alerta e não deve orientar decisões reais de emergência.

### Equipe 

| Nome completo              | RM     |
|----------------------------|--------|
| Eduardo Antonio Delarissia | 563468 |


## Estrutura do repositório

```
README.md
render.yaml                 Blueprint de deploy no Render
database/
  ddl.sql                   Criação das tabelas e constraints (Oracle)
  dml.sql                   Dados de exemplo (>= 5 linhas por tabela)
  dql.sql                   Consultas exigidas (INNER JOIN, GROUP BY, LEFT JOIN)
backend-java/               API RESTful Spring Boot (Domain Driven Design - Java)
  src/main/java/br/com/fiap/alertacidade/
    model/                  Entidades e enums do domínio (regra de classificação)
    dto/                    Corpos de requisição (com Bean Validation) e respostas
    dao/                    Acesso a dados via JDBC (JdbcTemplate) - SQL Oracle
    bo/                     Regras de negócio
    controller/             Endpoints REST (GET, POST, PUT, DELETE)
    exception/              Exceções de negócio + handler global (JSON padronizado)
    config/                 CORS
  src/main/resources/db/h2/ Cópia do DDL/DML para o perfil dev (H2 em modo Oracle)
  requests.http             Todas as requisições de exemplo
  scripts/cenario-cheia.sh  Cenário de teste de subida da água
  Dockerfile                Imagem usada no Render
```

## Modelo de dados

```
T_AC_PONTO_MONITORAMENTO 1 ──< N T_AC_SENSOR 1 ──< N T_AC_MEDICAO
T_AC_PONTO_MONITORAMENTO 1 ──< N T_AC_ALERTA >── 0..1 T_AC_MEDICAO  (alerta manual não tem medição)
```

| Tabela | Colunas principais | Constraints |
|---|---|---|
| T_AC_PONTO_MONITORAMENTO | id_ponto, nm_ponto, nm_bairro, nm_cidade, nr_latitude, nr_longitude, nr_cota_atencao_cm, nr_cota_critica_cm, st_ativo, dt_cadastro | PK; UNIQUE nome; CHECK lat/lon; CHECK crítica > atenção; CHECK st_ativo S/N |
| T_AC_SENSOR | id_sensor, id_ponto, cd_sensor, tp_sensor, ds_unidade, st_sensor, dt_instalacao | PK; FK ponto; UNIQUE código; CHECK tipo, unidade e status |
| T_AC_MEDICAO | id_medicao, id_sensor, nr_valor, dt_hora_medicao, ds_origem | PK; FK sensor; CHECK valor >= 0; CHECK origem |
| T_AC_ALERTA | id_alerta, id_ponto, id_medicao, nv_alerta, ds_mensagem, dt_hora_alerta, st_alerta, dt_encerramento | PK; FK ponto; FK medição ON DELETE SET NULL; CHECK nível, status e coerência do encerramento |

## Regras de negócio

- **Classificação** (sensores de nível de água): valor ≥ cota crítica → `CRITICO`; valor ≥ cota de atenção → `ATENCAO`; senão `NORMAL`. As cotas são definidas por ponto.
- **Geração de alertas**: uma leitura em ATENÇÃO/CRÍTICO abre um alerta, a menos que já exista alerta aberto de nível igual ou maior (evita duplicação). Ao escalar para CRÍTICO, o alerta automático de ATENÇÃO é encerrado.
- **Encerramento automático**: uma leitura NORMAL encerra os alertas automáticos abertos do ponto. Alertas manuais só são encerrados manualmente (`PUT` com `status: ENCERRADO`).
- Leituras de pluviômetro (mm) são registradas sem classificação.
- Só sensores `ATIVO` registram leituras; data/hora não pode estar no futuro.
- A unidade do sensor é definida pelo tipo (`NIVEL_AGUA` → cm, `PLUVIOMETRO` → mm) e o tipo não muda depois que há medições.
- Não é possível excluir ponto com sensores/alertas, nem sensor com medições (preserva o histórico): desative-os via `PUT`.

## Endpoints

Documentação interativa: `/swagger-ui.html` (local ou no Render).

| Método | Rota | Descrição | Sucesso |
|---|---|---|---|
| GET | `/api/pontos?bairro=` | Lista pontos (filtro opcional por bairro) | 200 |
| GET | `/api/pontos/{id}` | Busca ponto | 200 |
| GET | `/api/pontos/status` | Situação atual de todos os pontos (dashboard) | 200 |
| GET | `/api/pontos/{id}/status` | Situação atual de um ponto | 200 |
| POST | `/api/pontos` | Cadastra ponto | 201 |
| PUT | `/api/pontos/{id}` | Atualiza ponto (inclusive `ativo`) | 200 |
| DELETE | `/api/pontos/{id}` | Exclui ponto sem dependências | 204 |
| GET | `/api/sensores?pontoId=` | Lista sensores | 200 |
| GET | `/api/sensores/{id}` | Busca sensor | 200 |
| POST | `/api/sensores` | Cadastra sensor | 201 |
| PUT | `/api/sensores/{id}` | Atualiza sensor (ex.: status MANUTENCAO) | 200 |
| DELETE | `/api/sensores/{id}` | Exclui sensor sem medições | 204 |
| GET | `/api/medicoes?sensorId=&limite=50` | Lista leituras (mais recentes primeiro) | 200 |
| GET | `/api/medicoes/{id}` | Busca leitura | 200 |
| POST | `/api/medicoes` | Registra leitura, classifica e gera/encerra alertas | 201 |
| PUT | `/api/medicoes/{id}` | Corrige valor/data/origem (não reprocessa alertas) | 200 |
| DELETE | `/api/medicoes/{id}` | Exclui leitura | 204 |
| GET | `/api/alertas?status=&pontoId=` | Lista alertas | 200 |
| GET | `/api/alertas/{id}` | Busca alerta | 200 |
| POST | `/api/alertas` | Cria alerta manual | 201 |
| PUT | `/api/alertas/{id}` | Atualiza, encerra ou reabre alerta | 200 |
| DELETE | `/api/alertas/{id}` | Exclui alerta | 204 |

**Erros** (corpo padronizado `ErroResponse`: `timestamp, status, erro, mensagem, caminho, detalhes`):
400 dados inválidos · 404 não encontrado · 409 conflito (duplicidade ou dependências) · 415 sem JSON · 422 regra de negócio violada · 503 banco indisponível · 500 erro inesperado.

Exemplo de `POST /api/medicoes` → 201:

```json
{
  "medicao": { "id": 16, "sensorId": 3, "valor": 104.5, "dataHora": "2026-11-09T14:30:00", "origem": "SIMULADOR" },
  "classificacao": "CRITICO",
  "alertaGerado": { "id": 6, "pontoId": 2, "medicaoId": 16, "nivel": "CRITICO", "status": "ABERTO", "...": "..." },
  "alertasEncerrados": 1
}
```

## Como executar

**Ferramentas e versões:** Java JDK 21 · Maven 3.9+ (ou o Maven embutido do IntelliJ IDEA) · Oracle Database (servidor FIAP) · SQL Developer / Oracle Data Modeler · Git.

### 1. Rodar localmente sem Oracle (perfil `dev`, padrão)

```bash
cd backend-java
mvn spring-boot:run
```

Sobe em `http://localhost:8080` com H2 em memória (modo Oracle) e os mesmos dados de exemplo do `dml.sql`. Swagger: `http://localhost:8080/swagger-ui.html`. Console do banco: `http://localhost:8080/h2-console` (JDBC URL `jdbc:h2:mem:alertacidade`, usuário `sa`).

No IntelliJ: abrir a pasta `backend-java` como projeto Maven e rodar `AlertaCidadeApplication`.

### 2. Rodar com o Oracle da FIAP (perfil `prod`)

1. No SQL Developer, conectado com seu usuário FIAP, execute `database/ddl.sql` e depois `database/dml.sql` (na primeira execução, os `DROP TABLE` do início dão erro ORA-00942: pode ignorar).
2. Exporte as variáveis e suba a API:

```bash
export SPRING_PROFILES_ACTIVE=prod
export DB_URL=jdbc:oracle:thin:@oracle.fiap.com.br:1521:ORCL
export DB_USER=rmXXXXX
export DB_PASSWORD=sua_senha
mvn spring-boot:run
```

(No Windows/PowerShell: `$env:DB_USER="rmXXXXX"` etc. No IntelliJ: *Run Configuration → Environment variables*.)

### 3. Testes

```bash
mvn test
```

Testes de integração sobem a API em H2 e cobrem CRUD, o cenário de cheia e os códigos de erro.

### 4. Cenário de demonstração (subida da água)

Com a API rodando:

```bash
./scripts/cenario-cheia.sh                       # local
./scripts/cenario-cheia.sh https://SEU-APP.onrender.com
```

Cria um ponto e um sensor, envia leituras de 20 → 104 → 15 cm e mostra a classificação e o estado do ponto mudando (NORMAL → ATENÇÃO → CRÍTICO → NORMAL).

### 5. Deploy no Render

1. Faça push do repositório no GitHub.
2. No Render: **New → Web Service** → conecte o repositório → **Runtime: Docker** → **Root Directory: `backend-java`** → plano Free. (Alternativa: **New → Blueprint**, que usa o `render.yaml`.)
3. Em **Environment**, adicione `SPRING_PROFILES_ACTIVE=prod`, `DB_URL`, `DB_USER`, `DB_PASSWORD`.
4. Após o deploy, teste `https://SEU-APP.onrender.com/api/pontos` e o `/swagger-ui.html`.

> O plano gratuito "dorme" após inatividade: a primeira requisição pode levar ~1 minuto. Acorde o serviço antes de apresentar.
> **Commits após o prazo da GS zeram a parte técnica de DDD-Java.**
