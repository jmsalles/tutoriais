# Projeto 1: Como simular serviços AWS localmente com o Floci

> Série: laboratórios práticos para estudos e certificações  
> Nível: iniciante/intermediário  
> Tecnologias: Linux, Docker, Docker Compose, AWS CLI, Amazon S3, Amazon DynamoDB e Amazon SQS  
> Tipo de ambiente: laboratório local, sem necessidade de uma conta AWS

## Certificações AWS relacionadas

Este tutorial é especialmente útil como laboratório complementar para as seguintes certificações:

| Certificação | Código | Relação com o conteúdo |
| --- | --- | --- |
| AWS Certified Cloud Practitioner | `CLF-C02` | Ajuda a reconhecer a finalidade do Amazon S3, do Amazon DynamoDB e do Amazon SQS, além de reforçar conceitos básicos de serviços gerenciados da AWS. |
| AWS Certified Solutions Architect - Associate | `SAA-C03` | Permite praticar armazenamento de objetos, banco de dados NoSQL e desacoplamento de aplicações por meio de filas. |
| AWS Certified Developer - Associate | `DVA-C02` | Reforça o uso da AWS CLI e as operações de criação, consulta e integração entre S3, DynamoDB e SQS. |
| AWS Certified SysOps Administrator - Associate | `SOA-C03` | Contribui para a prática de provisionamento, validação, diagnóstico e remoção controlada de recursos por linha de comando. |
| AWS Certified DevOps Engineer - Professional | `DOP-C02` | Serve como base complementar para testes locais, automação e integração de serviços antes da adoção de pipelines e infraestrutura como código. |
| AWS Certified Solutions Architect - Professional | `SAP-C02` | Pode ser utilizado como revisão prática dos serviços que formam soluções distribuídas, embora a prova exija arquiteturas e cenários muito mais avançados. |

> O laboratório ajuda a fixar conceitos e operações presentes nessas certificações, mas não substitui o estudo do guia oficial de cada exame nem a prática em uma conta AWS real.

## Introdução

Estudar somente a teoria costuma não ser suficiente para consolidar os conteúdos cobrados em certificações. Sempre que possível, procuro transformar cada assunto em um laboratório no qual seja possível criar, testar, errar, corrigir, validar e repetir.

