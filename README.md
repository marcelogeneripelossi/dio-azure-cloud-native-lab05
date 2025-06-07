# dio-azure-cloud-native-lab05
Desafio Microsoft Azure Cloud Native - Criando um Serviço Autenticador de Boletos
---
## Serviço Autenticador de Boletos na Azure com Recursos Gratuitos
Este repositório documenta o processo de criação de um serviço básico de autenticação de boletos na Microsoft Azure, utilizando recursos gratuitos. O serviço consistirá em duas APIs (uma para gerar o código de barras do boleto e outra para validá-lo), implementadas como Azure Functions em C#/.NET, e será gerenciado e protegido por um API Gateway (Azure API Management) com autenticação Azure Active Directory. O Azure Service Bus será configurado para possíveis cenários de desacoplamento e processamento assíncrono.

### Sumário 
- Geral do Projeto
- Pré-requisitos
- Configuração do Azure Active Directory (Azure AD)
- Criação da Web App (API Backend - Azure Functions C#)
  - Criação do Resource Group
  - Criação do Azure Functions App
  - Implementando as Azure Functions em C#
  - Preparando e Implantando as Functions
- Configuração do Azure Service Bus
  - Criação do Namespace do Service Bus
  - Criação de Filas (Queues)
  - Obtenção das Shared Access Policies
- Configuração do Azure API Management (APIM)
  - Importando as Azure Functions como API
  - Configuração de Políticas no APIM
    - Configuração de CORS
    - Validação de JWT (Autenticação Bearer Token)
    - Configuração de Chave de Assinatura (Subscription Key)
- Testando a API com Postman/cURL
  - Passo 1: Obter o Token de Acesso (Bearer Token)
  - Passo 2: Chamar a API de Geração de Código de Barras
  - Passo 3: Chamar a API de Validação de Código de Barras
- Considerações Importantes sobre Recursos Gratuitos

## 1. Visão Geral do Projeto
Este guia demonstra como criar uma API RESTful para geração e validação de códigos de barras de boletos, utilizando Azure Functions (C#) para o backend, Azure Service Bus para mensageria (opcional para desacoplamento) e Azure API Management (APIM) como gateway de API, protegido por Azure Active Directory (Azure AD).

- Componentes Principais:
  - Azure Active Directory (Azure AD): Provedor de identidade para emitir Tokens JWT. Assume-se que um registro de aplicativo cliente já está configurado (conforme o guia anterior).
  - Azure Functions: Serviço de computação serverless para hospedar as APIs barcode-generate e barcode-validate.
  - Azure Service Bus: Serviço de mensageria para desacoplar a geração/validação de boletos de outros processos (opcional para o fluxo básico HTTP direto, mas útil para escalabilidade).
  - Azure API Management (APIM): Atua como o API Gateway, gerenciando o tráfego, aplicando políticas de segurança (autenticação JWT, chaves de assinatura, CORS) e roteando as requisições para as Azure Functions.

## 2. Pré-requisitos
- Conta Azure: Uma conta ativa na Microsoft Azure. Se não tiver, crie uma conta gratuita.
- .NET SDK: Instalado em sua máquina local (compatível com a versão do .NET das Functions).
- Azure Functions Core Tools: Para desenvolvimento e teste local de Azure Functions.
- Visual Studio Code (VS Code) com extensões:Azure FunctionsC#Azure Account
- Git: Para controle de versão e implantação.
- Postman ou cURL: Para testar a API.

## 3. Configuração do Azure Active Directory (Azure AD)
Para autenticação via Azure AD, você precisará de um Registro de Aplicativo Cliente que solicitará o token de acesso.

Referência: Por favor, siga as instruções detalhadas na Seção 3 (Configuração do Azure Active Directory (Azure AD)) do guia api-pagamento-azure-readme em seu repositório. Você precisará do "Application (client) ID" e do "Directory (tenant) ID" que foram gerados lá.

## 4. Criação da Web App (API Backend - Azure Functions C#)

### 4.1. Criação do Resource Group
Se você já possui um grupo de recursos, pode utilizá-lo. Caso contrário, crie um novo para organizar seus recursos.
- Acesse o [Portal Azure](https://portal.azure.com/).
- No menu esquerdo, clique em "Resource groups" ou use a barra de pesquisa.
- Clique em "+ Create" (Criar).
- Subscription: Selecione sua assinatura.
- Resource group name: Digite um nome para seu grupo de recursos, por exemplo, boleto-auth-rg.
- Region: Escolha a região mais próxima de você (ex: "Brazil South").
- Clique em "Review + create" e depois em "Create".
## 4.2. Criação do Azure Functions App
- No Portal Azure, na barra de pesquisa superior, digite "Function App" e selecione "Function App".
- Clique em "+ Create" (Criar).
- Basics (Noções básicas):
  - Subscription: Sua assinatura.
  - Resource Group: Selecione boleto-auth-rg.
  - Function App name: boleto-authenticator-api (um nome único globalmente).
  - Publish: "Code" (Código).
  - Runtime stack: ".NET" (ex: ".NET 6 (LTS)" ou ".NET 8 (LTS)").
  - Version: Selecione a versão do .NET correspondente.
  - Operating System: "Windows" (para .NET geralmente é Windows, mas Linux também é opção).
  - Region: A mesma do seu Resource Group (Brazil South).
  - Hosting Plan:
    - Plan type: "Consumption (Serverless)" (Consumo - Serverless). Este plano se qualifica para o nível gratuito de Azure Functions.
- Clique em "Review + create" e depois em "Create"
- .Após a criação, anote a URL do seu Function App (ex: https://boleto-authenticator-api.azurewebsites.net).
## 4.3. Implementando as Azure Functions em C#
Você pode usar o Visual Studio Code com as extensões Azure Functions e C# para criar e desenvolver seu projeto.
- Crie um novo projeto de Azure Functions (C#):
  - No VS Code, abra a paleta de comandos (Ctrl+Shift+P ou Cmd+Shift+P), digite "Azure Functions: Create New Project...".
  - Escolha uma pasta vazia.
  - Selecione "C#".Escolha um template "HTTP trigger".
  - Dê o nome à sua primeira função: BarcodeGenerate.
  - Namespace: BoletoAuthenticator.Functions.
  - Access right: Anonymous (a segurança será tratada pelo APIM).
- Referenciando o Código Base:
  - O repositório https://github.com/digitalinnovationone/Microsoft_Application_Platform/tree/main/Labs/Lab05 fornece um exemplo de estrutura de projeto .NET. Você pode usá-lo como um ponto de partida para a estrutura do seu projeto Azure Functions.
- Implementando barcode-generate:
  - No arquivo BarcodeGenerate.cs (ou similar), implemente a lógica para gerar um código de barras de boleto. Este endpoint deve receber dados de entrada (ex: valor, data de vencimento, dados do pagador) e retornar o código de barras gerado.

````
// BarcodeGenerate.cs (Exemplo conceitual)
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace BoletoAuthenticator.Functions
{
    public static class BarcodeGenerate
    {
        [FunctionName("barcode-generate")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "barcode/generate")] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request to generate barcode.");

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);

            // --- Início da Lógica de Geração de Código de Barras do Boleto ---
            // Esta é a parte que você precisará implementar.
            // Exemplo: Simplesmente concatenando dados para demonstração.
            string valor = data?.valor;
            string vencimento = data?.vencimento; // Ex: "YYYY-MM-DD"

            if (string.IsNullOrEmpty(valor) || string.IsNullOrEmpty(vencimento))
            {
                return new BadRequestObjectResult("Por favor, forneça 'valor' e 'vencimento' no corpo da requisição.");
            }

            // Lógica real de geração de código de barras de boleto seria aqui.
            // Isso pode envolver bibliotecas específicas ou algoritmos.
            string generatedBarcode = $"BOLETO_CODE_{valor.Replace(".", "").Replace(",", "")}_{vencimento.Replace("-", "")}_{System.Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
            // --- Fim da Lógica de Geração de Código de Barras do Boleto ---

            return new OkObjectResult(new {
                barcode = generatedBarcode,
                message = "Código de barras gerado com sucesso (implemente a lógica real aqui)."
            });
        }
    }
}
````
- Implementando barcode-validate:
  - Crie uma nova função HTTP Trigger (no mesmo projeto ou em um novo arquivo .cs) chamada BarcodeValidate. Esta função receberá um código de barras e retornará se ele é válido ou não.
````
// BarcodeValidate.cs (Exemplo conceitual)
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace BoletoAuthenticator.Functions
{
    public static class BarcodeValidate
    {
        [FunctionName("barcode-validate")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "barcode/validate")] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request to validate barcode.");

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);

            string barcodeToValidate = data?.barcode;

            if (string.IsNullOrEmpty(barcodeToValidate))
            {
                return new BadRequestObjectResult("Por favor, forneça 'barcode' no corpo da requisição.");
            }

            // --- Início da Lógica de Validação de Código de Barras do Boleto ---
            // Esta é a parte que você precisará implementar.
            // Exemplo: Simples validação de formato ou checksum.
            bool isValid = barcodeToValidate.StartsWith("BOLETO_CODE_") && barcodeToValidate.Length > 20; // Lógica de validação mock

            // Lógica real de validação de código de barras de boleto seria aqui.
            // Isso pode envolver cálculo de dígito verificador ou consulta a um padrão.
            // --- Fim da Lógica de Validação de Código de Barras do Boleto ---

            return new OkObjectResult(new {
                barcode = barcodeToValidate,
                isValid = isValid,
                message = "Validação do código de barras processada (implemente a lógica real aqui)."
            });
        }
    }
}
````
### 4.4. Preparando e Implantando as Functions
- Publicar o Projeto: No VS Code, clique no ícone do Azure na barra lateral.
- Em "Functions", selecione sua assinatura e encontre sua "Function App" (boleto-authenticator-api).
- Clique no ícone de "Deploy to Function App..." (seta para cima).
- Siga as instruções para selecionar a pasta do seu projeto Functions e implantar.

