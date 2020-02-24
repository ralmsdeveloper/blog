﻿---
title: "Logs e Consultas LINQ to SQL"
comments: true
excerpt_separator: "Ler mais"
categories:
  - Dica
  - "Entity Framework Core"
toc: true
toc_label: "Começando"
---

![01]({{site.url}}{{site.baseurl}}/assets/images/topoefnew.jpg)

<center><strong>Fala pessoal, tudo bem?! 💚</strong></center>
<hr>

## Introdução
<div style="text-align: justify;">
Bom para começarmos nosso pequeno artigo, vamos falar um pouco sobre o <strong>"EU TER"</strong>, é fundamental que todo desenvolvedor, ou integrador de software tenha 
o controle de tudo ou quase tudo que acontece no banco. é imprescindível que não tenhamos esse controle. É uma forma de saber se as consultas estão sendo geradas corretamente, mas em outra oportunidade iremos falar mais sobre isso, enfim ..., como o objetivo maior de nosso artigo é mostrar como visualizar/capturar os comandos enviados para o banco, nada mais justo que utilizarmos o <strong>EntityFramework Core</strong> para isso 😊.
<br><br>
O EF Core fornece já um conjunto de opções para que possamos verificar as saídas SQL, vale a pena ressaltar que para o SQL Server temos o magnífico <strong>SQL Server Profiler</strong>, monitor de instruções SQL em tempo real, ótimo para saber quais querys por exemplo consumiram mais tempo.
<br><br>
Iremos apresentar aqui 2 opções de Logs (1-Log no console do aplicativo, 2-Log em uma variável) e criaremos uma extensão para projetar o SQL de uma consulta LINQ (Queryable).
</div>
## Estrutura de nosso projeto
**Class Blog**
```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime Date { get; set; } 
}
```

**Nosso DbContext**
```csharp
public class SampleContext : DbContext
{ 
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        var sqlConnectionStringBuilder = "Server=(localdb)\\mssqllocaldb;Database=ExemploArtigo;Integrated Security=True;"; 
        optionsBuilder.UseSqlServer(sqlConnectionStringBuilder); 
        base.OnConfiguring(optionsBuilder);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>();
    }
}
```

Até aqui tudo bem, temos já o principal para continuar com nosso artigo.
## Registro de Logs
<div class="notice--success">
<strong>Considerações:</strong><br>
Para usar alguma das opções abaixo, tem que instalar, são pacotes separados, então requer uma instalação.<br><br>
<strong>Alguns dos principais registros de Logs são:</strong><br><br>

<strong>Microsoft.Extensions.Logging.Console</strong><br> Um agente de log de console simples.<br><br>
<strong>Microsoft.Extensions.Logging.AzureAppServices:</strong><br>Serviços de aplicativo do Azure oferece suporte a 'Logs de diagnóstico' e recursos de fluxo de Log.<br><br>
<strong>Microsoft.Extensions.Logging.Debug</strong><br>Logs de um monitor de depuração usando System.Diagnostics.Debug.WriteLine().<br><br>
<strong>Microsoft.Extensions.Logging.EventLog</strong><br>Registros de log de eventos do Windows.<br><br>
<strong>Microsoft.Extensions.Logging.EventSource</strong><br>Dá suporte a EventSource/EventListener.<br><br>
<strong>Microsoft.Extensions.Logging.TraceSource</strong><br>Logs para um ouvinte de rastreamento usando System.Diagnostics.TraceSource.TraceEvent().<br><br>
</div>
<div class="notice--success"> 
Você pode ver mais informações sobre as opções apresentadas aqui:<br>
<a href="https://docs.microsoft.com/pt-br/ef/core/miscellaneous/logging" target="_BLACK">https://docs.microsoft.com/pt-br/ef/core/miscellaneous/logging</a> <br>
que por sinal é uma excelente documentação.<br>
</div>
## Mão na massa
Vamos agora ver como utilizar alguns deles.<br>
<b>Primeiramente o Console</b><br />
O que o <b><i>AddConsole</i></b> faz é jogar todas instruções SQL no console do aplicativo, é bem simples, após a instalação do pacote basta apenas referenciar.