Neste primeiro projeto da série, utilizaremos o [Floci](https://github.com/floci-io/floci), um emulador local e open source de serviços AWS. Ele implementa os protocolos utilizados pela AWS e permite que ferramentas conhecidas, como AWS CLI e SDKs oficiais, enviem requisições para um ambiente executado localmente.

Em vez de a AWS CLI enviar as requisições para um endpoint real da AWS, direcionaremos os comandos para:

```text
http://localhost:4566
```

Neste laboratório, criaremos um ambiente persistente com Docker Compose e realizaremos exercícios detalhados com:

- Amazon S3, para armazenamento de objetos;
- Amazon DynamoDB, para persistência de dados NoSQL;
- Amazon SQS, para troca assíncrona de mensagens;
- persistência local, para comprovar que os recursos sobrevivem à recriação do container.

> O Floci é destinado a aprendizado, desenvolvimento e testes. Ele não substitui a validação final de uma solução em uma conta AWS real, principalmente quando o objetivo envolve segurança, desempenho, limites, disponibilidade, cobrança ou integrações não completamente emuladas.

## Visão geral da arquitetura do laboratório

O laboratório terá os seguintes componentes:

| Componente | Função |
| --- | --- |
| Docker Engine | Executar o container do Floci |
| Docker Compose | Declarar e controlar o ambiente |
| Floci | Receber e processar as chamadas compatíveis com os serviços AWS |
| AWS CLI | Criar, consultar, atualizar e remover os recursos simulados |
| Volume `floci-data` | Manter os dados persistentes fora da camada gravável do container |
| Porta `4566` | Expor localmente o endpoint utilizado pelos serviços |

O fluxo de uma requisição será:

1. você executa um comando na AWS CLI;
2. a opção `--endpoint-url` direciona a chamada para `http://localhost:4566`;
3. o Floci identifica o serviço e a operação solicitada;
4. o serviço local processa a operação;
5. os dados persistentes são gravados no volume Docker;
6. a resposta é devolvida no mesmo formato utilizado pela API correspondente da AWS.

## O que você aprenderá

Ao concluir o laboratório, você terá praticado:

- criação de um ambiente local reproduzível com Docker Compose;
- validação do arquivo Compose antes da execução;
- análise do estado do container e do endpoint de saúde;
- configuração temporária e segura da AWS CLI;
- diferenciação entre endpoint local e endpoint real da AWS;
- criação de bucket, objetos, metadados, cópias e downloads no S3;
- validação da integridade de um objeto com comparação de arquivos e hash;
- criação de tabela, chave de partição, item, consulta, varredura e atualização no DynamoDB;
- criação de fila, envio, inspeção, recebimento e exclusão de mensagem no SQS;
- entendimento do `ReceiptHandle` e do tempo de invisibilidade de uma mensagem;
- integração manual entre S3, DynamoDB e SQS;
- comprovação da persistência após reinicialização e recriação do container;
- remoção controlada dos recursos e do laboratório.

## Antes de começar: local não significa idêntico à nuvem

Os comandos deste tutorial utilizam a mesma AWS CLI usada contra a AWS real, mas o destino das requisições será diferente.

Na AWS real, o cliente normalmente acessaria endpoints públicos como:

```text
https://s3.us-east-1.amazonaws.com
https://dynamodb.us-east-1.amazonaws.com
https://sqs.us-east-1.amazonaws.com
```

No laboratório, todos os serviços serão acessados por:

```text
http://localhost:4566
```

Por isso, a presença do parâmetro abaixo é uma medida importante de segurança:

```text
--endpoint-url "$AWS_ENDPOINT_URL"
```

Ele deixa explícito que o comando deve ser enviado ao emulador local. Durante os exercícios, não remova esse parâmetro.

## Pré-requisitos

Para acompanhar todo o tutorial, tenha disponível:

- Docker Engine ou Docker Desktop;
- plugin Docker Compose v2;
- AWS CLI v2;
- `curl`;
- `jq`, utilizado para formatar e extrair campos das respostas JSON;
- `sha256sum` e `cmp`, normalmente fornecidos pelo pacote `coreutils`;
- terminal Bash em Linux, macOS ou WSL.

Não é necessário:

- possuir uma conta AWS;
- criar usuário IAM real;
- informar cartão de crédito;
- utilizar chaves de acesso reais.

### Validar as ferramentas

Execute:

```bash
docker --version
docker compose version
aws --version
curl --version
jq --version
sha256sum --version | head -n 1
```

Exemplos de saídas válidas:

```text
Docker version 28.x.x, build ...
Docker Compose version v2.x.x
aws-cli/2.x.x Python/3.x.x Linux/...
curl 8.x.x (...)
jq-1.7.x
sha256sum (GNU coreutils) 9.x
```

As versões exatas podem ser diferentes. O importante é que todos os comandos sejam encontrados e terminem sem erro.

### Confirmar que o Docker está operacional

```bash
docker info >/dev/null && echo "OK: Docker está operacional."
```

Saída esperada:

```text
OK: Docker está operacional.
```

Se aparecer uma mensagem de permissão, valide se seu usuário pode acessar o Docker. Em uma estação Linux, isso normalmente envolve associação ao grupo `docker` e um novo login. Em ambiente corporativo, siga a política definida pela equipe responsável.

## 1. Criar o diretório do laboratório

Crie um diretório exclusivo para o projeto:

```bash
mkdir -p laboratorio-floci
cd laboratorio-floci
```

Confirme o diretório atual:

```bash
pwd
```

Exemplo:

```text
/home/seu-usuario/laboratorio-floci
```

No início, o diretório estará vazio:

```bash
find . -maxdepth 2 -type f -print
```

Nenhuma saída é esperada.

## 2. Criar e entender o arquivo `compose.yaml`

Crie o arquivo `compose.yaml` com o conteúdo abaixo:

```yaml
services:
  floci:
    image: floci/floci:latest
    container_name: floci
    ports:
      - "4566:4566"
    environment:
      FLOCI_DEFAULT_REGION: us-east-1
      FLOCI_DEFAULT_ACCOUNT_ID: "000000000000"
      FLOCI_STORAGE_MODE: hybrid
      FLOCI_STORAGE_PERSISTENT_PATH: /app/data
    volumes:
      - floci-data:/app/data
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:4566/_floci/health"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 5s
    restart: unless-stopped

volumes:
  floci-data:
```

### 2.1 Entender cada configuração

#### Imagem

```yaml
image: floci/floci:latest
```

Solicita a versão estável mais recente publicada no canal `latest`.

Para um laboratório pessoal, essa opção facilita o primeiro contato. Para cenários que exigem execução rigorosamente reproduzível, fixe uma versão previamente testada, por exemplo:

```yaml
image: floci/floci:1.5.33
```

Ao fixar uma versão, confirme antes se a tag existe no repositório de imagens.

#### Nome do container

```yaml
container_name: floci
```

Define um nome previsível. Isso facilita comandos como:

```bash
docker inspect floci
docker logs floci
```

#### Porta

```yaml
ports:
  - "4566:4566"
```

O primeiro valor representa a porta no host. O segundo representa a porta no container.

Assim:

```text
localhost:4566 -> container floci:4566
```

#### Região e conta simuladas

```yaml
FLOCI_DEFAULT_REGION: us-east-1
FLOCI_DEFAULT_ACCOUNT_ID: "000000000000"
```

Esses valores tornam o laboratório previsível. A conta `000000000000` é local e simulada; ela não corresponde a uma conta AWS real.

#### Armazenamento

```yaml
FLOCI_STORAGE_MODE: hybrid
FLOCI_STORAGE_PERSISTENT_PATH: /app/data
```

No modo `hybrid`, as leituras ocorrem em memória e o estado é descarregado para disco. É um perfil adequado para desenvolvimento local porque combina persistência com boa velocidade.

#### Volume

```yaml
volumes:
  - floci-data:/app/data
```

O volume mantém os dados fora da camada interna do container. Com isso, um novo container pode acessar o mesmo estado persistido.

#### Verificação de saúde

```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://localhost:4566/_floci/health"]
```

O Docker consultará periodicamente o endpoint nativo de saúde do Floci. O estado esperado do container será `healthy`.

#### Política de reinicialização

```yaml
restart: unless-stopped
```

O container poderá ser reiniciado automaticamente pelo Docker, exceto quando tiver sido parado intencionalmente.

### 2.2 Por que não montaremos o socket do Docker

S3, DynamoDB e SQS são executados internamente pelo Floci. Portanto, este primeiro laboratório não precisa montar:

```text
/var/run/docker.sock
```

Alguns serviços avançados criam containers auxiliares e exigem acesso ao Docker. Esse acesso aumenta o nível de privilégio do container e deve ser utilizado somente quando necessário.

### 2.3 Validar a sintaxe do Compose

Antes de iniciar o ambiente, execute:

```bash
docker compose config -q
```

Saída esperada:

```text
nenhuma saída
```

O código de retorno deve ser zero:

```bash
echo $?
```

Saída esperada:

```text
0
```

Também é possível visualizar a configuração final interpretada pelo Docker:

```bash
docker compose config
```

Esse comando ajuda a identificar:

- erro de indentação;
- nome de chave incorreto;
- variável não resolvida;
- volume ou porta configurados no nível errado.

## 3. Iniciar o Floci

### 3.1 Baixar a imagem

```bash
docker compose pull
```

Uma execução bem-sucedida termina indicando que a imagem foi baixada ou que já está atualizada.

### 3.2 Iniciar o ambiente em segundo plano

```bash
docker compose up -d
```

O parâmetro `-d` executa o ambiente em modo destacado, liberando o terminal.

### 3.3 Verificar o container

```bash
docker compose ps
```

Exemplo de saída válida:

```text
NAME    IMAGE                 COMMAND                  SERVICE   CREATED          STATUS                    PORTS
floci   floci/floci:latest    "/usr/local/bin/..."    floci     10 seconds ago   Up 9 seconds (healthy)    0.0.0.0:4566->4566/tcp
```

Os valores de tempo e o texto do comando podem variar. Verifique principalmente:

- `NAME` igual a `floci`;
- `STATUS` contendo `Up`;
- após alguns segundos, `STATUS` contendo `healthy`;
- mapeamento da porta `4566`.

### 3.4 Acompanhar os logs

```bash
docker compose logs --tail 50 floci
```

Para acompanhar continuamente:

```bash
docker compose logs -f floci
```

Use `Ctrl+C` para sair do acompanhamento. Isso interrompe apenas a visualização dos logs; o container continua ativo.

### 3.5 Aguardar o serviço de forma controlada

Em máquinas mais lentas, o container pode aparecer como iniciado antes de o endpoint estar pronto. Use:

```bash
for tentativa in $(seq 1 30); do
  if curl -fsS http://localhost:4566/_floci/health >/dev/null; then
    echo "OK: Floci respondeu na tentativa ${tentativa}."
    break
  fi

  if [ "$tentativa" -eq 30 ]; then
    echo "ERRO: Floci não ficou disponível dentro do tempo esperado."
    docker compose logs --tail 100 floci
    exit 1
  fi

  sleep 2
done
```

Esse laço:

1. tenta acessar o endpoint;
2. considera sucesso somente uma resposta HTTP válida;
3. aguarda dois segundos entre as tentativas;
4. mostra os logs caso o serviço não responda em até aproximadamente um minuto.

## 4. Validar o endpoint e reconhecer uma saída válida

Esta etapa confirma que o processo está acessível e que os serviços utilizados no laboratório foram inicializados.

### 4.1 Consultar o endpoint nativo

```bash
curl -fsS http://localhost:4566/_floci/health
printf '\n'
```

O Floci retornará um JSON com os serviços e seus respectivos estados. A lista completa pode ser extensa.

### 4.2 Exibir somente os serviços deste laboratório

Para tornar a validação objetiva, filtre S3, DynamoDB e SQS:

```bash
curl -fsS http://localhost:4566/_floci/health \
  | jq '{
      s3: .services.s3,
      dynamodb: .services.dynamodb,
      sqs: .services.sqs
    }'
```

Exemplo de saída válida:

```json
{
  "s3": "running",
  "dynamodb": "running",
  "sqs": "running"
}
```

Para este tutorial, uma validação positiva exige:

- resposta HTTP `200`;
- campo `services`;
- `s3` com valor `running`;
- `dynamodb` com valor `running`;
- `sqs` com valor `running`.

### 4.3 Validar automaticamente os três estados

```bash
SERVICOS_OK=$(
  curl -fsS http://localhost:4566/_floci/health \
    | jq -r '[.services.s3, .services.dynamodb, .services.sqs] | all(. == "running")'
)

if [ "$SERVICOS_OK" = "true" ]; then
  echo "OK: S3, DynamoDB e SQS estão em execução."
else
  echo "ERRO: pelo menos um serviço não está em execução."
  exit 1
fi
```

Saída esperada:

```text
OK: S3, DynamoDB e SQS estão em execução.
```

### 4.4 Conferir apenas o código HTTP

```bash
curl -fsS \
  -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  http://localhost:4566/_floci/health
```

Saída esperada:

```text
HTTP 200
```

### 4.5 Validar o endpoint compatível com LocalStack

O Floci também oferece o caminho compatível:

```bash
curl -fsS http://localhost:4566/_localstack/health \
  | jq -r '.services | {s3, dynamodb, sqs}'
```

Saída esperada:

```json
{
  "s3": "running",
  "dynamodb": "running",
  "sqs": "running"
}
```

Para novos laboratórios, prefira o endpoint nativo `/_floci/health`. O caminho `/_localstack/health` é útil em scripts que precisam manter compatibilidade.

### 4.6 Relacionar a resposta HTTP com o estado do Docker

```bash
docker inspect \
  --format 'container={{.State.Status}} health={{if .State.Health}}{{.State.Health.Status}}{{else}}sem-healthcheck{{end}}' \
  floci
```

Saída esperada:

```text
container=running health=healthy
```

As duas validações observam camadas diferentes:

- `container=running` confirma que o processo do container não terminou;
- `health=healthy` confirma que o teste configurado no Compose está passando;
- `HTTP 200` confirma que a API responde;
- os campos `running` confirmam que os serviços necessários estão disponíveis.

## 5. Configurar a AWS CLI para utilizar o Floci

### 5.1 Exportar credenciais e endpoint locais

As variáveis abaixo valem somente para a sessão atual do terminal:

```bash
unset AWS_PROFILE

export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_PAGER=""
export AWS_EC2_METADATA_DISABLED=true
```

O Floci aceita credenciais não vazias. `test/test` são valores fictícios usados apenas para assinar localmente as requisições.

`AWS_PAGER=""` evita que respostas longas sejam abertas em um paginador interativo.

`AWS_EC2_METADATA_DISABLED=true` impede tentativas desnecessárias de consultar o serviço de metadados de uma instância EC2.

### 5.2 Conferir a configuração sem exibir o segredo

```bash
printf 'Endpoint: %s\n' "$AWS_ENDPOINT_URL"
printf 'Região:   %s\n' "$AWS_DEFAULT_REGION"
printf 'Access:   %s\n' "$AWS_ACCESS_KEY_ID"
```

Saída esperada:

```text
Endpoint: http://localhost:4566
Região:   us-east-1
Access:   test
```

Não é necessário imprimir `AWS_SECRET_ACCESS_KEY`.

### 5.3 Criar uma proteção simples contra endpoint incorreto

```bash
if [ "$AWS_ENDPOINT_URL" != "http://localhost:4566" ]; then
  echo "ERRO: o endpoint não aponta para o laboratório local."
  exit 1
fi

echo "OK: endpoint local validado."
```

Saída esperada:

```text
OK: endpoint local validado.
```

### 5.4 Validar a identidade simulada

Execute:

```bash
aws sts get-caller-identity \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

A resposta terá a estrutura:

```json
{
  "UserId": "valor-simulado",
  "Account": "000000000000",
  "Arn": "arn:aws:iam::000000000000:..."
}
```

Os campos `UserId` e `Arn` podem variar conforme a versão. Para uma validação estável, consulte somente a conta:

```bash
aws sts get-caller-identity \
  --query Account \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
000000000000
```

Se o comando tentar acessar a internet ou retornar uma conta AWS real, interrompa o exercício e revise `AWS_ENDPOINT_URL` e `--endpoint-url`.

## 6. Exercício 1: Armazenamento de objetos com Amazon S3

### Objetivo

Neste exercício, você irá:

- criar um bucket;
- validar sua existência;
- criar arquivos locais;
- enviar objetos para o bucket;
- trabalhar com prefixos;
- consultar metadados;
- baixar e comparar o conteúdo;
- copiar e excluir um objeto;
- confirmar o estado final.

### Conceitos essenciais

| Conceito | Significado |
| --- | --- |
| Bucket | Contêiner lógico que armazena objetos |
| Objeto | Conteúdo armazenado, acompanhado de metadados |
| Key | Identificador completo do objeto dentro do bucket |
| Prefixo | Parte inicial da key, frequentemente exibida como diretório |
| Metadata | Pares de chave e valor associados ao objeto |
| ETag | Identificador retornado pelo S3; em uploads simples costuma se relacionar ao MD5, mas não deve ser tratado universalmente como checksum |

### 6.1 Definir o nome do bucket

```bash
export LAB_BUCKET=lab-certificacoes-floci
```

Valide:

```bash
echo "$LAB_BUCKET"
```

Saída esperada:

```text
lab-certificacoes-floci
```

Use somente letras minúsculas, números e hífens. Na AWS real, nomes de buckets possuem regras adicionais e o namespace é global. Neste laboratório, o recurso existe apenas no emulador local.

### 6.2 Verificar se o bucket já existe

```bash
if aws s3api head-bucket \
  --bucket "$LAB_BUCKET" \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  2>/dev/null; then
  echo "O bucket já existe."
else
  echo "O bucket ainda não existe."
fi
```

Na primeira execução, a saída esperada é:

```text
O bucket ainda não existe.
```

### 6.3 Criar o bucket

```bash
aws s3 mb \
  "s3://${LAB_BUCKET}" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
make_bucket: lab-certificacoes-floci
```

O comando `aws s3 mb` pertence ao conjunto de comandos de alto nível da AWS CLI. Ele simplifica a operação `CreateBucket`.

### 6.4 Validar o bucket com a API

```bash
aws s3api head-bucket \
  --bucket "$LAB_BUCKET" \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "Código de retorno: $?"
```

Saída esperada:

```text
Código de retorno: 0
```

`head-bucket` não apresenta corpo quando a validação é bem-sucedida. O código zero confirma o sucesso.

Liste os buckets e filtre pelo nome:

```bash
aws s3api list-buckets \
  --query "Buckets[?Name=='${LAB_BUCKET}'].Name | [0]" \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
lab-certificacoes-floci
```

### 6.5 Preparar os arquivos locais

Crie a estrutura:

```bash
mkdir -p arquivos/downloads
```

Crie o primeiro arquivo:

```bash
printf '%s\n' \
  'projeto=Laboratorio Floci' \
  'servico=Amazon S3' \
  'status=em-estudo' \
  > arquivos/estudo.txt
```

Visualize:

```bash
sed -n '1,20p' arquivos/estudo.txt
```

Saída esperada:

```text
projeto=Laboratorio Floci
servico=Amazon S3
status=em-estudo
```

Crie também um registro JSON:

```bash
printf '%s\n' \
  '{"projeto":"floci","servico":"s3","status":"objeto-criado"}' \
  > arquivos/execucao.json
```

Valide a sintaxe:

```bash
jq . arquivos/execucao.json
```

Saída esperada:

```json
{
  "projeto": "floci",
  "servico": "s3",
  "status": "objeto-criado"
}
```

### 6.6 Registrar o hash do arquivo original

```bash
sha256sum arquivos/estudo.txt
```

Exemplo:

```text
HASH_SHA256  arquivos/estudo.txt
```

O valor será uma sequência hexadecimal. Guarde a ideia, não o texto `HASH_SHA256`: posteriormente calcularemos o hash do arquivo baixado e compararemos os dois.

### 6.7 Enviar o primeiro objeto

```bash
aws s3 cp \
  arquivos/estudo.txt \
  "s3://${LAB_BUCKET}/materiais/estudo.txt" \
  --content-type "text/plain; charset=utf-8" \
  --metadata "autor=jeferson,projeto=floci" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
upload: arquivos/estudo.txt to s3://lab-certificacoes-floci/materiais/estudo.txt
```

Observe que a key completa é:

```text
materiais/estudo.txt
```

O prefixo `materiais/` não representa um diretório real. Ele faz parte do nome da key e é apresentado pelas ferramentas como uma estrutura hierárquica.

### 6.8 Enviar o objeto JSON para outro prefixo

```bash
aws s3 cp \
  arquivos/execucao.json \
  "s3://${LAB_BUCKET}/registros/execucao.json" \
  --content-type application/json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
upload: arquivos/execucao.json to s3://lab-certificacoes-floci/registros/execucao.json
```

### 6.9 Listar os objetos

Use o comando de alto nível:

```bash
aws s3 ls \
  "s3://${LAB_BUCKET}" \
  --recursive \
  --human-readable \
  --summarize \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

A saída deverá incluir:

```text
materiais/estudo.txt
registros/execucao.json
Total Objects: 2
```

Data, hora e tamanho serão apresentados junto às keys.

Agora use a API de baixo nível e selecione campos:

```bash
aws s3api list-objects-v2 \
  --bucket "$LAB_BUCKET" \
  --query 'Contents[].{Key:Key,Size:Size}' \
  --output table \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```text
-----------------------------------------
|             ListObjectsV2             |
+--------------------------+------------+
| Key                      | Size       |
+--------------------------+------------+
| materiais/estudo.txt     | valor      |
| registros/execucao.json  | valor      |
+--------------------------+------------+
```

Os tamanhos dependem do conteúdo exato dos arquivos.

### 6.10 Consultar os metadados sem baixar o objeto

```bash
aws s3api head-object \
  --bucket "$LAB_BUCKET" \
  --key materiais/estudo.txt \
  --query '{
    ContentLength: ContentLength,
    ContentType: ContentType,
    ETag: ETag,
    Metadata: Metadata
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```json
{
  "ContentLength": 61,
  "ContentType": "text/plain; charset=utf-8",
  "ETag": "\"valor-gerado-pelo-servico\"",
  "Metadata": {
    "autor": "jeferson",
    "projeto": "floci"
  }
}
```

O `ContentLength` e o `ETag` podem variar se o conteúdo for alterado. Valide principalmente:

- tipo de conteúdo;
- presença dos metadados;
- tamanho maior que zero.

Validação automatizada:

```bash
TAMANHO_OBJETO=$(
  aws s3api head-object \
    --bucket "$LAB_BUCKET" \
    --key materiais/estudo.txt \
    --query ContentLength \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

if [ "$TAMANHO_OBJETO" -gt 0 ]; then
  echo "OK: objeto possui ${TAMANHO_OBJETO} bytes."
else
  echo "ERRO: objeto vazio ou não encontrado."
  exit 1
fi
```

### 6.11 Ler o objeto diretamente no terminal

```bash
aws s3 cp \
  "s3://${LAB_BUCKET}/materiais/estudo.txt" \
  - \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
projeto=Laboratorio Floci
servico=Amazon S3
status=em-estudo
```

O hífen `-` representa a saída padrão do terminal.

### 6.12 Baixar o objeto

```bash
aws s3 cp \
  "s3://${LAB_BUCKET}/materiais/estudo.txt" \
  arquivos/downloads/estudo-baixado.txt \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
download: s3://lab-certificacoes-floci/materiais/estudo.txt to arquivos/downloads/estudo-baixado.txt
```

### 6.13 Comparar o conteúdo original com o baixado

```bash
cmp -s \
  arquivos/estudo.txt \
  arquivos/downloads/estudo-baixado.txt \
  && echo "OK: os arquivos são idênticos." \
  || echo "ERRO: os arquivos são diferentes."
```

Saída esperada:

```text
OK: os arquivos são idênticos.
```

Compare também os hashes:

```bash
sha256sum \
  arquivos/estudo.txt \
  arquivos/downloads/estudo-baixado.txt
```

Saída válida:

```text
MESMO_HASH  arquivos/estudo.txt
MESMO_HASH  arquivos/downloads/estudo-baixado.txt
```

Os dois valores hexadecimais devem ser iguais.

### 6.14 Copiar um objeto dentro do bucket

```bash
aws s3 cp \
  "s3://${LAB_BUCKET}/materiais/estudo.txt" \
  "s3://${LAB_BUCKET}/backup/estudo.txt" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Confirme a quantidade:

```bash
aws s3api list-objects-v2 \
  --bucket "$LAB_BUCKET" \
  --query 'length(Contents)' \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
3
```

### 6.15 Excluir somente a cópia

```bash
aws s3 rm \
  "s3://${LAB_BUCKET}/backup/estudo.txt" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
delete: s3://lab-certificacoes-floci/backup/estudo.txt
```

Confirme o estado final:

```bash
aws s3api list-objects-v2 \
  --bucket "$LAB_BUCKET" \
  --query 'sort_by(Contents,&Key)[].Key' \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
materiais/estudo.txt    registros/execucao.json
```

Mantenha esses dois objetos. Eles serão utilizados na validação de persistência.

### Resultado do exercício S3

Ao final deste exercício, você deverá ter:

- bucket `lab-certificacoes-floci`;
- objeto `materiais/estudo.txt`;
- objeto `registros/execucao.json`;
- metadados no objeto de texto;
- cópia local baixada e validada;
- integridade comprovada por `cmp` e SHA-256.

## 7. Exercício 2: Dados NoSQL com Amazon DynamoDB

### Objetivo

Neste exercício, você irá:

- criar uma tabela;
- definir uma chave de partição;
- aguardar o estado ativo;
- inserir um item;
- ler o item pela chave;
- utilizar `Query`;
- utilizar `Scan`;
- atualizar atributos com condição;
- projetar somente os campos necessários;
- testar uma consulta de chave inexistente.

### Conceitos essenciais

| Conceito | Significado |
| --- | --- |
| Tabela | Coleção de itens |
| Item | Conjunto de atributos identificado por uma chave |
| Atributo | Campo pertencente a um item |
| Partition key | Chave usada para distribuir e localizar dados |
| `S` | Tipo String no formato JSON do DynamoDB |
| `N` | Tipo Number no formato JSON do DynamoDB |
| `Query` | Consulta orientada pela chave de partição |
| `Scan` | Leitura de todos os itens avaliados na tabela |
| `PAY_PER_REQUEST` | Modo de cobrança sob demanda na AWS real; no laboratório não há cobrança |

### 7.1 Definir o nome da tabela

```bash
export LAB_TABLE=Estudos
```

Valide:

```bash
echo "$LAB_TABLE"
```

Saída esperada:

```text
Estudos
```

### 7.2 Verificar se a tabela já existe

```bash
if aws dynamodb describe-table \
  --table-name "$LAB_TABLE" \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  >/dev/null 2>&1; then
  echo "A tabela já existe."
else
  echo "A tabela ainda não existe."
fi
```

Na primeira execução:

```text
A tabela ainda não existe.
```

### 7.3 Criar a tabela

```bash
aws dynamodb create-table \
  --table-name "$LAB_TABLE" \
  --attribute-definitions \
    AttributeName=id,AttributeType=S \
  --key-schema \
    AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --query 'TableDescription.{
    Tabela: TableName,
    Status: TableStatus,
    Chave: KeySchema
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```json
{
  "Tabela": "Estudos",
  "Status": "ACTIVE",
  "Chave": [
    {
      "AttributeName": "id",
      "KeyType": "HASH"
    }
  ]
}
```

Interpretação:

- `id` é uma string;
- `id` é a chave de partição;
- `HASH` é o nome utilizado pela API para a partition key;
- `ACTIVE` significa que a tabela está disponível.

### 7.4 Aguardar a tabela

Mesmo que o emulador geralmente ative a tabela rapidamente, utilize o waiter da AWS CLI:

```bash
aws dynamodb wait table-exists \
  --table-name "$LAB_TABLE" \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "OK: tabela disponível."
```

Saída esperada:

```text
OK: tabela disponível.
```

Na AWS real, essa prática é importante porque várias operações são assíncronas.

### 7.5 Listar e descrever a tabela

```bash
aws dynamodb list-tables \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "TableNames": [
    "Estudos"
  ]
}
```

Consulte os principais detalhes:

```bash
aws dynamodb describe-table \
  --table-name "$LAB_TABLE" \
  --query 'Table.{
    Nome: TableName,
    Status: TableStatus,
    Chave: KeySchema,
    Atributos: AttributeDefinitions
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Valide:

- nome `Estudos`;
- status `ACTIVE`;
- chave `id`;
- tipo `S`.

### 7.6 Inserir o item

```bash
aws dynamodb put-item \
  --table-name "$LAB_TABLE" \
  --item '{
    "id": {"S": "projeto-001"},
    "certificacao": {"S": "AWS"},
    "tema": {"S": "S3, DynamoDB e SQS"},
    "status": {"S": "concluido"},
    "execucoes": {"N": "1"}
  }' \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "Código de retorno: $?"
