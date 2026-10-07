# Chatbot Telegram de Temperatura — n8n + OpenWeather + Redis + Gemini

Chatbot desenvolvido em **n8n** para receber uma cidade brasileira pelo Telegram, consultar a temperatura atual na OpenWeather e responder em português.

Além dos requisitos obrigatórios do desafio, esta versão inclui:

- cache em **Redis por 30 minutos** para reduzir chamadas repetidas à OpenWeather;
- **Google Gemini 2.5 Flash** opcional para melhorar a redação da resposta;
- fallback determinístico obrigatório, permitindo executar o workflow sem Gemini;
- validação de cidade, UF, país, HTTP status e estrutura das respostas;
- consulta alternativa por coordenadas quando houver risco de cidade homônima;
- nenhuma chave ou token embutido no JSON exportado.

## Arquivos

- `workflow-chatbot-telegram.json`: workflow principal para importação no n8n.
- `README.md`: documentação do projeto.
- `docker-compose.yml`: ambiente opcional com n8n + Redis.
- `.env.example`: exemplo de variáveis sem segredos reais.

## Formato das mensagens

Envie ao bot:

```text
Cidade,UF,BR
```

Exemplos:

```text
São Paulo,SP,BR
Jaraguá do Sul,SC,BR
Belo Horizonte,MG,BR
```

Também é aceito o formato abreviado:

```text
Cidade,UF
```

Exemplo de resposta:

```text
🌤️ A temperatura em Jaraguá do Sul (SC) é de 24°C.
```

Quando a cidade não for localizada:

```text
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## 1. Importando o workflow

1. Abra o n8n.
2. Crie um novo workflow.
3. Use **Import from File**.
4. Selecione `workflow-chatbot-telegram.json`.
5. Configure as credenciais descritas abaixo.
6. Execute manualmente para testar.
7. Depois dos testes, publique/ative o workflow.

## 2. Variáveis esperadas

As variáveis abaixo devem ficar fora do repositório:

```env
OPENWEATHER_API_KEY=
TELEGRAM_BOT_TOKEN=
REDIS_CACHE_ENABLED=false
GEMINI_ENABLED=false
```

### OPENWEATHER_API_KEY

Obrigatória.

A chave é usada nos nodes HTTP Request por meio de:

```text
{{ $env.OPENWEATHER_API_KEY }}
```

Nunca coloque a chave diretamente no node ou no JSON exportado.

### TELEGRAM_BOT_TOKEN

Obrigatória para criar a credencial **Telegram API** no n8n.

1. Crie o bot usando o `@BotFather`.
2. Guarde o token em local seguro.
3. No n8n, crie uma credencial do tipo **Telegram API**.
4. Informe nessa credencial o valor mantido como `TELEGRAM_BOT_TOKEN` no seu ambiente/gerenciador de segredos.
5. Selecione a mesma credencial nos nodes:
   - `Telegram Trigger`
   - `Telegram - Temperatura`
   - `Telegram - Erro ou ajuda`

O token não deve ser salvo no repositório.

## 3. OpenWeather

O endpoint obrigatório utilizado é:

```text
https://api.openweathermap.org/data/2.5/weather
```

Parâmetros enviados:

- `q`: recebe a variável interna `queue`;
- `units=metric`;
- `lang=pt_br`;
- `appid={{ $env.OPENWEATHER_API_KEY }}`.

A variável `queue` é criada no início do workflow a partir da mensagem normalizada do Telegram.

Antes da consulta de clima, o workflow usa a geocodificação da própria OpenWeather para validar se a cidade realmente pertence à UF informada. Isso evita retornar uma cidade homônima de outro estado.

## 4. Cache Redis — opcional

O Redis é uma melhoria adicional e não é necessário para atender ao fluxo obrigatório.

Para ativar:

```env
REDIS_CACHE_ENABLED=true
```

Depois:

1. crie uma credencial do tipo **Redis** no n8n;
2. selecione essa credencial nos nodes:
   - `Redis - Buscar cache`
   - `Redis - Salvar 30 min`.

### Estratégia de cache

A chave usa cidade + UF:

```text
weather:br:<uf>:<cidade-normalizada>
```

Exemplo:

```text
weather:br:sc:jaragua-do-sul
```

O TTL é:

```text
1800 segundos = 30 minutos
```

Fluxo:

```text
Telegram
  ↓
Validação
  ↓
Redis GET
  ├─ HIT  → usa temperatura armazenada
  └─ MISS → OpenWeather → Redis SET com TTL 1800