Após a implantação, suas funções estarão disponíveis em URLs como:
- https://boleto-authenticator-api.azurewebsites.net/api/barcode/generate
- https://boleto-authenticator-api.azurewebsites.net/api/barcode/validate

Teste-as diretamente antes de prosseguir para o APIM.

## 5. Configuração do Azure Service Bus
O Azure Service Bus é um serviço de mensageria para desacoplar componentes da sua aplicação. Embora não seja estritamente necessário para o fluxo HTTP direto de geração/validação, ele é excelente para processamento assíncrono (ex: enviar o código de barras gerado para uma fila para outro sistema processar, ou receber solicitações de validação em massa).
### 5.1 Criação do Namespace do Service Bus
- No Portal Azure, na barra de pesquisa superior, digite "Service Bus" e selecione "Service Bus".
- Clique em "+ Create" (Criar).
- Basics (Noções básicas):
  - Subscription: Sua assinatura.
  - Resource Group: Selecione boleto-auth-rg.
  - Namespace name: boleto-auth-sb (um nome único globalmente).
  - Location: A mesma dos seus outros recursos (Brazil South).Pricing tier: Escolha "Basic" (Básico) para se qualificar para recursos gratuitos (o plano Basic oferece até 100 entidades de mensageria).
- Clique em "Review + create" e depois em "Create".