```

Saída esperada:

```text
Código de retorno: 0
```

Uma operação `PutItem` bem-sucedida pode não apresentar corpo. O código zero indica que a solicitação foi aceita.

Observe:

- strings usam `{"S":"valor"}`;
- números usam `{"N":"1"}`;
- embora `1` seja número lógico, o protocolo representa o valor dentro de uma string;
- a chave `id=projeto-001` identifica unicamente o item.

### 7.7 Consultar diretamente pela chave

```bash
aws dynamodb get-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --consistent-read \
  --query Item \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "id": {
    "S": "projeto-001"
  },
  "certificacao": {
    "S": "AWS"
  },
  "tema": {
    "S": "S3, DynamoDB e SQS"
  },
  "status": {
    "S": "concluido"
  },
  "execucoes": {
    "N": "1"
  }
}
```

A ordem dos atributos pode variar. O conteúdo deve ser equivalente.

### 7.8 Validar somente um atributo

```bash
STATUS_ITEM=$(
  aws dynamodb get-item \
    --table-name "$LAB_TABLE" \
    --key '{"id":{"S":"projeto-001"}}' \
    --query 'Item.status.S' \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

if [ "$STATUS_ITEM" = "concluido" ]; then
  echo "OK: item encontrado com o status esperado."
else
  echo "ERRO: status inesperado: $STATUS_ITEM"
  exit 1
fi
```

Saída esperada:

```text
OK: item encontrado com o status esperado.
```

### 7.9 Consultar com `Query`

```bash
aws dynamodb query \
  --table-name "$LAB_TABLE" \
  --key-condition-expression "id = :id" \
  --expression-attribute-values '{
    ":id": {"S": "projeto-001"}
  }' \
  --query '{
    Quantidade: Count,
    Itens: Items
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

A saída deverá conter:

```json
{
  "Quantidade": 1,
  "Itens": [
    {
      "id": {
        "S": "projeto-001"
      }
    }
  ]
}
```

O item real conterá também os demais atributos.

`Query` utiliza uma condição relacionada à chave. Em tabelas grandes, esse padrão é normalmente mais eficiente do que examinar toda a tabela.

### 7.10 Listar o conteúdo com `Scan`

```bash
aws dynamodb scan \
  --table-name "$LAB_TABLE" \
  --query '{
    Quantidade: Count,
    Avaliados: ScannedCount,
    Itens: Items
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada neste laboratório:

```json
{
  "Quantidade": 1,
  "Avaliados": 1,
  "Itens": [
    {
      "id": {
        "S": "projeto-001"
      }
    }
  ]
}
```

O item exibido conterá todos os atributos. Como existe somente um item:

- `Count` deve ser `1`;
- `ScannedCount` deve ser `1`.

Na AWS real, `Scan` pode consumir muitos recursos porque avalia a tabela. Use-o com cuidado em ambientes grandes.

### 7.11 Atualizar o item com uma condição

Registre o horário UTC:

```bash
ATUALIZADO_EM=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
echo "$ATUALIZADO_EM"
```

Atualize o status somente se o item já existir:

```bash
aws dynamodb update-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --update-expression \
    "SET #st = :novoStatus, atualizadoEm = :data" \
  --condition-expression \
    "attribute_exists(id)" \
  --expression-attribute-names \
    '{"#st":"status"}' \
  --expression-attribute-values \
    "{\":novoStatus\":{\"S\":\"validado\"},\":data\":{\"S\":\"${ATUALIZADO_EM}\"}}" \
  --return-values ALL_NEW \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```json
{
  "Attributes": {
    "id": {
      "S": "projeto-001"
    },
    "status": {
      "S": "validado"
    },
    "atualizadoEm": {
      "S": "DATA_UTC_GERADA"
    }
  }
}
```

Os demais atributos também serão apresentados. `DATA_UTC_GERADA` representa o valor real gerado pelo comando `date`.

Por que utilizamos `#st`?

Alguns nomes podem colidir com palavras reservadas do DynamoDB. `ExpressionAttributeNames` cria um alias seguro para o atributo `status`.

Por que utilizamos `attribute_exists(id)`?

Sem uma condição, `UpdateItem` pode criar um novo item quando a chave não existe. A condição exige que o item já esteja presente.

### 7.12 Projetar somente os campos necessários

```bash
aws dynamodb get-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --projection-expression "id,#st,atualizadoEm" \
  --expression-attribute-names '{"#st":"status"}' \
  --query Item \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "id": {
    "S": "projeto-001"
  },
  "status": {
    "S": "validado"
  },
  "atualizadoEm": {
    "S": "DATA_UTC_GERADA"
  }
}
```

Uma projeção reduz os dados retornados quando a aplicação precisa de poucos atributos.

### 7.13 Consultar uma chave inexistente

```bash
aws dynamodb get-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"nao-existe"}}' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{}
```

Uma chave ausente não é necessariamente um erro de protocolo. A requisição pode terminar com código zero, mas sem o campo `Item`.

Validação explícita:

```bash
ITEM_INEXISTENTE=$(
  aws dynamodb get-item \
    --table-name "$LAB_TABLE" \
    --key '{"id":{"S":"nao-existe"}}' \
    --query 'Item' \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

if [ "$ITEM_INEXISTENTE" = "None" ]; then
  echo "OK: a chave inexistente não retornou item."
else
  echo "Revise a resposta: $ITEM_INEXISTENTE"
fi
```

Dependendo da versão da AWS CLI, um valor ausente pode ser exibido como `None` ou como saída vazia. O ponto principal é não receber um item.

### 7.14 Confirmar o estado final

```bash
aws dynamodb scan \
  --table-name "$LAB_TABLE" \
  --select COUNT \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "Count": 1,
  "ScannedCount": 1
}
```

Mantenha a tabela e o item. Eles serão usados na validação de persistência.

### Resultado do exercício DynamoDB

Ao final deste exercício, você deverá ter:

- tabela `Estudos` em estado `ACTIVE`;
- chave de partição `id` do tipo String;
- um item identificado por `projeto-001`;
- status atualizado para `validado`;
- data de atualização registrada;
- diferença prática observada entre `GetItem`, `Query` e `Scan`.

## 8. Exercício 3: Mensageria assíncrona com Amazon SQS

### Objetivo

Neste exercício, você irá:

- criar uma fila padrão;
- configurar o tempo de invisibilidade;
- obter e validar a URL;
- enviar uma mensagem;
- adicionar atributos à mensagem;
- inspecionar a fila sem consumir;
- receber a mensagem;
- observar a invisibilidade;
- excluir a mensagem com `ReceiptHandle`;
- confirmar que a fila ficou vazia.

### Conceitos essenciais

| Conceito | Significado |
| --- | --- |
| Queue URL | Endereço utilizado nas operações da fila |
| Message body | Conteúdo principal da mensagem |
| Message attributes | Metadados tipados associados à mensagem |
| Visibility timeout | Período no qual uma mensagem recebida fica invisível para outros consumidores |
| ReceiptHandle | Identificador temporário usado para excluir ou alterar a visibilidade de uma mensagem recebida |
| Long polling | Espera por mensagens durante alguns segundos antes de retornar vazio |

Em um fluxo típico:

1. um produtor envia a mensagem;
2. um consumidor recebe a mensagem;
3. a mensagem fica temporariamente invisível;
4. o consumidor processa o conteúdo;
5. o consumidor exclui a mensagem usando o `ReceiptHandle`;
6. se o consumidor não excluir a mensagem, ela volta a ficar disponível após o tempo de invisibilidade.

### 8.1 Definir o nome da fila

```bash
export LAB_QUEUE=fila-estudos
```

Valide:

```bash
echo "$LAB_QUEUE"
```

Saída esperada:

```text
fila-estudos
```

### 8.2 Criar a fila

Crie uma fila padrão com:

- 30 segundos de visibilidade;
- dois segundos de long polling como padrão.

```bash
QUEUE_URL=$(
  aws sqs create-queue \
    --queue-name "$LAB_QUEUE" \
    --attributes \
      VisibilityTimeout=30,ReceiveMessageWaitTimeSeconds=2 \
    --query QueueUrl \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

export QUEUE_URL
```

Exiba:

```bash
echo "$QUEUE_URL"
```

Saída esperada:

```text
http://localhost:4566/000000000000/fila-estudos
```

A URL contém:

- endpoint local;
- identificador da conta simulada;
- nome da fila.

### 8.3 Validar que a URL não está vazia

```bash
if [ -n "$QUEUE_URL" ] && [ "$QUEUE_URL" != "None" ]; then
  echo "OK: URL da fila capturada."
else
  echo "ERRO: não foi possível obter a URL da fila."
  exit 1
fi
```

Saída esperada:

```text
OK: URL da fila capturada.
```

### 8.4 Listar e localizar a fila

```bash
aws sqs list-queues \
  --queue-name-prefix "$LAB_QUEUE" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "QueueUrls": [
    "http://localhost:4566/000000000000/fila-estudos"
  ]
}
```

### 8.5 Consultar os atributos

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names All \
  --query 'Attributes.{
    QueueArn: QueueArn,
    VisibilityTimeout: VisibilityTimeout,
    ReceiveMessageWaitTimeSeconds: ReceiveMessageWaitTimeSeconds,
    ApproximateNumberOfMessages: ApproximateNumberOfMessages
  }' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```json
{
  "QueueArn": "arn:aws:sqs:us-east-1:000000000000:fila-estudos",
  "VisibilityTimeout": "30",
  "ReceiveMessageWaitTimeSeconds": "2",
  "ApproximateNumberOfMessages": "0"
}
```

### 8.6 Preparar o corpo da mensagem

```bash
MESSAGE_BODY='{"projeto":"floci","servico":"sqs","status":"pronto-para-processamento"}'
```

Valide:

```bash
printf '%s\n' "$MESSAGE_BODY" | jq .
```

Saída esperada:

```json
{
  "projeto": "floci",
  "servico": "sqs",
  "status": "pronto-para-processamento"
}
```

### 8.7 Enviar a mensagem

```bash
aws sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body "$MESSAGE_BODY" \
  --message-attributes '{
    "Origem": {
      "DataType": "String",
      "StringValue": "laboratorio-floci"
    },
    "Prioridade": {
      "DataType": "Number",
      "StringValue": "1"
    }
  }' \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Exemplo de saída válida:

