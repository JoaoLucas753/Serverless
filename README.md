# Aluno: João Lucas Mendes dos Santos
# Rota Vital · Função serverless "Guia de transporte"

Atividade individual 1 · Infraestrutura de Software · ADS · CESAR School · 2026.2

Função serverless (AWS Lambda, Python 3.12) que recebe uma requisição do Rota Vital (hospital e bolsas de sangue) e devolve a guia de transporte. Ela roda localmente com o **AWS SAM CLI**, sem conta em provedor de nuvem.

## Caminho escolhido

**Caminho A: AWS SAM CLI local**, com Docker, `sam local invoke` (acionamento por evento) e `sam local start-api` (acionamento por HTTP).

## Estrutura do repositório

```
template.yaml          # API POST /guias e Timeout em Globals
hello_world/app.py     # handler da função
events/                # eventos no formato do API Gateway (aws-proxy)
evidencias/            # prints dos comandos e das saídas
```

## Como rodar

Pré-requisitos: Docker Desktop aberto, AWS SAM CLI (`sam --version`) e Python 3.12. No Windows, use Git Bash ou WSL.

```bash
sam build

# 1) Invocação com evento (rodada duas vezes)
sam local invoke -e events/pedido.json

# 2) Gatilho HTTP (três chamadas)
sam local start-api
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'

# 3) Reaproveitamento do ambiente
sam local start-api --warm-containers LAZY
# repetir os mesmos curl

# 4) Limite de tempo (Timeout = 3 no template.yaml)
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
# depois: Timeout = 10 no template.yaml, sam build e repetir
```

A pasta `events/` tem também `pedido-demorar.json` (dorme 5 s) e `pedido-invalido.json` (sem `requisicaoId`, retorna 400).

## O que a função faz

- Lê o `body` do evento e exige o campo `requisicaoId`; sem ele, responde **400**.
- Monta a guia (`G-` + `requisicaoId`), com hospital, bolsas e o valor de `invocacaoNesteAmbiente`, e responde **201**.
- Mantém uma variável global `contador`, que serve para observar se o ambiente de execução foi reaproveitado.
- Se o pedido traz `demorar: true`, dorme 5 s para estourar o Timeout de 3 s.

## Experimentos

| Experimento | O que observei |
|---|---|
| `sam local invoke` (2 vezes) | A primeira invocação foi mais lenta porque inclui a criação do ambiente. No `invoke`, o SAM sobe um container por execução, e a segunda ficou mais rápida porque a imagem já estava em cache. O contador veio 1 nas duas. |
| `start-api` sem `--warm-containers` | Cada requisição subiu um container novo, e `invocacaoNesteAmbiente` veio 1 nas três chamadas. |
| `start-api --warm-containers LAZY` | O container foi criado na primeira chamada e reaproveitado, e `invocacaoNesteAmbiente` cresceu: 1, 2, 3. |
| `demorar=true`, Timeout 3 | A execução foi interrompida por timeout após 3 s e a API respondeu com erro (502). |
| `demorar=true`, Timeout 10 | A função terminou após cerca de 5 s e respondeu **201**. |

Os prints de cada comando estão em `evidencias/`.

## Respostas

**1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?**

Na primeira chamada ainda não existe ambiente de execução pronto, e a fase de Init cria esse ambiente. Ela prepara o ambiente (no SAM local, sobe o container), carrega o código da função, inicia o runtime do Python e executa o código que está fora do handler, como os `import` e o `contador = 0`. Só depois o handler roda. Isso é o cold start. Quando o ambiente já existe e é reaproveitado, o Init não se repete e só o handler executa, por isso a chamada fica mais rápida.

**2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global?**

Sem `--warm-containers`, o campo veio sempre 1, porque cada chamada caiu num ambiente novo e o contador recomeçou. Com `--warm-containers LAZY`, o mesmo ambiente foi reaproveitado e o valor cresceu: 1, 2, 3. Isso mostra que uma variável global só sobrevive se o ambiente for reaproveitado, e o Lambda não garante isso. Ele pode descartar o ambiente a qualquer momento, e chamadas simultâneas usam ambientes diferentes, cada um com seu próprio contador. Por isso a variável global serve, no máximo, como cache de otimização. Estado que precisa ser confiável deve ficar fora da função, em banco de dados ou cache externo.

**3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?**

A função dorme 5 s, mas o template limita a execução a 3 s, então o Lambda interrompeu a função por timeout e a API devolveu erro. Depois de aumentar o Timeout para 10 s e rodar `sam build` de novo, a função terminou e respondeu 201. O tempo máximo de uma execução no Lambda é de **15 minutos (900 s)**. Um trabalho mais longo deve ser dividido em etapas menores (por exemplo, com fila ou Step Functions) ou rodar em um serviço pensado para processos longos, como um container.

**4. Quanto custa um milhão de execuções com 128 MB e 100 ms (preços de us-east-1)?**

- Requisições: 1.000.000 × US$ 0,20 / 1.000.000 = **US$ 0,20**
- Memória em GB: 128 MB / 1024 = 0,125 GB
- Computação por execução: 0,125 GB × 0,1 s = 0,0125 GB-s
- Computação total: 1.000.000 × 0,0125 = 12.500 GB-s
- Custo de computação: 12.500 × US$ 0,0000166667 ≈ **US$ 0,2083**
- **Total ≈ US$ 0,41** por milhão de execuções, sem considerar a faixa gratuita.

**5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?**

Faz sentido em tarefas curtas, sem estado e acionadas por evento ou HTTP, como gerar a guia de transporte quando uma requisição chega ou disparar uma notificação para o hospital. A carga é irregular, a execução é rápida e se paga apenas pelo uso. Não faz sentido em partes que precisam manter estado ou conexão por muito tempo, como o rastreamento em tempo real das bolsas durante o transporte, que pede um serviço sempre ativo, ou em processamentos que passem dos 15 minutos.

## Vídeo

Gravação de tela de até 3 minutos mostrando a invocação com evento, a chamada por HTTP e o erro de tempo limite. O link foi entregue no Google Classroom junto com o link deste repositório.

## Uso de IA

- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** montar o roteiro de execução, organizar a estrutura do repositório e do README, e redigir um rascunho das respostas conceituais.
- **Como validei:** executei o handler com os eventos de teste (201, 400 e a espera de 5 s com `demorar`), refiz a conta de custo passo a passo e conferi os conceitos de Init, reaproveitamento de ambiente e limite de tempo na documentação do AWS Lambda e do SAM CLI.
rverless