### 5.2. Criação de Filas (Queues)
Vamos criar duas filas de exemplo para demonstrar o uso do Service Bus.
- Após o namespace do Service Bus ser criado, vá para o recurso.
- No menu esquerdo, clique em "Queues" (Filas).
- Clique em "+ Queue" (Adicionar Fila).
  - Queue name: barcode-generate-requests (para solicitações de geração assíncronas).
  - Deixe as configurações padrão para este exemplo.
- Clique em "Create" (Criar).
- Repita o processo para criar outra fila:
  - Queue name: barcode-validation-results (para resultados de validação ou processamento posterior).

## 5.3. Obtenção das Shared Access Policies
Você precisará da Connection String para que suas Azure Functions (ou outros aplicativos) possam interagir com o Service Bus.
- No seu namespace do Service Bus, no menu esquerdo, clique em "Shared access policies" (Políticas de acesso compartilhado).
- Clique em "RootManageSharedAccessKey".
- Copie a "Primary Connection String" (Cadeia de Conexão Primária). Você a usaria nas configurações do seu Function App (Application settings) se fosse integrar as Functions com o Service Bus (via input/output bindings).

## 6. Configuração do Azure API Management (APIM)
O API Management atuará como seu gateway, protegendo e gerenciando sua API de boletos.

