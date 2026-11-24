# Atividade individual 1: Serverless

Infraestrutura de Software · ADS · CESAR School · 2026.2
Autor: João Felipe (github.com/jonthefool)

## Contexto

Função serverless acionada por evento e por HTTP que recebe uma requisição de bolsas de sangue feita por um hospital e devolve a guia de transporte. O fluxo é inspirado em requisição → entrega do HemoFlow (projeto integrador).

Entrada (JSON no body):

```json
{"requisicaoId": "R-001", "hospital": "H-Recife", "bolsas": ["O-NEG-0142"]}
```

Saída (201):

```json
{"guia": "G-R-001", "hospital": "H-Recife", "bolsas": ["O-NEG-0142"], "invocacaoNesteAmbiente": 1}
```

Sem `requisicaoId` a função responde 400.

## Caminho escolhido

Caminho A: AWS SAM CLI rodando localmente (Docker), Python 3.12, pacote zip. Sem conta na AWS e sem publicação na nuvem.

## Vídeo

https://drive.google.com/drive/folders/1qzc6uPwd0rshWtVcONDQROE745Mw80vp?usp=drive_link

## Estrutura

- `template.yaml`: recurso da função, API `POST /guias` e `Timeout` em Globals
- `hello_world/app.py`: handler da função
- `events/pedido.json`: evento gerado com `sam local generate-event`
- `r002.json` e `r003.json`: corpos usados nos `curl` pelo PowerShell
- `evidencias/`: prints dos comandos e das saídas

## Como rodar

Pré-requisitos: Docker Desktop, AWS SAM CLI (`sam --version`) e Python 3.12. Os comandos abaixo estão em Git Bash. No PowerShell, usar `curl.exe -d "@r002.json"` em vez de `-d '{...}'`.

```bash
# 1. gerar o evento e construir
sam local generate-event apigateway aws-proxy --method POST --path guias \
  --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' \
  > events/pedido.json
sam build

# 2. invocar com evento (rodar duas vezes)
sam local invoke -e events/pedido.json

# 3. acionar por HTTP
sam local start-api
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'

# 4. limite de tempo
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

## Resultados observados

Invocação com evento (`sam local invoke`):

| Invocação | Duration | Init Duration |
|---|---|---|
| 1ª | [PREENCHER] ms | [PREENCHER] ms |
| 2ª | [PREENCHER] ms | [PREENCHER] ms |

Chamadas HTTP (`sam local start-api`), campo `invocacaoNesteAmbiente` nas três chamadas com `R-002`:

| Modo | Chamada 1 | Chamada 2 | Chamada 3 |
|---|---|---|---|
| `sam local start-api` | 1 | 2 | 3 |
| `sam local start-api --warm-containers LAZY` | não executado por falta de tempo | | |

Limite de tempo (`demorar: true`, a função dorme 5 s):

| Timeout | Resultado |
|---|---|
| 3 s | `Function 'HelloWorldFunction' timed out after 3 seconds`, Duration 3000 ms, status 502 no servidor e `{"message":"Internal server error"}` no `curl` |
| 10 s | não executado por falta de tempo |

## Respostas

**1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?**

[CONFIRME COM OS SEUS TEMPOS DA TABELA E REESCREVA COM SUAS PALAVRAS]
A primeira execução precisa criar o ambiente de execução, o chamado cold start. Na fase de Init o ambiente é preparado: o código da função é carregado, o runtime do Python é iniciado e o código que fica fora do handler roda uma vez (os imports e a linha `contador = 0`). Só depois disso o handler executa, na fase de Invoke. Quando o ambiente já existe e é reaproveitado, o Init não se repete e a chamada é mais rápida.

**2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global da função?**

Na minha máquina, o `sam local start-api` já reaproveitou o container e o campo subiu 1, 2, 3 nas três chamadas, mesmo sem `--warm-containers`. Não executei a variação com `--warm-containers LAZY` por falta de tempo. A variável global sobrevive enquanto o ambiente for reaproveitado, mas isso não é garantido: no Lambda real o ambiente pode ser descartado ou criado em paralelo, cada um com o seu contador. Estado que precisa ser confiável deve ficar fora da função, em banco de dados, cache ou armazenamento de objetos. A variável global serve no máximo como cache oportunista.

**3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?**

Com `Timeout: 3` no template, a função dormiu 5 s e foi interrompida aos 3 s: o log mostrou `Function 'HelloWorldFunction' timed out after 3 seconds`, o servidor local respondeu 502 e o `curl` recebeu `Internal server error`. Não executei a repetição com `Timeout: 10`. O tempo máximo de uma execução no Lambda é 15 minutos (900 s). Trabalho mais longo deve ser dividido em etapas menores (por exemplo, orquestradas por Step Functions ou encadeadas por fila) ou rodar em um serviço sem esse limite, como contêineres (ECS/Fargate) ou AWS Batch.

**4. Custo de um milhão de execuções com 128 MB e 100 ms (preços us-east-1 da aula)**

- Requisições: 1.000.000 ÷ 1.000.000 × US$ 0,20 = US$ 0,20
- Memória em GB: 128 MB ÷ 1024 = 0,125 GB
- Duração em segundos: 100 ms = 0,1 s
- GB-s por execução: 0,125 × 0,1 = 0,0125
- GB-s em 1.000.000 de execuções: 0,0125 × 1.000.000 = 12.500 GB-s
- Custo de computação: 12.500 × US$ 0,0000166667 ≈ US$ 0,2083
- Total: US$ 0,20 + US$ 0,2083 ≈ **US$ 0,41**

Esse valor não considera a faixa gratuita (1 milhão de requisições e 400 mil GB-s por mês), que cobriria esse volume por inteiro.

**5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?**

[REESCREVA COM SUAS PALAVRAS]
Faz sentido onde a tarefa é curta, sem estado e disparada por evento ou em picos: gerar a guia de transporte quando uma requisição é aprovada, enviar alerta quando uma leitura de temperatura sai do limite da cadeia fria, avisar sobre bolsas perto do vencimento em uma rotina agendada e mandar a confirmação de um agendamento de doação. Não faz sentido no back-end principal, que mantém sessão (a autenticação usa `HttpSession`) e as regras de negócio do estoque, nem no monitoramento contínuo em tempo real e em cálculos longos, como as rotas, que dependem de estado em memória ou de execução prolongada.

## Uso de IA

[EDITE PARA REFLETIR O QUE VOCÊ DE FATO FEZ]
- Ferramenta: Claude (Anthropic).
- Finalidade: apoio na organização do roteiro de execução, na correção de erros de execução (aspas do PowerShell no evento de teste) e em um rascunho da estrutura do README e das respostas.
- Como validei: executei todos os comandos no meu computador, conferi as saídas reais, ajustei as respostas ao que observei e refiz a conta de custo à mão.
