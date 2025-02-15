﻿Publicando aplicativo web no Azure, utilizando Identity, SQLite e GitHub Actions.

Sumário

1. [Introdução](#introducao)
2. [Instalações.](#instalacoes)
3. [Criando repositório remoto.](#criandorepositorioremoto)
4. [Repositório local e criação do app.](#repositoriolocalecriacaodoapp)
5. [Gerando as páginas do Identity.](#gerandoaspaginasdoidentity)
7. [Modelo de usuário.](#modelodeusuario)
8. [Personalizando a página de registro do usuário.](#personalizandoapaginaderegistrodousuario)
9. [Email service.](#emailservice)
10. [Microsoft Azure](#microsoftazure)
11. [Configurando Azure.](#configurandoazure)
12. [Workflow, GitHub Actions.](#workflowgithubactions)
9. [Referências.](#referencias)

*******

<div id='introducao'></div> 

### Introdução

Neste artigo vou falar sobre uma forma simples e prática de implementar e publicar um aplicativo web no Azure. Este app será criado através de comandos CLIs: .NET, Git, GitHub, e Azure. E nele, vamos implementar autenticação e autorização com Identity, SQLite como banco de dados, e o processo de deployment será feito em um App Service no Azure, automatizado pelo GitHub Actions, em uma máquina Linux.

<div id='instalacoes'></div>

### Instalações

Para executar os passos que vou demonstrar neste artigo será necessário ter instaladas as CLIs: .NET, Git, GitHub, e Azure. Eu estou usando Windows Terminal, mas estes comandos podem ser executados em um terminal de sua escolha, em uma máquina Windows, Linux ou Mac. Para editar o nosso código, também vamos precisar de uma IDE, estou utilizando Rider, mas podem ser VS Code, Visual Studio, etc.

<div id='criandorepositorioremoto'></div>

### Criando repositório remoto.

Vamos dar início à nossa jornada criando o nosso repositório remoto no GitHub. No browser, https://github.com, vamos acessar a página dos nossos repositórios, clicando em "New", seremos redirecionados à página onde vamos dar um nome ao nosso repositório, e sem adicionar o arquivo .gitignore, vamos concluir clicando no botão "Create Repository", então vamos copiar a Url do repo, disponibilizado pelo GitHub, e vamos para a próxima etapa.

![New_repo](images/newrepo.png?raw=true)

<div id='repositoriolocalecriacaodoapp'></div> 

### Criação do app e repositório local.

Na nossa máquina é recomendado que o repositório local seja armazenado em uma pasta com um caminho curto, como C:/dev/. Nela vamos criar um aplicativo do tipo `webapp` vazio, que vou dar o nome de Article, depois disso, navegando para pasta do app, vamos criar o arquivo .gitignore, vamos iniciar os arquivos .git, renomear a branch principal para "main", e adicionar uma referência ao nosso repositório remoto com os seguintes comandos de terminal.

```
cd .\dev\
dotnet new webapp -o Article
cd .\Article\
dotnet new gitignore
git init
git branch -M main
git remote add origin https://github.com/crisnordev/Article.git
```

Lembrando que no último comando você deve trocar a Url, pela referência ao seu repositório remoto.



<div id='gerandoaspaginasdoidentity'></div>

### Gerando as páginas do Identity.

Para gerar as páginas do Identity, biblioteca .NET que vai fornecer suporte para autenticação do usuário da nossa aplicação, utilizaremos o scaffolder do Identity, pela linha de comando. Para isso, devemos instalar a ferramenta de geração de código do ASP.NET, o pacote para geração de código do Visual Studio, e mais alguns pacotes:

```
dotnet tool install -g dotnet-aspnet-codegenerator
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Identity.UI
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

Agora, vamos chamar as opções do scaffolder do Identity, e se por acaso retornar alguma mensagem de erro verifique as versões dos pacotes instalados no nosso app, que devem ser compatíveis. Na sequência vamos executar o scaffolder, passando como parâmetro, o nome da classe do nosso usuário, quais as páginas que deverão ser criadas, e por fim o SqLite.

```
dotnet aspnet-codegenerator identity --help
dotnet aspnet-codegenerator identity --userClass ApplicationUser --files "Account.Register;Account.Login;Account.Logout;Account.RegisterConfirmation" --useSqLite
```

Para disponibilizar os botões do login nas páginas da nossa aplicação, no arquivo Pages/Shared/_Layout.cshtml, depois da `</ul>` que fecha a declaração dos links das páginas do nosso app, devemos adicionar a linha de código `<partial name="_LoginPartial" />`. E caso queira que o scaffolder gere todas as páginas do identity, basta remover o parâmetro ``--files`` que está recebendo os nomes dos arquivos que especifiquei para serem criados.

<div id='modelodeusuario'></div>

### Modelo de usuário.

A fim de oferecer uma experiência um mais personalizada ao usuário da nossa aplicação, junto com as páginas do Identity nós criamos a classe para o modelo do usuário. Porém, como sugestão, eu vou modificar o local da pasta Data, e também vou criar a pasta Models, na raiz do nosso app, onde vou colocar a classe "ApplicationUser". Note que esta classe está indicando ser do tipo IdentityUser através de herança, e também vamos alterar o tipo que será atribuído ao Id do usuário para Guid (padrão string). Além disso, vamos sobrescrever algumas propriedades, para que o cadastro do usuário tenha o comportamento que nós queremos.

![ApplicationUser](images/applicationuser.png?raw=true)

Lembrando que caso você tenha modificado o local onde estão dispostos DataContext e ApplicationUser, os namespaces deverão ser corrigidos, assim como as referências "using" das classes que estão utilizando eles. Note que estamos sobrescrevendo UserName, para poder receber a entrada de um valor do tipo string (por padrão recebe o valor do e-mail), que vai nos possibilitar a utilização de um nome de usuário personalizado, o que nos obriga a alterar a propriedade NormalizedUserName, que vai receber o valor de UserName convertido em letras maiúsculas.

Note também que na classe em que manipulamos o DataContext, devemos adicionar uma referência ao nosso ApplicationUser, e como estamos alterando o tipo do Id, também devemos referenciar IdentityRole, informando que os dois devem estar utilizando o tipo Guid no Id.

<div id='personalizandoapaginaderegistrodousuario'></div>

### Personalizando a página de registro do usuário.

Agora nós vamos alterar Register.cshtml.cs, e Register.cshtml, para receber a entrada personalizada do UserName. Para isso, em Areas/Identity/Pages/Account, em Register.cshtml.cs, vamos até a classe ImputModel, e vamos adicionar a propriedade UserName.

```csharp
[Required]
[Display(Name = "User Name")]
public string UserName { get; set; }
```

Também em Register.cshtml.cs, vamos alterar o método OnPostAsync(), na linha onde estamos atribuindo o valor para o nome do usuário, substituindo pela seguinte linha.

```csharp
await _userStore.SetUserNameAsync(user, Input.UserName, CancellationToken.None);
```

Agora, em Register.cshtml, na `<div>` que está recebendo o input do email, na primeira linha do `<input>` vamos alterar o componente "autocomplete" para `autocomplete="email"`, e antes dessa `<div>` vamos adicionar o código para receber o input do UserName.
```csharp
<div class="form-floating mb-3">
    <input asp-for="Input.UserName" class="form-control" autocomplete="username" aria-required="true" placeholder="User name"/>
    <label asp-for="Input.UserName">User Name</label>
    <span asp-validation-for="Input.UserName" class="text-danger"></span>
</div>
```
<div id='emailservice'></div>

### Email service.

Para configuração do serviço de e-mail, eu estou utilizando a API do SendGrid, adicionando o pacote nuget através do comando `dotnet add package SendGrid`. Você pode criar uma conta grátis, https://sendgrid.com/free/?source=sendgrid-csharp. Na sua conta, para possibilitar a configuração do serviço de e-mail na nossa aplicação, vamos precisar de uma chave da API do SendGrid. Para criar a chave, no portal do SendGrid você deve navegar para aba "Settings", "Api Keys", "Create API Key", adicionando o nome que será atribuido a essa chave, selecionando "Restricted Access", arrastando a barra "Mail Send" até "Full Access", e por fim "Create API Key". Você será redirecionado à página, que irá te mostrar a chave uma única vez, é recomendado copiar e armazenar essa chave em um local seguro, pois nós vamos precisar dela mais para frente, lembrando que essa chave é sua e ninguém mais pode ter acesso a ela.

Na nossa aplicação, criando a pasta Services, vamos adicionar a classe EmailSender, que vai receber os dados para o serviço de envio de e-mail.

```csharp
public class EmailSender : IEmailSender
{
    public async Task SendEmailAsync(string toEmail, string subject, string message)
    {
        if (string.IsNullOrEmpty(Configuration.SendGridKey.SendGridApiKey)) throw new Exception("Null SendGridKey");

        await Execute(Configuration.SendGridKey.SendGridApiKey, subject, message, toEmail);
    }

    public static async Task Execute(string apiKey, string subject, string message, string toEmail)
    {
        var client = new SendGridClient(apiKey);
        var msg = new SendGridMessage
        {
            From = new EmailAddress("email@email.com", "Cristiano Noronha"),
            Subject = subject,
            PlainTextContent = message,
            HtmlContent = message
        };
        msg.AddTo(new EmailAddress(toEmail));

        msg.SetClickTracking(false, false);
        await client.SendEmailAsync(msg);
    }
}
```
O Identity usa injeção de dependência da interface IEmailSender para instanciar o serviço de e-mail. Na classe EmailSender, que acabamos de criar, nós estamos implementando essa interface, porém para a nossa aplicação saber que nas injeções de dependência da interface IEmailSender, deverá ser utilizada a implementação que criamos no Program.cs, nós devemos resolver a dependência adicionando a linha de código `builder.Services.AddSingleton<IEmailSender, EmailSender>();`, onde estamos passando o tempo de vida da instância como Singleton, porque ela não vai ser modificada ao longo do tempo, e assim, sempre que o Identity precisar do IEmailSender, saberá que deve utilizar a implementação EmailSender.

Além disso, para receber a API Key do SendGrid, no arquivo appsettings.json, vamos adicionar uma seção com o nome usado na criação da chave, no SendGrid, para receber a API Key, que estará configurada no Azure. No caso eu nomeei como "My_SendGrid_Key".
```json
{
  "SendGridKey": {
    "SendGridApiKey": "My_SendGrid_Key"
  }
}
```

E, para receber essa chave na nossa aplicação vamos criar a classe Configuration.cs: 
```csharp
public class Configuration
{
    public static SendGridConfig SendGridKey { get; set; }
    
    public class SendGridConfig
    {
        public string SendGridApiKey { get; set; }
    }
}
```

E,  no Program.cs, vamos instanciá-la, pegando o valor da chave que estará entrando pelo appsettings.json, e atribuindo 
à propriedade SendGridKey, adicionando as seguintes linhas:

```csharp
var sendGridKey = new Configuration.SendGridConfig();
app.Configuration.GetSection("SendGridKey").Bind(sendGridKey);
Configuration.SendGridKey = sendGridKey;
```

<div id='microsoftazure'></div>

### Microsoft Azure.

Nós vamos fazer toda a parte do deployment do nosso app no Azure, plataforma de nuvem da Microsoft. Onde é possível criar uma conta grátis, acessando https://azure.microsoft.com/. Ao criar a conta, é gerado uma "Assinatura"(Subscription), "Azure Subscription 1". Com essa assinatura nós vamos criar um "Grupo de Recursos" (Resource Group), que também pode ser encontrado pela barra de busca, e navegando até a respectiva página, clicando no botão "+ Criar" (Create), onde selecionando a assinatura, dando um nome ao grupo de recursos, e selecionando a nossa região" (South America) Brazil South", vamos clicar no botão "Revisar + criar" (Review + create), em seguida "Criar" (Create).

Da mesma forma, vamos criar o "Serviços de Aplicativos" (App Service), navegando até a página, clicando no botão" + Criar", seremos redirecionados à pagina onde selecionando a nossa "Assinatura" (Subscription), o"Grupo de recursos" (Resource Group) que criamos, escolhendo um nome para a aplicação, que deve ser único pois será utilizado na Url, "Publicar" (Publish) como "Código", "Pilha de Runtime" (Runtime Stack) nós vamos selecionar a versão do .NET que estamos utilizando no nosso app, "Sistema Operacional" (Operating System) como "Linux", "Região" (Region)como "Brazil South", "Plano do Linux (Brazil South)" (Linux Plan) clicando no "Criar Novo" (Create New), vamos dar um nome ao plano Linux, que na sequência, clicando no botão "Alterar Tamanho" (Change Size), estaremos selecionando o plano "Gratuito F1, 1 GB de memória", então clicar em "Revisar + criar" (Review + create), e por fim "Criar" (Create).

<div id='configurandoazure'></div>

### Connection string, e configurações no Azure.

Quando criamos o app, por estarmos utilizando Sqlite, a connection string é gerada no appsettings.json como "Data Source=Article.db", que está indicando a raíz da nossa aplicação como local para criação do banco de dados, porém no Azure nós devemos modificar a connection string, indicando wwroot como local, caso contrário irá gerar uma Exception, porque não será possível acessar o banco.

Para configuração da connection string, no portal do Azure, navegando para "Serviços de Aplicativos", clicando no link que está com o nome que utilizamos para criar a Url, seremos redirecionados para a aba do nosso app, podemos parar a execução do app clicando em "Parar" (Stop), e, no menu lateral, vamos para as "Configurações", e clicando em + New connection string", no campo "Name" vamos colocar `DefaultConnection`, em "Value" `DataSource=wwwroot/app.db;Cache=Shared`, "Type" `Custom`, "Ok" parar criar. E dessa forma, no arquivo appsettings.json, a connection string ficará da seguinte forma:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "My_Connection_String"
  }
}
```
Lembrando que no Program.cs devemos utilizar o mesmo nome. 

```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
```

 E da mesma forma, ainda na página das configurações do nosso app, no portal do Azure, nós também vamos passar o valor da API key do SendGrid, clicando em "+ New application setting", o nome deverá ser o mesmo que utilizamos no appsettings.jso da nossa aplicação, porém devemos indicar qual seção, e como estamos em uma máquina Linux, para separar a seção e o item devemos usar "__" (2 X underline), `SendGridKey__SendGridApiKey`, o valor será a chave que criamos no SendGrid, `SG.HashGeradoNoPortalDoSendGrid`, lembrando que nunca devemos expor esse hash, "Ok" para criar.

<div id='workflowgithubactions'></div>

### Workflow, GitHub Actions.

O primeiro passo para a criação do workflow, que vai automatizar o processo de deploy da nossa aplicação, será feito no portal do Azure, navegando para "Serviços de aplicativos" (App Services), clicando no link do nosso app, no menu lateral selecionando "Centro de Implantação" (Deployment Center), vamos setar "Source" como "GitHub", que vai disponibilizar o link para autenticar nosso github, que será feito via Http, no próprio navegador, em seguida, selecionando o nosso usuário do GitHub em "Organization", "Repository" será o nome do repo da nossa aplicação no GitHub, "Branch" iremos selecionar "Main", assim toda mudança feita na branch `main` irá disparar as ações do GitHub Actions, para verificar o arquivo do workflow que vai ser gerado automaticamente no repositório remoto, clicamos no botão "Preview file", para fechar "Close", lembrando de salvar as alterações clicando em "Save".

Após finalizada a execução no Azure, navegando até a página do nosso repositório remoto, no GitHub, verificamos a criação da pasta .github/workflows, com o arquivo main_nomeDoApp.yml, contendo os passos que serão executados nas Actions. Então, vamos finalizar essa parte das configurações, navegando até a aba "Settings", no menu lateral seleciona "Actions", "General", e em "Actions permissions", selecionar "Allow all actions and reusable workflow", "Save". Em seguida, no menu lateral, "Secrets", "Actions", onde foi gerado o secret`AZUREAPPSERVICE_PUBLISHPROFILE_HashDoSecret`, referenciando o nosso AppService no Azure, e, vamos criar mais dois secrets.

Primeiro, uma referências à connection string, informando esse caminho às Actions go GitHub. Vamos clicar no botão "New repository secret", no campo"Name" vamos preencher com `AZURE_SQLITE_CONNECTION_STRING`, e no campo"Secret" com `DataSource=wwwroot/app.db;Cache=Shared`. 

A próxima secret, que vai receber o nome `AZURE_CREDENTIALS`, depende de um processo que executaremos no nosso terminal, usando Azure CLI, que vão disponibilizar as credenciais para autorizar a execução das Actions pelo GitHub Actions no nosso App Service do Azure. Então, no terminal, com A CLI do Azure instalada, e autenticada, nós vamos executar o seguinte comando `az ad sp create-for-rbac --name "NomeDoAplicativo" --role contributor --scopes /subscriptions/IdDaAssinatura/resourceGroups/NomeDoGrupoDeRecursos --sdk-auth`, lembrando de buscar no portal do Azure, na página "Serviços de aplicativos", e alterar no comando anterior o"NomeDoAplicativo", "IdDaAssinatura", "NomeDoGrupoDeRecursos". Este comando vai retornar um Json, o qual nós vamos copiar e colar como valor da "Secret".

Por fim, vamos atualizar o app no nosso repo local, executando um `git pull` no terminal, e vamos acessar o arquivo .yml do nosso workflow, e iremos adicionar as referências da connection string, das credenciais do Azure, os comandos para instalar as ferramentas EF Tools e executar um update database.

```yaml
name: Build and deploy ASP.Net Core app to Azure Web App - articleidentity

on:
  push:
    branches:
      - main
  workflow_dispatch:
    
env:
  ConnectionStrings__DefaultConnection: ${{ secrets.AZURE_SQLITE_CONNECTION_STRING }}

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Set up .NET Core
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '7.x'
          include-prerelease: true

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          
      - name: Install EF Tools
        run: dotnet tool install --global dotnet-ef    

      - name: Build with dotnet
        run: dotnet build --configuration Release
        
      - name: Update database
        run: dotnet ef database update  

      - name: dotnet publish
        run: dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/myapp

      - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v2
        with:
          name: .net-app
          path: ${{env.DOTNET_ROOT}}/myapp

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: 'Production'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
      - name: Download artifact from build job
        uses: actions/download-artifact@v2
        with:
          name: .net-app

      - name: Deploy to Azure Web App
        id: deploy-to-webapp
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'articleidentity'
          slot-name: 'Production'
          publish-profile: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_D2984E3B0F684422BDE0CB2E6B678773 }}
          package: .

```

Por último, vamos executar uma migration, `dotnet ef migrations add v1`. No arquivo appsettings.json vamos alterar temporariamente o valor da connection string `DataSource=wwwroot/app.db;Cache=Shared`, em seguida executando`dotnet ef database update`, lembrando de desfazer a alteração que acabamos de fazer na connection string,`My_Connection_String`. No portal do Azure, navegando para "Serviços de aplicativos" (App Services), clicando no link do nosso app, clicando em "Iniciar", para deixar a aplicação em execução. E para mandar as alterações, para o nosso repo remoto, vamos adicionar as alterações, realizar um commit, e um push, que irão disparar as Actions, que pode ser conferidos os detalhes no portal do GitHub, na aba "Actions".

```
git add .
git commit -m "Adicionando App.db, migrations, e alterações do .yml"
git push -u origin main

```

Assim, com o termino das ações das Actions, nós teremos um app, completo, com um processo de deployment automatizado, publicado, e funcional, que poderá ser testado clicando no link da Url do app, na página "Serviços de aplicativos" do portal do Azure.                                 

<div id='referencias'></div>

### Referências.

[Curso: Fundamentos do Azure, Git, GitHub e DevOps](https://balta.io/player/assistir/442da086-3cac-4d96-9332-cdab3797c01c)

[Introduction to Identity on ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-6.0&tabs=visual-studio)

[https://github.com/sendgrid/sendgrid-csharp#setup](https://github.com/sendgrid/sendgrid-csharp#setup)