```json
{
  "MD5OfMessageBody": "valor-md5",
  "MD5OfMessageAttributes": "valor-md5",
  "MessageId": "identificador-gerado"
}
```

Os valores MD5 e `MessageId` variam a cada execução.

### 8.8 Verificar a quantidade de mensagens disponíveis

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names \
    ApproximateNumberOfMessages \
    ApproximateNumberOfMessagesNotVisible \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Logo após o envio, a resposta deverá indicar aproximadamente:

```json
{
  "Attributes": {
    "ApproximateNumberOfMessages": "1",
    "ApproximateNumberOfMessagesNotVisible": "0"
  }
}
```

Na AWS real, campos com o prefixo `Approximate` não devem ser tratados como contadores transacionais exatos.

### 8.9 Inspecionar a mensagem sem consumi-la

O Floci disponibiliza um endpoint local de inspeção:

```bash
curl -fsS -G \
  "${AWS_ENDPOINT_URL}/_aws/sqs/messages" \
  --data-urlencode "QueueUrl=${QUEUE_URL}" \
  | jq .
```

Exemplo de estrutura válida:

```json
{
  "messages": [
    {
      "MessageId": "identificador-gerado",
      "MD5OfBody": "valor-md5",
      "Body": "{\"projeto\":\"floci\",\"servico\":\"sqs\",\"status\":\"pronto-para-processamento\"}",
      "ReceiptHandle": null,
      "Attributes": {
        "ApproximateReceiveCount": "0"
      },
      "MessageAttributes": {
        "Origem": {
          "DataType": "String",
          "StringValue": "laboratorio-floci"
        }
      }
    }
  ]
}
```