```

O cache não é feito apenas por estado porque cidades diferentes da mesma UF podem ter temperaturas diferentes.

Se `REDIS_CACHE_ENABLED=false`, os nodes Redis são ignorados e o chatbot continua funcionando normalmente.

## 5. Google Gemini 2.5 Flash — opcional

O Gemini é utilizado **somente para melhorar a redação da resposta**. A temperatura sempre vem da OpenWeather ou do cache Redis.

Modelo configurado:

```text
models/gemini-2.5-flash
```

Configurações:

```text
Temperature: 0.2
Top P: 0.9
Top K: 40
Max output tokens: 120
Thinking budget: 0
```

Para ativar:

```env
GEMINI_ENABLED=true
```

Depois:

1. abra o node `Google Gemini - Melhorar resposta`;
2. crie/selecione uma credencial **Google Gemini(PaLM) API**;
3. informe sua chave Gemini somente dentro da credencial do n8n;
4. não coloque a chave no workflow ou no GitHub.

### Formato exigido do Gemini

O modelo recebe instrução para responder somente:

```json
{"message":"mensagem final","ok":true}
```

### Fallback obrigatório

Antes de chamar o Gemini, o node `Preparar mensagem para IA` já gera uma resposta determinística.

O node `Validar resposta Gemini` verifica a saída da IA. Se ocorrer qualquer uma destas situações:

- erro na chamada;
- JSON inválido;
- `ok` diferente de `true`;
- temperatura alterada;
- UF alterada;
- resposta excessivamente longa;

então o fluxo ignora a IA e envia a mensagem determinística.

Assim, para executar sem custo e sem credencial Gemini, basta manter:

```env
GEMINI_ENABLED=false
```

## 6. Tratamento de erros

O workflow trata:

- mensagem fora do padrão;
- UF inexistente;
- país diferente de Brasil;
- cidade não encontrada;
- cidade homônima em UF diferente;
- resposta inválida da OpenWeather;
- HTTP 401, 403, 404, 429 e erros 5xx;
- timeout/indisponibilidade da API;
- cache Redis ausente ou inválido;
- falha/saída inválida do Gemini.

Falhas de infraestrutura não expõem tokens, headers ou detalhes internos ao usuário.

## 7. Testes recomendados

Teste pelo menos estas três cidades:

```text
São Paulo,SP,BR
Jaraguá do Sul,SC,BR
Belo Horizonte,MG,BR
```

Depois teste uma cidade inválida:

```text
CidadeQueNaoExiste,SC,BR
```

Resultado esperado:

```text
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

### Testando o Redis

Com `REDIS_CACHE_ENABLED=true`:

1. consulte uma cidade;
2. consulte a mesma cidade novamente antes de 30 minutos;
3. a segunda execução deve seguir pelo ramo `Cache encontrado? = true` e não chamar a OpenWeather.

### Testando o fallback do Gemini

1. execute com `GEMINI_ENABLED=false` e confirme que a resposta funciona;
2. configure a credencial Gemini;
3. altere para `GEMINI_ENABLED=true`;
4. execute novamente;
5. opcionalmente desative/remova a credencial e volte `GEMINI_ENABLED=false` para comprovar que o fluxo obrigatório é independente da IA.

## 8. Telegram Trigger e HTTPS

O Telegram precisa alcançar o webhook do n8n pela internet usando HTTPS.

Em instalação local ou rede corporativa, configure uma URL pública válida, por exemplo por proxy reverso/túnel, e defina `WEBHOOK_URL` no n8n quando necessário.

Se o host não puder ser resolvido externamente, o Telegram Trigger pode retornar erro semelhante a `bad webhook: Failed to resolve host`.

## 9. Docker opcional

O arquivo `docker-compose.yml` sobe:

- n8n;
- Redis.

Copie `.env.example` para `.env`, preencha seus próprios valores e execute:

```bash
docker compose up -d
```

Nunca envie o arquivo `.env` real para o GitHub.

## 10. Checklist de entrega

- [ ] Workflow importado sem erro.
- [ ] Telegram Trigger configurado.
- [ ] `OPENWEATHER_API_KEY` configurada no ambiente.
- [ ] Credencial Telegram configurada.
- [ ] Testado com pelo menos 3 cidades válidas.
- [ ] Testado com cidade inexistente.
- [ ] Redis testado ou mantido desabilitado.
- [ ] Gemini testado ou mantido desabilitado.
- [ ] Fallback sem Gemini validado.
- [ ] Nenhum token real presente no JSON.
- [ ] Nenhum segredo real presente no README.
- [ ] Repositório GitHub público.

## Segurança

Não versionar:

```text
.env
*.secret
arquivos com tokens
exports de credenciais do n8n
```

O workflow entregue referencia variáveis/credenciais, mas não contém segredos reais.