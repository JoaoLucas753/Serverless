## Aluno: João Lucas Mendes dos Santos
# Rota Vital · Função serverless "Guia de transporte"

Atividade individual 1 · Infraestrutura de Software · ADS · CESAR School · 2026.2

Função serverless (AWS Lambda, Python 3.12) que recebe uma requisição do Rota Vital (hospital e bolsas de sangue) e devolve a guia de transporte. Ela roda localmente com o **AWS SAM CLI**, sem conta em provedor de nuvem.

## Caminho escolhido

**Caminho A: AWS SAM CLI local**, executado no WSL (Ubuntu) com Docker, usando `sam local invoke` (acionamento por evento) e `sam local start-api` (acionamento por HTTP).

## Estrutura do repositório

```
template.yaml          # API POST /guias e Timeout em Globals (valor final: 10 s)
hello_world/app.py     # handler da função
events/pedido.json     # evento no formato do API Gateway (aws-proxy)
evidencias/            # prints dos comandos e das saídas
```

## Como rodar

Pré-requisitos: Docker Desktop aberto, AWS SAM CLI (`sam --version`) e Python 3.12. No Windows, use o WSL ou o Git Bash.

```bash
sam build

# 1) Invocação com evento (rodada duas vezes)
sam local invoke -e events/pedido.json

# 2) Gatilho HTTP
sam local start-api
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'

# 3) Reaproveitamento do ambiente (comparação)
sam local start-api --warm-containers LAZY

# 4) Limite de tempo: com Timeout 3 no template.yaml
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
# depois: Timeout 10 no template.yaml, sam build e repetir
```

## O que a função faz

- Lê o `body` do evento e exige o campo `requisicaoId`; sem ele, responde **400**.
- Monta a guia (`G-` + `requisicaoId`), com hospital, bolsas e o valor de `invocacaoNesteAmbiente`, e responde **201**.
- Mantém uma variável global `contador`, para observar se o ambiente de execução foi reaproveitado.
- Se o pedido traz `demorar: true`, dorme 5 s, o que passa do Timeout de 3 s do template original.

## Resultados dos experimentos

**Invocação com evento (`sam local invoke -e events/pedido.json`)**

| Invocação | Init Duration | Duration | Billed Duration | `invocacaoNesteAmbiente` |
|---|---|---|---|---|
| 1ª (`evidencias/01-invocacao-evento.png`) | 0,08 ms | 334,07 ms | 335 ms | 1 |
| 2ª (`evidencias/02-segunda-invocacao.png`) | 0,08 ms | 414,59 ms | 415 ms | 1 |

Nas duas, a função usou 128 MB de memória e respondeu **201** com a guia `G-R-001`.

**Chamada por HTTP (`sam local start-api` + `curl`)**

- Uma chamada isolada (`evidencias/03-api-http.png`): resposta 201 com a guia `G-R-002` e `invocacaoNesteAmbiente` = 1.
- Três chamadas seguidas (`evidencias/04-contador-sem-warm.png`): `G-R-008`, `G-R-009` e `G-R-010`, com `invocacaoNesteAmbiente` = **1, 2 e 3**, ou seja, o mesmo ambiente atendeu as três requisições e o contador foi acumulando.

**Limite de tempo**

- Com Timeout 3 s, a chamada com `demorar: true` (5 s de espera) passa do limite e a execução é interrompida por timeout.
- Depois de mudar o Timeout para 10 s no `template.yaml` (`evidencias/template-timeout-10s.png`) e rodar `sam build`, o limite cobre os 5 s da função e a chamada termina com **201**.

## Respostas

**1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?**

Nas minhas medições, a primeira invocação (334,07 ms) **não** foi mais lenta que a segunda (414,59 ms), e o Init Duration foi o mesmo nas duas (0,08 ms). A explicação é que o `sam local invoke` sobe um container novo a cada execução, então as duas invocações foram "frias" e a diferença entre elas é só variação normal de tempo do ambiente local. O Init Duration de 0,08 ms mostra apenas a parte de carregar o código da função, e não o tempo de criar o container.

No Lambda de verdade, a diferença aparece assim: na primeira chamada não existe ambiente de execução pronto, e a fase de Init cria o ambiente, carrega o código, inicia o runtime do Python e executa o código que está fora do handler (os `import` e o `contador = 0`). Só depois o handler roda. Isso é o cold start. Quando o ambiente é reaproveitado, o Init não se repete e só o handler executa, por isso as chamadas seguintes costumam ser mais rápidas.

**2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global?**

Na chamada isolada o campo veio 1. Nas três chamadas seguidas ao `start-api` o campo cresceu: 1, 2, 3, o que mostra que o mesmo ambiente foi reaproveitado entre as requisições e a variável global continuou viva. O valor só volta a 1 quando um ambiente novo é criado, como acontece a cada `sam local invoke`. A opção `--warm-containers` serve para manter os containers vivos entre as chamadas, aumentando a chance de reaproveitamento.

O que isso diz sobre estado em variável global: ela só sobrevive se o ambiente for reaproveitado, e o Lambda não garante isso. Ele pode descartar o ambiente a qualquer momento, e chamadas simultâneas usam ambientes diferentes, cada um com seu próprio contador. Por isso a variável global serve no máximo como cache de otimização. Estado que precisa ser confiável deve ficar fora da função, em banco de dados ou cache externo.

**3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?**

A função dorme 5 s, mas o template limitava a execução a 3 s, então o Lambda interrompeu a função por timeout. Depois de aumentar o Timeout para 10 s e rodar `sam build` de novo, a função conseguiu terminar e respondeu 201. O tempo máximo de uma execução no Lambda é de **15 minutos (900 s)**. Um trabalho mais longo deve ser dividido em etapas menores (por exemplo, com fila ou Step Functions) ou rodar em um serviço pensado para processos longos, como um container.

**4. Quanto custa um milhão de execuções com 128 MB e 100 ms (preços de us-east-1)?**

- Requisições: 1.000.000 × US$ 0,20 / 1.000.000 = **US$ 0,20**
- Memória em GB: 128 MB / 1024 = 0,125 GB
- Computação por execução: 0,125 GB × 0,1 s = 0,0125 GB-s
- Computação total: 1.000.000 × 0,0125 = 12.500 GB-s
- Custo de computação: 12.500 × US$ 0,0000166667 ≈ **US$ 0,2083**
- **Total ≈ US$ 0,41** por milhão de execuções, sem considerar a faixa gratuita.

**5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?**

Faz sentido em tarefas curtas, sem estado e acionadas por evento ou HTTP, como gerar a guia de transporte quando uma requisição chega ou disparar uma notificação para o hospital. A carga é irregular, a execução é rápida e se paga apenas pelo uso. Não faz sentido em partes que precisam manter estado ou conexão por muito tempo, como o rastreamento em tempo real das bolsas durante o transporte, que pede um serviço sempre ativo, ou em processamentos que passem dos 15 minutos.


## Uso de IA

- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** montar o roteiro de execução, organizar a estrutura do repositório e do README, e redigir um rascunho das respostas conceituais.
- **Como validei:** executei os comandos no WSL com o SAM CLI e o Docker, registrei as saídas em `evidencias/`, atualizei o README com os valores que realmente medi (inclusive onde diferiram do esperado), refiz a conta de custo passo a passo e conferi os conceitos de Init, reaproveitamento de ambiente e limite de tempo na documentação do AWS Lambda e do SAM CLI.