Esse endpoint:

- não consome a mensagem;
- não incrementa a contagem de recebimentos;
- pode exibir mensagens visíveis e em processamento;
- é útil para depuração e testes locais;
- não é uma API que deve ser esperada na AWS real.

### 8.10 Receber a mensagem

```bash
aws sqs receive-message \
  --queue-url "$QUEUE_URL" \
  --max-number-of-messages 1 \
  --wait-time-seconds 2 \
  --visibility-timeout 30 \
  --attribute-names All \
  --message-attribute-names All \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  > recebimento.json
```

Visualize:

```bash
jq . recebimento.json
```

Exemplo de saída válida:

```json
{
  "Messages": [
    {
      "MessageId": "identificador-gerado",
      "ReceiptHandle": "identificador-temporario",
      "MD5OfBody": "valor-md5",
      "Body": "{\"projeto\":\"floci\",\"servico\":\"sqs\",\"status\":\"pronto-para-processamento\"}",
      "Attributes": {
        "ApproximateReceiveCount": "1"
      },
      "MessageAttributes": {
        "Origem": {
          "StringValue": "laboratorio-floci",
          "DataType": "String"
        },
        "Prioridade": {
          "StringValue": "1",
          "DataType": "Number"
        }
      }
    }
  ]
}
```

### 8.11 Extrair e validar o conteúdo recebido

