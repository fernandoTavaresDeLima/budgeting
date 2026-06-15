# Budgeting — Gerenciador Financeiro com IA e Voz

## O que o projeto faz

O **Budgeting** é uma aplicação de gerenciamento de transações financeiras que combina uma API REST tradicional com comandos de voz processados por Inteligência Artificial.

O usuário pode:

- **Criar transações financeiras** (despesas categorizadas como Supermercado, Farmácia ou Automóvel) via API REST ou por meio de um áudio gravado
- **Consultar transações** por categoria
- **Interagir por voz**: enviar um áudio dizendo algo como *"Registre uma compra de R$ 150 no supermercado"* e receber uma resposta também em áudio

O fluxo de voz funciona da seguinte forma:

```
Áudio do usuário
    ↓
Transcrição via OpenAI Whisper (speech-to-text, em português)
    ↓
Interpretação pelo GPT-4o-mini com ferramentas registradas (@Tool)
    ↓
Execução do caso de uso correto (criar ou listar transações)
    ↓
Resposta sintetizada em áudio via OpenAI TTS (voz "nova")
    ↓
Arquivo MP3 retornado ao cliente
```

---

## Como executar a aplicação

### Pré-requisitos

| Ferramenta | Versão mínima |
|---|---|
| Java | 25 |
| Docker | qualquer versão recente |
| Chave de API OpenAI | com acesso a `gpt-4o-mini`, `whisper-1` e `gpt-4o-mini-tts` |

### Passo a passo

**1. Clone o repositório**

```bash
git clone <url-do-repositorio>
cd budgeting
```

**2. Configure a chave da OpenAI**

No Windows (PowerShell):
```powershell
$env:OPENAI_API_KEY = "sk-..."
```

No Linux/macOS:
```bash
export OPENAI_API_KEY="sk-..."
```

**3. Suba o banco de dados MySQL via Docker**

```bash
docker compose up -d
```

Isso inicia um MySQL 9.6 na porta `3307`, com banco `transaction`, usuário `app` e senha `app`.

**4. Execute a aplicação**

```bash
./gradlew bootRun
```

A aplicação sobe na porta **8081**.

---

## Tecnologias utilizadas

| Camada | Tecnologia |
|---|---|
| Linguagem | Java 25 |
| Framework principal | Spring Boot 4.0.5 |
| IA / LLM | Spring AI 2.0.0-M4 + OpenAI (GPT-4o-mini) |
| Speech-to-text | OpenAI Whisper-1 |
| Text-to-speech | OpenAI TTS (`gpt-4o-mini-tts`, voz `nova`) |
| Banco de dados | MySQL 9.6 (via Docker Compose) |
| ORM | Spring Data JPA + Hibernate |
| Redução de boilerplate | Lombok |
| Build | Gradle (wrapper incluído) |
| Testes | JUnit 5 + AssertJ |

---

## Como testar o fluxo principal

### 1. Criar uma transação via REST

```bash
curl -X POST http://localhost:8081/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Compra no mercado",
    "category": "GROCERIES",
    "amount": 15000
  }'
```

> O campo `amount` está em **centavos** (ex: `15000` = R$ 150,00).

Resposta esperada:
```json
{
  "id": "...",
  "description": "Compra no mercado",
  "category": "GROCERIES",
  "amount": 15000
}
```

### 2. Listar transações por categoria

```bash
curl http://localhost:8081/transactions/GROCERIES
```

Categorias disponíveis: `GROCERIES`, `PHARMA`, `AUTO`.

### 3. Criar uma transação por voz (fluxo principal de IA)

Grave um áudio `.m4a` ou `.mp3` dizendo algo como:

> *"Registre uma compra de cento e cinquenta reais na farmácia."*

Em seguida, envie o arquivo:

```bash
curl -X POST http://localhost:8081/transactions/ai \
  -F "file=@seu-audio.m4a" \
  --output resposta.mp3
```

A aplicação vai:
1. Transcrever o áudio
2. Interpretar o comando com o GPT-4o-mini
3. Persistir a transação automaticamente
4. Retornar um arquivo `resposta.mp3` com a confirmação em voz

### 4. Executar os testes automatizados

```bash
./gradlew test
```

Os testes de integração com a API da OpenAI são executados somente se a variável `OPENAI_API_KEY` estiver definida. Se a variável não estiver presente, esses testes são ignorados automaticamente.

Os arquivos de áudio utilizados nos testes estão em `src/test/resources/audio/` (6 gravações em `.m4a`).

---

## O que é possível aprender com esse projeto

### Arquitetura em camadas (DDD simplificado)
O projeto organiza o código em três camadas bem separadas — **domínio**, **aplicação** e **infraestrutura** — o que demonstra como isolar regras de negócio de detalhes técnicos como banco de dados e HTTP.

### Spring AI e Tool Calling
Os casos de uso (`PersistTransactionUseCase` e `ListTransactionsByCategoryUseCase`) são anotados com `@Tool` do Spring AI. Isso permite que o modelo de linguagem identifique automaticamente qual operação executar a partir de linguagem natural, sem lógica de roteamento manual.

### Integração com múltiplas APIs de IA da OpenAI
O projeto usa três modelos diferentes da OpenAI em conjunto:
- **Whisper** para transcrição de áudio
- **GPT-4o-mini** para raciocínio e decisão
- **TTS** para síntese de voz

Isso ilustra como compor serviços de IA em um único fluxo coeso.

### Tipagem forte no domínio
O uso de `TransactionId` (um record Java que envolve um `UUID`) em vez de usar `UUID` diretamente previne erros de troca de parâmetros e deixa o código mais expressivo.

### Valores monetários sem ponto flutuante
O campo `amount` armazena valores em centavos como `long`, evitando os problemas clássicos de precisão de `double` e `float` em cálculos financeiros.

### Testes de integração condicionais
O uso de `@EnabledIfEnvironmentVariable` nos testes demonstra como criar testes que dependem de recursos externos sem quebrar o build em ambientes que não possuem as credenciais configuradas.

### Docker Compose integrado ao Spring Boot
A dependência `spring-boot-docker-compose` permite que o Spring Boot inicie o banco de dados automaticamente durante o desenvolvimento, sem precisar subir o Docker manualmente.