O pacote <b><i>Microsoft.Extensions.Logging.Console</i></b> disponibiliza um método de extensão <b>AddConsole</b> para o <b>LoggerFactory</b><br>
Veja um exemplo simples de fazer isso!
```csharp
var logConsole = new LoggerFactory().AddConsole();
optionsBuilder.UseLoggerFactory(logConsole);
```
<strong>Completo:</strong>
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    var strConexao = "..."; 
    optionsBuilder.UseSqlServer(strConexao); 
    optionsBuilder.UseLoggerFactory(new LoggerFactory().AddConsole());
}
```
<br>
<strong>Output SQL de meu aplicativo console:</strong>
<br>
```sql
info: Microsoft.EntityFrameworkCore.Infrastructure[10403]
      Entity Framework Core 2.1.1-rtm-30846 initialized 'SampleContext' using provider 'Microsoft.EntityFrameworkCore.SqlServer' with options: None
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (146ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE') SELECT 1 ELSE SELECT 0
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (62ms) [Parameters=[@p0='?' (DbType = DateTime2), @p1='?' (Size = 4000)], CommandType='Text', CommandTimeout='30']
      SET NOCOUNT ON;
      INSERT INTO [Blogs] ([Date], [Name])
      VALUES (@p0, @p1);
      SELECT [Id]
      FROM [Blogs]
      WHERE @@ROWCOUNT = 1 AND [Id] = scope_identity();
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (15ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT [p].[Id], [p].[Date], [p].[Name]
      FROM [Blogs] AS [p]
      WHERE [p].[Id] > 0
``` 
Muito simples não é?!<br>
Certo, mas aqui temos apenas as querys sendo projetadas no console, eu gostaria de ter algo mais customizado isso é possível?<br>
Resposta: <strong>SIM</strong><br><br>
E iremos ver um exemplo básico de como podemos construir um log manipulável, é um exemplo básico, mas você terá uma ideia de como construir algo mais complexo para sua aplicação!
## Log Customizado
Agora a coisa começa a ficar melhor... :), vamos criar uma classe com a seguinte estrutura:
<br>
<strong>Classe responsável por fazer a manipulação do log.</strong>
```csharp
private class CustomLoggerProvider : ILoggerProvider
{
    public ILogger CreateLogger(string categoryName) => new SampleLogger(); 

    private class SampleLogger : ILogger
    {
        public void Log<TState>(
            LogLevel logLevel, 
            EventId eventId, 
            TState state, 
            Exception exception,
            Func<TState, Exception, string> formatter)
        {
            if (eventId.Id == RelationalEventId.CommandExecuting.Id)
            {
                var log = formatter(state, exception);
                Logs.Add(log); 
            }
        }

        public bool IsEnabled(LogLevel logLevel) => true;

        public IDisposable BeginScope<TState>(TState state) => null;
    }

    public void Dispose() { }
}
```
<strong>Observações:</strong><br>
existe uma variável <b>Logs</b> em minha classe acima, e minha classe também está como privada, fiz isso para não expor ela, apenas quero utilizar de forma que apenas meu DbContext tenha acesso a ela, veja nosso exemplo completo como ficou.

<strong>Nosso contexto completo ficou assim:</strong>
```csharp
public class SampleContext : DbContext
{
    public SampleContext()
    {
        if (Logs == null)
        {
            this.GetService<ILoggerFactory>().AddProvider(new CustomLoggerProvider());
            Logs = new List<string>();
        }
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        var sqlConnectionStringBuilder = "Server=(localdb)\\mssqllocaldb;Database=ExemploExtensao;Integrated Security=True;";
        optionsBuilder.UseSqlServer(sqlConnectionStringBuilder); 
        base.OnConfiguring(optionsBuilder);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>();
    }

    public static IList<string> Logs = null;

    private class CustomLoggerProvider : ILoggerProvider
    {
        public ILogger CreateLogger(string categoryName) => new SampleLogger();

        private class SampleLogger : ILogger
        {
            public void Log<TState>(
                LogLevel logLevel,
                EventId eventId,
                TState state,
                Exception exception,
                Func<TState, Exception, string> formatter)
            {
                if (eventId.Id == RelationalEventId.CommandExecuting.Id)
                {
                    var log = formatter(state, exception);
                    Logs.Add(log);
                }
            }

            public bool IsEnabled(LogLevel logLevel) => true;

            public IDisposable BeginScope<TState>(TState state) => null;
        }

        public void Dispose() { }
    }
}
```
Essa classe é tudo que precisamos para criar uma instância de <strong>ILogger</strong>, onde é feito todo rastreamento das query's, mas claro falando de forma genérica, já que podemos fazer "N" coisas!<br>
Feito isso vamos agora injetar/adicionar ele como um provider customizado, a forma mais simples é recuperar o serviço que já foi injetado por (DI - injeção de dependência) através da interface <strong>ILoggerFactory</strong>, da seguinte maneira.
```csharp
this.GetService<ILoggerFactory>().AddProvider(new CustomLoggerProvider());
```
Como foi mostrado no exemplo completo acima!
<br>
<strong>Nosso exemplo de uso</strong>
```csharp
static void Main(string[] args)
{
    using(var db = new SampleContext())
    {
        db.Database.EnsureCreated();
        db.Set<Blog>().Add(new Blog
        {
            Name = "Rafael Almeida",
            Date = DateTime.Now
        });
        db.SaveChanges();

        db.Set<Blog>().Where(p=>p.Id > 0).ToList();
    }

	// Recuperar o log dos comandos executados
    foreach (var log in SampleContext.Logs)
    {
        Console.WriteLine(log);
    }
    Console.ReadKey();
}
```

## Vamos criar nossa Extensão
Sabemos que podemos monitorar os comandos SQL como mostrado acima, mas em alguns casos podemos querer ver uma query especifica de um comando LINQ especifico.<br><br>
O EF Core nos fornece uma possibilidade de obter todo DDL de nosso banco de dados com o método de extensão <strong>GenerateCreateScript</strong> disponibilizado pelo próprio EF Core, veja o exemplo abaixo:
```csharp
var scriptBanco = db.Database.GenerateCreateScript();
```
Agora iremos construir nosso próprio conversor LINQ to SQL com base em uma consulta LINQ tipo <b>Queryable</b>.<br><br>
Como sempre falo, <strong>System.Reflection</strong> I LOVE, SEMPRE, SEMPRE!!!<br>
<br>
Com algumas magias usando <b>Reflection</b> podemos fazer a recuperação de algumas informações serializadas que estão em memória.

## Algumas informações para você
<div class="notice--warning">
<b>EntityQueryProvider (IQueryCompiler)</b><br>
Essa API oferece suporte à infraestrutura do Entity Framework Core e não se destina a ser usada diretamente em seu código.<br><br>
<b>DatabaseDependencies</b><br>
Classe de parâmetro de dependências de serviço para o banco de dados. Esse tipo é normalmente usado por provedores de banco de dados (e outras extensões). Geralmente não é usado no código do aplicativo.<br>
Não construa instâncias dessa classe diretamente do provedor ou do código do aplicativo, pois a assinatura do construtor pode mudar à medida que novas dependências são adicionadas. Em vez disso, use esse tipo em seu construtor para que uma instância seja criada e injetada automaticamente pelo contêiner de injeção de dependência.<br>
</div>
<strong>Veja nossa classe completa</strong>
```csharp
public static class RalmsExtensionSql
{
    private static readonly TypeInfo _queryCompilerTypeInfo = typeof(QueryCompiler).GetTypeInfo();

    private static readonly FieldInfo _queryCompiler
        = typeof(EntityQueryProvider)
            .GetTypeInfo()
            .DeclaredFields
            .Single(x => x.Name == "_queryCompiler"); 

    private static readonly FieldInfo _queryModelGenerator
        = _queryCompilerTypeInfo
            .DeclaredFields
            .Single(x => x.Name == "_queryModelGenerator");

    private static readonly FieldInfo _database = _queryCompilerTypeInfo
        .DeclaredFields
        .Single(x => x.Name == "_database");

    private static readonly PropertyInfo _dependencies
        = typeof(Database)
            .GetTypeInfo()
            .DeclaredProperties
            .Single(x => x.Name == "Dependencies");

    public static string ToSql<T>(this IQueryable<T> queryable)
        where T : class
    {
        var queryCompiler = _queryCompiler.GetValue(queryable.Provider) as IQueryCompiler;
        var queryModelGen = _queryModelGenerator.GetValue(queryCompiler) as IQueryModelGenerator;
        var queryCompilationContextFactory
            = ((DatabaseDependencies)_dependencies.GetValue(_database.GetValue(queryCompiler)))
                .QueryCompilationContextFactory;

        var queryCompilationContext = queryCompilationContextFactory.Create(false);
        var modelVisitor = (RelationalQueryModelVisitor)queryCompilationContext.CreateQueryModelVisitor();

        modelVisitor.CreateQueryExecutor<T>(queryModelGen.ParseQuery(queryable.Expression));

        return modelVisitor
            .Queries
            .FirstOrDefault()
            .ToString();
    }
}
```
<br>
<strong>Exemplo de uso</strong>
```csharp
static void Main(string[] args)
{
    using (var db = new SampleContext())
    { 
        db.Database.EnsureCreated();
        db.Set<Blog>().Add(new Blog
        {
            Name = "Rafael Almeida",
            Date = DateTime.Now
        });
        db.SaveChanges(); 

        // Gerar/Projetar o SQL 
        var strSQL = db.Set<Blog>().Where(p => p.Id > 0).ToSql();
    } 

    Console.ReadKey();
}
```
<br>
<strong>Veja o exemplo</strong>
![01]({{site.url}}{{site.baseurl}}/assets/images/toSql.PNG)
 
## Referências
<a href="https://docs.microsoft.com/en-us/ef/core/api/microsoft.entityframeworkcore" target="_BLACK">EF Core</a><br>
<a href="https://docs.microsoft.com/en-us/ef/core/api/microsoft.entityframeworkcore.query.internal.querycompiler" target="_BLACK">QueryCompiler</a><br>
<a href="https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.storage.databasedependencies?view=efcore-2.1" target="_BLACK">DatabaseDependencies</a><br>
<a href="https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.query.relationalquerymodelvisitor?view=efcore-2.1" target="_BLACK">RelationalQueryModelVisitor</a><br>
<br><br>
Click <a href="https://github.com/ralmsdeveloper/artigolog" target="_BLACK">aqui</a> para acessar os fontes no github!
<br><br>
Pessoal, fico por aqui <strong>#efcore #mvp #mvpbr #mvpbuzz</strong>