Extraia o corpo:

```bash
CORPO_RECEBIDO=$(jq -r '.Messages[0].Body // empty' recebimento.json)
printf '%s\n' "$CORPO_RECEBIDO" | jq .
```

Saída esperada:

```json
{
  "projeto": "floci",
  "servico": "sqs",
  "status": "pronto-para-processamento"
}
```

Compare o corpo enviado com o recebido:

```bash
if [ "$CORPO_RECEBIDO" = "$MESSAGE_BODY" ]; then
  echo "OK: o corpo recebido é igual ao enviado."
else
  echo "ERRO: o corpo recebido é diferente."
  exit 1
fi
```

Saída esperada:

```text
OK: o corpo recebido é igual ao enviado.
```

### 8.12 Extrair o `ReceiptHandle`

```bash
RECEIPT_HANDLE=$(jq -r '.Messages[0].ReceiptHandle // empty' recebimento.json)
```

Valide sem exibir o identificador completo:

```bash
if [ -n "$RECEIPT_HANDLE" ]; then
  echo "OK: ReceiptHandle recebido."
else
  echo "ERRO: ReceiptHandle ausente."
  exit 1
fi
```

Saída esperada:

```text
OK: ReceiptHandle recebido.
```

O `ReceiptHandle` pertence a esse recebimento específico. Um novo recebimento da mesma mensagem pode gerar outro valor.

### 8.13 Observar a mensagem invisível

Antes de excluir a mensagem, consulte os contadores:

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names \
    ApproximateNumberOfMessages \
    ApproximateNumberOfMessagesNotVisible \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Durante o tempo de invisibilidade, o comportamento esperado é:

```json
{
  "Attributes": {
    "ApproximateNumberOfMessages": "0",
    "ApproximateNumberOfMessagesNotVisible": "1"
  }
}
```

O endpoint de inspeção local ainda pode mostrar a mensagem, pois ele inclui mensagens em processamento:

```bash
curl -fsS -G \
  "${AWS_ENDPOINT_URL}/_aws/sqs/messages" \
  --data-urlencode "QueueUrl=${QUEUE_URL}" \
  | jq '.messages | length'
```

Saída esperada:

```text
1
```

### 8.14 Excluir a mensagem após o processamento

```bash
aws sqs delete-message \
  --queue-url "$QUEUE_URL" \
  --receipt-handle "$RECEIPT_HANDLE" \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "Código de retorno: $?"
```

Saída esperada:

```text
Código de retorno: 0
```

Uma exclusão bem-sucedida normalmente não apresenta corpo.

### 8.15 Confirmar que a fila está vazia

```bash
curl -fsS -G \
  "${AWS_ENDPOINT_URL}/_aws/sqs/messages" \
  --data-urlencode "QueueUrl=${QUEUE_URL}" \
  | jq '.messages | length'
```

Saída esperada:

```text
0
```

Também valide pela API:

```bash
aws sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attribute-names \
    ApproximateNumberOfMessages \
    ApproximateNumberOfMessagesNotVisible \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Os dois valores devem ser `0`.

Mantenha a fila criada. A existência dela será verificada após a recriação do container.

### Resultado do exercício SQS

Ao final deste exercício, você deverá ter:

- fila `fila-estudos`;
- URL local válida;
- configuração de 30 segundos de visibilidade;
- mensagem enviada e recebida;
- corpo validado;
- `ReceiptHandle` utilizado para confirmar o processamento;
- fila existente e sem mensagens pendentes.

## 9. Exercício integrado: Relacionar S3, DynamoDB e SQS

Até aqui, os serviços foram praticados separadamente. Agora simularemos um fluxo simples:

1. um arquivo de resultado já está armazenado no S3;
2. o DynamoDB registra onde esse resultado está;
3. uma mensagem informa que o laboratório foi validado;
4. um consumidor recebe o evento;
5. o consumidor verifica o objeto e o registro;
6. a mensagem é excluída após a validação.

Esse fluxo representa uma arquitetura orientada a eventos em escala reduzida.

### 9.1 Atualizar o item com a localização do objeto

```bash
aws dynamodb update-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --update-expression \
    "SET bucketS3 = :bucket, objetoS3 = :objeto" \
  --expression-attribute-values \
    "{\":bucket\":{\"S\":\"${LAB_BUCKET}\"},\":objeto\":{\"S\":\"materiais/estudo.txt\"}}" \
  --return-values UPDATED_NEW \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
{
  "Attributes": {
    "bucketS3": {
      "S": "lab-certificacoes-floci"
    },
    "objetoS3": {
      "S": "materiais/estudo.txt"
    }
  }
}
```

### 9.2 Construir um evento JSON de forma segura

Use `jq` para evitar erros de escape:

```bash
EVENTO_INTEGRADO=$(
  jq -nc \
    --arg evento "laboratorio.validado" \
    --arg id "projeto-001" \
    --arg bucket "$LAB_BUCKET" \
    --arg key "materiais/estudo.txt" \
    '{
      evento: $evento,
      id: $id,
      bucket: $bucket,
      key: $key
    }'
)
```

Visualize:

```bash
printf '%s\n' "$EVENTO_INTEGRADO" | jq .
```

Saída esperada:

```json
{
  "evento": "laboratorio.validado",
  "id": "projeto-001",
  "bucket": "lab-certificacoes-floci",
  "key": "materiais/estudo.txt"
}
```

### 9.3 Enviar o evento

```bash
aws sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body "$EVENTO_INTEGRADO" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Confirme sem consumir:

```bash
curl -fsS -G \
  "${AWS_ENDPOINT_URL}/_aws/sqs/messages" \
  --data-urlencode "QueueUrl=${QUEUE_URL}" \
  | jq -r '.messages[0].Body | fromjson'
```

### 9.4 Simular o consumidor

```bash
aws sqs receive-message \
  --queue-url "$QUEUE_URL" \
  --max-number-of-messages 1 \
  --wait-time-seconds 2 \
  --attribute-names All \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  > evento-integrado.json
```

Valide:

```bash
jq . evento-integrado.json
```

Extraia os campos:

```bash
CORPO_EVENTO=$(jq -r '.Messages[0].Body // empty' evento-integrado.json)
EVENTO_ID=$(printf '%s\n' "$CORPO_EVENTO" | jq -r '.id')
EVENTO_BUCKET=$(printf '%s\n' "$CORPO_EVENTO" | jq -r '.bucket')
EVENTO_KEY=$(printf '%s\n' "$CORPO_EVENTO" | jq -r '.key')
EVENTO_RECEIPT=$(jq -r '.Messages[0].ReceiptHandle // empty' evento-integrado.json)
```

Confira sem expor o `ReceiptHandle`:

```bash
printf 'ID:     %s\n' "$EVENTO_ID"
printf 'Bucket: %s\n' "$EVENTO_BUCKET"
printf 'Key:    %s\n' "$EVENTO_KEY"
```

Saída esperada:

```text
ID:     projeto-001
Bucket: lab-certificacoes-floci
Key:    materiais/estudo.txt
```

### 9.5 Validar o objeto indicado pelo evento

```bash
aws s3api head-object \
  --bucket "$EVENTO_BUCKET" \
  --key "$EVENTO_KEY" \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  >/dev/null

echo "Código S3: $?"
```

Saída esperada:

```text
Código S3: 0
```

### 9.6 Validar o registro indicado pelo evento

```bash
ID_DYNAMO=$(
  aws dynamodb get-item \
    --table-name "$LAB_TABLE" \
    --key "{\"id\":{\"S\":\"${EVENTO_ID}\"}}" \
    --query 'Item.id.S' \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

if [ "$ID_DYNAMO" = "$EVENTO_ID" ]; then
  echo "OK: registro localizado no DynamoDB."
else
  echo "ERRO: registro não localizado."
  exit 1
fi
```

Saída esperada:

```text
OK: registro localizado no DynamoDB.
```

### 9.7 Confirmar o processamento

Somente após as duas validações:

```bash
aws sqs delete-message \
  --queue-url "$QUEUE_URL" \
  --receipt-handle "$EVENTO_RECEIPT" \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "OK: evento processado e removido da fila."
```

Saída esperada:

```text
OK: evento processado e removido da fila.
```

### Resultado do exercício integrado

Você simulou um consumidor que:

- recebeu um evento no SQS;
- interpretou o JSON;
- localizou o objeto no S3;
- localizou o item no DynamoDB;
- confirmou o processamento excluindo a mensagem.

## 10. Validar a persistência

Agora comprovaremos que os dados não dependem apenas do processo atual do container.

### 10.1 Confirmar o volume montado

```bash
docker inspect floci \
  --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
```