### 6.1. Importando as Azure Functions como API
Se você já possui uma instância APIM configurada, pode utilizá-la. Caso contrário, crie uma nova com a camada "Developer" (conforme a Seção 5.1 do guia api-pagamento-azure-readme).
- Após a instância do APIM ser criada/selecionada, vá para o recurso.
- No menu esquerdo, clique em "APIs".
- Clique em "+ Add API" (Adicionar API).
- Selecione "Function App".
  - Function App: Clique em "Browse" (Procurar) e selecione seu boleto-authenticator-api Function App.
- Create from Function App (Criar do Function App):
  - Display name: API Autenticador Boletos
  - Name: api-boleto-auth
  - Description: API para gerar e validar códigos de barras de boletos
  - Products: Selecione "Unlimited" (Ilimitado) para acesso público por padrão.
  - URL suffix: boleto (isso fará com que a URL da API no APIM seja algo como https://api-pagamento-apim.azure-api.net/boleto/...).
  - Version identifier: v1
  - Operations: Selecione as operações barcode-generate e barcode-validate que você criou.
- Clique em "Create" (Criar).

### 6.2. Configuração de Políticas no APIM
As políticas são cruciais para aplicar lógica e segurança às suas APIs no APIM.
- Na sua instância APIM, vá em "APIs" e selecione a API Autenticador Boletos.
- Clique na aba "Design" (Design).
- Clique em "All operations" (Todas as operações).
- No painel à direita, na seção "Inbound processing", clique no ícone <code/> (Editor de política).

#### 6.2.1. Configuração de CORS
Adicione a política <cors> dentro da seção <inbound>:
````
<policies>
    <inbound>
        <base />
        <cors>
            <allowed-origins>
                <origin>*</origin> <!-- Permite qualquer origem para desenvolvimento -->
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
            </allowed-methods>
            <allowed-headers>
                <header>*</header>
            </allowed-headers>
            <expose-headers>
                <header>Content-Disposition</header>
            </expose-headers>
            <allow-credentials>false</allow-credentials>
        </cors>
        <!-- Outras políticas de inbound virão aqui -->
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
````
Nota: Para produção, substitua <origin>*</origin> por domínios específicos para segurança.

#### 6.2.2. Validação de JWT (Autenticação Bearer Token)
Esta política validará o token JWT (Bearer Token) emitido pelo Azure AD.

Adicione a política <validate-jwt> dentro da seção <inbound>, após a política <cors> e antes de qualquer outra lógica de acesso.
````
<policies>
    <inbound>
        <base />
        <cors>
            <!-- ... suas configurações CORS ... -->
        </cors>
        <!-- Política de Validação de JWT -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Token não tem permissão ou é inválido.">
            <openid-config url="https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0/.well-known/openid-configuration" />
            <audiences>
                <audience>YOUR_APPLICATION_CLIENT_ID</audience>
            </audiences>
            <issuers>
                <issuer>https://sts.windows.net/YOUR_TENANT_ID/</issuer>
            </issuers>
        </validate-jwt>
        <!-- Outras políticas de inbound virão aqui -->
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
````
- Substitua:
  - YOUR_TENANT_ID: Pelo Directory (tenant) ID que você anotou do registro de aplicativo no Azure AD (da Seção 3).
  - YOUR_APPLICATION_CLIENT_ID: Pelo Application (client) ID do seu registro de aplicativo no Azure AD (da Seção 3).
  
#### 6.2.3. Configuração de Chave de Assinatura (Subscription Key)
O APIM também pode exigir uma chave de assinatura para acesso, além do JWT.
- Na sua instância APIM, vá em "Subscriptions" (Assinaturas) no menu esquerdo.
- Você verá uma assinatura "Built-in all-access subscription" ou "Unlimited". Copie uma das "Primary key" (Chave primária) ou "Secondary key" (Chave secundária). Esta é a x-api-key.
- O APIM, por padrão, já requer a x-api-key no cabeçalho Ocp-Apim-Subscription-Key. Não é necessário adicionar uma política explícita a menos que você queira personalizar o comportamento ou usar um nome de cabeçalho diferente.

## 7. Testando a API com Postman/cURL
Agora que tudo está configurado, vamos testar o fluxo completo.

### 7.1. Passo 1: Obter o Token de Acesso (Bearer Token)
Você precisará usar o endpoint OAuth 2.0 do Azure AD para obter um token usando o Client Credentials Flow.
- URL do Token Endpoint: https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/token
  - Substitua YOUR_TENANT_ID pelo seu Directory (tenant) ID.
Exemplo cURL (GET do Token):
````
curl -X POST "https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/token" \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "client_id=YOUR_APPLICATION_CLIENT_ID" \
-d "client_secret=YOUR_CLIENT_SECRET_VALUE" \
-d "grant_type=client_credentials" \
-d "scope=https://graph.microsoft.com/.default"
````
- Substitua:
  - YOUR_TENANT_ID: Seu Directory (tenant) ID.
  - YOUR_APPLICATION_CLIENT_ID: Seu Application (client) ID.
  - YOUR_CLIENT_SECRET_VALUE: O valor do segredo do cliente que você copiou.
  
A resposta será um JSON contendo o access_token. Copie este valor.

### 7.2. Passo 2: Chamar a API de Geração de Código de Barras
Use o token de acesso e a x-api-key para chamar a API via APIM.
- URL da API via APIM (Generate): https://api-pagamento-apim.azure-api.net/boleto/barcode/generate
  - Substitua api-pagamento-apim pelo nome da sua instância APIM.

Exemplo cURL (POST para barcode-generate):
````
curl -X POST "https://api-pagamento-apim.azure-api.net/boleto/barcode/generate" \
-H "Authorization: Bearer SEU_TOKEN_DE_ACESSO" \
-H "Ocp-Apim-Subscription-Key: SUA_CHAVE_DE_ASSINATURA_APIM" \
-H "Content-Type: application/json" \
-d '{ "valor": "100.50", "vencimento": "2025-12-31" }'
````
- Substitua:
  - SEU_TOKEN_DE_ACESSO: O access_token obtido no Passo 1.
  - SUA_CHAVE_DE_ASSINATURA_APIM: A chave primária ou secundária da sua assinatura APIM.

Você deve receber uma resposta JSON com o código de barras gerado (mock neste exemplo).

### 7.3. Passo 3: Chamar a API de Validação de Código de Barras
- URL da API via APIM (Validate): https://api-pagamento-apim.azure-api.net/boleto/barcode/validate
  - Substitua api-pagamento-apim pelo nome da sua instância APIM.

Exemplo cURL (POST para barcode-validate):
````
curl -X POST "https://api-pagamento-apim.azure-api.net/boleto/barcode/validate" \
-H "Authorization: Bearer SEU_TOKEN_DE_ACESSO" \
-H "Ocp-Apim-Subscription-Key: SUA_CHAVE_DE_ASSINATURA_APIM" \
-H "Content-Type: application/json" \
-d '{ "barcode": "BOLETO_CODE_10050_20251231_ABCD1234" }'
````
- Substitua:
  - SEU_TOKEN_DE_ACESSO: O access_token obtido no Passo 1.
  - SUA_CHAVE_DE_ASSINATURA_APIM: A chave primária ou secundária da sua assinatura APIM.
  - BOLETO_CODE_10050_20251231_ABCD1234: Use o código de barras que foi retornado pela API barcode-generate ou um código de barras de teste.

Você deve receber uma resposta JSON indicando a validade do código de barras.

## 8. Considerações Importantes sobre Recursos Gratuitos
- Azure Functions (Consumo): O plano "Consumption" é gratuito para um número significativo de execuções e uso de memória. No entanto, ele pode ter latência para "cold starts" (primeira execução após inatividade).
- Azure Service Bus (Básico): O plano "Basic" é muito mais barato que o "Standard" e "Premium" e geralmente se qualifica para os créditos gratuitos iniciais, mas possui limites no número de entidades e no tamanho das mensagens.
- Azure API Management (Desenvolvedor): A camada "Developer" do Azure API Management é excelente para testes e desenvolvimento, mas não possui um SLA (Service Level Agreement) e não é adequada para produção. Além disso, o Private Endpoint para o gateway APIM é um recurso da camada Premium, não disponível nas camadas gratuitas ou de desenvolvedor. Isso significa que o endpoint da sua API no APIM será público.
- Custos: Monitore seu uso no Portal Azure. Embora os recursos sejam gratuitos para este exemplo, o excesso de uso ou a criação de outros recursos podem gerar custos.
- Segurança: Para ambientes de produção, é crucial fortalecer a segurança (ex: usar certificados gerenciados pelo Azure, implementar políticas mais complexas no APIM, monitoramento aprofundado, integração com VNet para backends privados, etc.).