Exemplo de saída:

```text
laboratorio-floci_floci-data -> /app/data
```

O prefixo do volume pode mudar conforme o nome do diretório ou do projeto Compose. O destino deve ser:

```text
/app/data
```

### 10.2 Registrar o estado antes da reinicialização

S3:

```bash
aws s3api list-objects-v2 \
  --bucket "$LAB_BUCKET" \
  --query 'sort_by(Contents,&Key)[].Key' \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

DynamoDB:

```bash
aws dynamodb get-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --query 'Item.{id:id.S,status:status.S,bucket:bucketS3.S,key:objetoS3.S}' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

SQS:

```bash
aws sqs get-queue-url \
  --queue-name "$LAB_QUEUE" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Todos devem retornar os recursos criados.

### 10.3 Reiniciar o container

```bash
docker compose restart floci
```

Aguarde:

```bash
for tentativa in $(seq 1 30); do
  if curl -fsS http://localhost:4566/_floci/health >/dev/null; then
    echo "OK: Floci voltou após o restart."
    break
  fi

  if [ "$tentativa" -eq 30 ]; then
    echo "ERRO: serviço não voltou."
    docker compose logs --tail 100 floci
    exit 1
  fi

  sleep 2
done
```

### 10.4 Verificar os recursos após o restart

S3:

```bash
aws s3api head-object \
  --bucket "$LAB_BUCKET" \
  --key materiais/estudo.txt \
  --query '{Tamanho:ContentLength,Tipo:ContentType}' \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

DynamoDB:

```bash
aws dynamodb get-item \
  --table-name "$LAB_TABLE" \
  --key '{"id":{"S":"projeto-001"}}' \
  --query 'Item.status.S' \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
validado
```

SQS:

```bash
aws sqs get-queue-url \
  --queue-name "$LAB_QUEUE" \
  --query QueueUrl \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
http://localhost:4566/000000000000/fila-estudos
```

### 10.5 Recriar o container sem remover o volume

Essa é uma validação mais forte:

```bash
docker compose down
docker compose up -d
```

`docker compose down` remove o container e a rede do projeto, mas não remove o volume nomeado quando `-v` não é utilizado.

Aguarde o estado saudável:

```bash
for tentativa in $(seq 1 30); do
  STATUS=$(docker inspect --format '{{if .State.Health}}{{.State.Health.Status}}{{else}}unknown{{end}}' floci 2>/dev/null || true)

  if [ "$STATUS" = "healthy" ]; then
    echo "OK: novo container está saudável."
    break
  fi

  if [ "$tentativa" -eq 30 ]; then
    echo "ERRO: o novo container não ficou saudável."
    docker compose logs --tail 100 floci
    exit 1
  fi

  sleep 2
done
```

### 10.6 Executar uma validação consolidada

```bash
S3_OK=$(
  aws s3api head-object \
    --bucket "$LAB_BUCKET" \
    --key materiais/estudo.txt \
    --query ContentLength \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL" \
    2>/dev/null || true
)

DYNAMO_OK=$(
  aws dynamodb get-item \
    --table-name "$LAB_TABLE" \
    --key '{"id":{"S":"projeto-001"}}' \
    --query 'Item.id.S' \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL" \
    2>/dev/null || true
)

SQS_OK=$(
  aws sqs get-queue-url \
    --queue-name "$LAB_QUEUE" \
    --query QueueUrl \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL" \
    2>/dev/null || true
)

if [ -n "$S3_OK" ] \
  && [ "$DYNAMO_OK" = "projeto-001" ] \
  && [ "$SQS_OK" = "http://localhost:4566/000000000000/fila-estudos" ]; then
  echo "OK: S3, DynamoDB e SQS persistiram após a recriação do container."
else
  echo "ERRO: a validação de persistência falhou."
  printf 'S3=%s\nDynamoDB=%s\nSQS=%s\n' "$S3_OK" "$DYNAMO_OK" "$SQS_OK"
  exit 1
fi
```

Saída esperada:

```text
OK: S3, DynamoDB e SQS persistiram após a recriação do container.
```

### 10.7 O que foi comprovado

Essa validação demonstra que:

- o container original foi removido;
- um novo container foi criado;
- o volume nomeado permaneceu;
- os objetos do S3 permaneceram;
- a tabela e o item do DynamoDB permaneceram;
- a fila do SQS permaneceu;
- as variáveis da sua sessão continuaram direcionando a AWS CLI ao endpoint local.

Se abrir um novo terminal, exporte novamente as variáveis da seção 5.

## 11. Parar, limpar ou remover o laboratório

Escolha a ação de acordo com seu objetivo.

### Opção A: Parar e continuar depois

```bash
docker compose stop
```

Mantém:

- container;
- volume;
- recursos;
- configuração.

Para voltar:

```bash
docker compose start
```

### Opção B: Remover o container e preservar os dados

```bash
docker compose down
```

Mantém o volume nomeado. Na próxima execução:

```bash
docker compose up -d
```

Os recursos deverão ser restaurados.

### Opção C: Remover primeiro os recursos pela AWS CLI

Essa opção permite praticar o ciclo de vida de cada serviço.

#### Excluir a fila

```bash
aws sqs delete-queue \
  --queue-url "$QUEUE_URL" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Valide:

```bash
aws sqs list-queues \
  --queue-name-prefix "$LAB_QUEUE" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

A fila não deve ser apresentada.

#### Excluir a tabela

```bash
aws dynamodb delete-table \
  --table-name "$LAB_TABLE" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Aguarde:

```bash
aws dynamodb wait table-not-exists \
  --table-name "$LAB_TABLE" \
  --endpoint-url "$AWS_ENDPOINT_URL"

echo "OK: tabela removida."
```

#### Excluir os objetos e o bucket

Primeiro remova os objetos:

```bash
aws s3 rm \
  "s3://${LAB_BUCKET}" \
  --recursive \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Depois remova o bucket:

```bash
aws s3 rb \
  "s3://${LAB_BUCKET}" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Valide:

```bash
aws s3api list-buckets \
  --query "Buckets[?Name=='${LAB_BUCKET}'].Name" \
  --output json \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Saída esperada:

```json
[]
```

### Opção D: Apagar container e dados persistentes

> Atenção: execute esta opção somente se não precisar mais dos recursos do laboratório.

```bash
docker compose down -v
```

O parâmetro `-v` remove também o volume declarado no arquivo Compose.

Confirme:

```bash
docker compose ps -a
docker volume ls
```

O container não deverá aparecer no projeto. O volume do laboratório também não deverá existir.

### Limpar arquivos locais gerados

Depois de concluir o laboratório, estes arquivos podem ser removidos manualmente:

```text
arquivos/
recebimento.json
evento-integrado.json
```

O arquivo `compose.yaml` pode ser mantido para repetir o projeto.

## 12. Como repetir o laboratório com segurança

Quando o ambiente já contém recursos, alguns comandos de criação podem retornar erros indicando que o recurso existe.

Antes de repetir:

```bash
docker compose up -d
```

Exporte novamente:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_PAGER=""
export AWS_EC2_METADATA_DISABLED=true

export LAB_BUCKET=lab-certificacoes-floci
export LAB_TABLE=Estudos
export LAB_QUEUE=fila-estudos
```

Recupere a URL da fila existente:

```bash
QUEUE_URL=$(
  aws sqs get-queue-url \
    --queue-name "$LAB_QUEUE" \
    --query QueueUrl \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

export QUEUE_URL
```

Antes de recriar qualquer recurso, consulte:

```bash
aws s3api head-bucket \
  --bucket "$LAB_BUCKET" \
  --endpoint-url "$AWS_ENDPOINT_URL"

aws dynamodb describe-table \
  --table-name "$LAB_TABLE" \
  --endpoint-url "$AWS_ENDPOINT_URL"

aws sqs get-queue-url \
  --queue-name "$LAB_QUEUE" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

## Solução de problemas

### 1. A porta `4566` já está em uso

Linux:

```bash
ss -lntp | grep ':4566'
```

Alternativa:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep 4566
```

Se outro container estiver usando a porta, pare somente o container identificado ou altere o mapeamento:

```yaml
ports:
  - "4567:4566"
```

Nesse caso, atualize:

```bash
export AWS_ENDPOINT_URL=http://localhost:4567
```

O endpoint interno do healthcheck continua em `4566`, porque ele é executado dentro do container.

### 2. `Could not connect to the endpoint URL`

Valide:

```bash
docker compose ps
curl -v http://localhost:4566/_floci/health
docker compose logs --tail 100 floci
```

Confira:

- container em execução;
- porta publicada;
- endpoint correto;
- firewall local;
- proxy corporativo;
- serviço saudável.

Em ambientes com proxy, garanta que `localhost` não seja enviado ao proxy:

```bash
export NO_PROXY=localhost,127.0.0.1
export no_proxy=localhost,127.0.0.1
```

### 3. A AWS CLI tenta acessar a AWS real

Confirme:

```bash
echo "$AWS_ENDPOINT_URL"
```

Saída esperada:

```text
http://localhost:4566
```

Mantenha em todos os comandos:

```text
--endpoint-url "$AWS_ENDPOINT_URL"
```

Se houver dúvida, valide a conta:

```bash
aws sts get-caller-identity \
  --query Account \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

O resultado deste laboratório deve ser:

```text
000000000000
```

### 4. `Unable to locate credentials`

Exporte:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

Valide sem mostrar o segredo:

```bash
env | grep -E '^AWS_(ACCESS_KEY_ID|DEFAULT_REGION|ENDPOINT_URL)='
```

### 5. O JSON digitado no terminal está inválido

Valide antes:

```bash
printf '%s\n' "$MESSAGE_BODY" | jq .
```

Para documentos maiores, grave o JSON em um arquivo e use:

```bash
jq . arquivo.json
```

Erros comuns:

- aspas simples e duplas misturadas;
- vírgula após o último campo;
- variável Shell não expandida;
- barra invertida ausente;
- quebra de linha dentro de string.

### 6. `jq: command not found`

Em distribuições baseadas em RHEL:

```bash
sudo dnf install -y jq
```

Em Debian ou Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y jq
```

Se não puder instalar pacotes, execute o `curl` sem o filtro:

```bash
curl -fsS http://localhost:4566/_floci/health
printf '\n'
```

### 7. O bucket já existe

Valide:

```bash
aws s3api head-bucket \
  --bucket "$LAB_BUCKET" \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Se o comando terminar com código zero, prossiga para as etapas de objetos. Para começar do zero, remova os objetos e depois o bucket.

### 8. A tabela já existe

Consulte:

```bash
aws dynamodb describe-table \
  --table-name "$LAB_TABLE" \
  --query 'Table.TableStatus' \
  --output text \
  --endpoint-url "$AWS_ENDPOINT_URL"
```

Se retornar `ACTIVE`, não execute novamente `create-table`.

### 9. A fila já existe

Recupere a URL:

```bash
QUEUE_URL=$(
  aws sqs get-queue-url \
    --queue-name "$LAB_QUEUE" \
    --query QueueUrl \
    --output text \
    --endpoint-url "$AWS_ENDPOINT_URL"
)

export QUEUE_URL
echo "$QUEUE_URL"
```

### 10. A variável `QUEUE_URL` foi perdida

Isso ocorre ao abrir um novo terminal. Recupere com o comando da seção anterior.

### 11. O SQS retorna uma URL com hostname incorreto

Neste tutorial, a AWS CLI é executada diretamente no host. A URL esperada utiliza:

```text
localhost
```

Não defina `FLOCI_HOSTNAME: floci` nesse cenário.

Quando a aplicação cliente roda em outro container da mesma rede, `localhost` apontaria para o próprio container cliente. Nesse caso:

```yaml
environment:
  FLOCI_HOSTNAME: floci
```

E o cliente deve utilizar:

```text
http://floci:4566
```

### 12. `ReceiptHandle` vazio

Confira o arquivo:

```bash
jq . recebimento.json
```

Valide se existe:

```bash
jq -e '.Messages[0].ReceiptHandle' recebimento.json
```

Se não houver `Messages`:

- a fila pode estar vazia;
- a mensagem pode estar temporariamente invisível;
- outro consumidor pode tê-la recebido;
- a variável `QUEUE_URL` pode apontar para outra fila.

Consulte sem consumir:

```bash
curl -fsS -G \
  "${AWS_ENDPOINT_URL}/_aws/sqs/messages" \
  --data-urlencode "QueueUrl=${QUEUE_URL}" \
  | jq .
```

### 13. A mensagem reaparece

Receber não significa excluir. Se o consumidor não executar `DeleteMessage`, a mensagem poderá reaparecer quando terminar o `VisibilityTimeout`.

O fluxo correto é:

1. receber;
2. processar;
3. validar;
4. excluir com o `ReceiptHandle`.

### 14. Os recursos desapareceram depois do `down`

Verifique se foi utilizado:

```bash
docker compose down -v
```

O parâmetro `-v` remove o volume e, portanto, o estado persistente.

Confira a configuração:

```bash
docker compose config | sed -n '/volumes:/,$p'
```

E os mounts:

```bash
docker inspect floci \
  --format '{{json .Mounts}}' \
  | jq .
```

### 15. O container reinicia continuamente

```bash
docker compose ps -a
docker compose logs --tail 200 floci
docker inspect floci \
  --format 'exit={{.State.ExitCode}} error={{.State.Error}}'
```

Confira:

- imagem compatível com a arquitetura do host;
- espaço em disco;
- memória disponível;
- sintaxe das variáveis;
- permissão do volume;
- conflito de porta.

### 16. O healthcheck continua `starting`

```bash
docker inspect floci \
  --format '{{range .State.Health.Log}}{{println .Start .ExitCode .Output}}{{end}}'
```

Esse comando apresenta o histórico das tentativas do healthcheck.

Teste manualmente dentro do container:

```bash
docker exec floci \
  curl -fsS http://localhost:4566/_floci/health
```

### 17. Ver mais detalhes de uma chamada da AWS CLI

Adicione `--debug` ao comando:

```bash
aws s3api list-buckets \
  --endpoint-url "$AWS_ENDPOINT_URL" \
  --debug
```

Use essa saída somente para diagnóstico local. Ela é extensa e pode conter cabeçalhos e informações que não devem ser publicadas sem revisão.

## Checklist final do laboratório

Antes de considerar o projeto concluído, confirme:

- [ ] `docker compose config -q` termina com código zero;
- [ ] container `floci` está `running`;
- [ ] healthcheck está `healthy`;
- [ ] endpoint retorna HTTP `200`;
- [ ] S3, DynamoDB e SQS aparecem como `running`;
- [ ] STS retorna a conta `000000000000`;
- [ ] bucket `lab-certificacoes-floci` existe;
- [ ] objeto `materiais/estudo.txt` existe;
- [ ] objeto baixado é idêntico ao original;
- [ ] tabela `Estudos` está `ACTIVE`;
- [ ] item `projeto-001` está com status `validado`;
- [ ] fila `fila-estudos` existe;
- [ ] mensagem foi recebida e excluída com `ReceiptHandle`;
- [ ] fluxo integrado validou S3 e DynamoDB a partir de um evento;
- [ ] recursos permaneceram após a recriação do container;
- [ ] você escolheu conscientemente entre preservar ou remover o volume.

## Relação com estudos e certificações

Este laboratório permite praticar conceitos recorrentes em trilhas de cloud e DevOps:

### S3

- object storage;
- buckets e objects;
- keys e prefixes;
- metadados;
- upload e download;
- integridade;
- diferença entre comandos `s3` e `s3api`.

### DynamoDB

- banco NoSQL;
- partition key;
- modelagem orientada a acesso;
- tipos de atributos;
- leitura por chave;
- diferença entre `Query` e `Scan`;
- atualização condicional;
- projeção de atributos.

### SQS

- desacoplamento;
- processamento assíncrono;
- produtor e consumidor;
- visibility timeout;
- long polling;
- entrega pelo menos uma vez;
- confirmação por exclusão;
- `ReceiptHandle`.

### Docker e operação

- infraestrutura declarativa com Compose;
- healthcheck;
- logs;
- volumes;
- persistência;
- ciclo de vida do container;
- troubleshooting orientado por evidências.

## Conclusão

Com um único container, um endpoint local e a AWS CLI, construímos um laboratório reproduzível para praticar S3, DynamoDB e SQS sem depender de uma conta AWS.

Mais importante do que apenas executar os comandos foi validar cada etapa:

- verificamos a saúde do emulador;
- confirmamos que as requisições estavam indo para o endpoint local;
- validamos objetos e metadados no S3;
- consultamos e atualizamos um item no DynamoDB;
- acompanhamos o ciclo completo de uma mensagem no SQS;
- integramos os três serviços;
- removemos e recriamos o container;
- comprovamos que o estado permaneceu no volume.

Esse tipo de prática ajuda a transformar conceitos abstratos em comportamento observável. O mesmo raciocínio pode ser reaproveitado em ambientes reais: criar, verificar, registrar evidências, testar falhas, validar persistência e remover recursos conscientemente.

## Referências

- [Repositório oficial do Floci](https://github.com/floci-io/floci)
- [Instalação do Floci](https://floci.io/floci/getting-started/installation/)
- [Execução com Docker Compose](https://floci.io/floci/configuration/docker-compose/)
- [Configuração da AWS CLI e dos SDKs](https://floci.io/floci/getting-started/aws-setup/)
- [Modos de armazenamento](https://floci.io/floci/configuration/storage/)
- [Serviço S3 no Floci](https://floci.io/floci/services/s3/)
- [Serviço DynamoDB no Floci](https://floci.io/floci/services/dynamodb/)
- [Serviço SQS no Floci](https://floci.io/floci/services/sqs/)
- [Referência da AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/)
