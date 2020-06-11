﻿---
title: "Perform Identity Resolution"
comments: true
excerpt_separator: "Ler mais"
toc: true
toc_label: "Tópicos"
categories:
  - EF5
  - EntityFrameworkCore
---

![01]({{site.url}}{{site.baseurl}}/assets/images/ef5identityresolution/EF5_PerformIdentityResolution.png)

<center><strong>Olá tudo bem?!</strong></center>
<hr /> 
<div class="notice--warning">
O novo recurso que irei apresentar está em preview ainda, e será lançada de oficialmente em Novembro deste ano!
<br><br>
<b>Entity Framework Core 5</b> será umas das versões mais esperadas até o momento e está recheada de várias novidades, hoje irei apresentar um recurso muito interessante e extremamente importante.
</div> 

## AsNoTracking
<div style="text-align: justify;">
AsNoTracking é um dos recursos mais utilizados por usuários do <b>Entity Framework Core</b> para fazer consultas, 
constumamos dizer que é uma consulta somente leitura, isso significa que os dados retornados pela consulta não 
serão rastreados e pode existir situações que se torna muito mais rápido, por não ter essa responsabilidade de 
gerenciar o estado do objeto.
<br /><br />
Veja uma exemplo de uma consulta utilizando <b>AsNoTracking</b>:
</div>
```csharp
using var db = new ExemploContext();

var itens = db
    .Itens
    .AsNoTracking()
    .Include(p => p.Pedido)
    .ToList()
```
<div style="text-align: justify;">
Basicamente esse é o comportamento que todos conhecem, mas existe algo que você precisa saber, na consulta acima
para cada <b>Item</b> será criada uma nova instância de <b>Pedido</b>, de forma resumida é o seguinte, se sua consulta 
retornou 1.000 (mil itens) e todos fazem parte de um único <b>Pedido</b>, teremos 2.000 (duas mil) instâncias de objetos agora, 
isso pode ser um problema de uso de <b>memória</b>, e pode causar lentidão em sua aplicação, o time do <b>Entity Framework Core</b> 
vem fazendo um ótimo trabalho e fazendo com que o <b>ORM</b> a cada versão seja mais produtivo e performático.<br><br>
</div>
## PerformIdentityResolution
<div style="text-align: justify;">
Certo temos um problema e qual é a solução?
Existe uma nova feature, que é um método de extensão, extremamente inteligente e capaz de resolver esse problema de alocar objetos em memória,
assim em vez de ter 1.000(mil) instâncias de <b>Pedido</b>, passa agora ter uma única instância e a lista de <b>Itens</b> agora passa a usar esta única referência, 
veja como ficou simples de resolver isso:
</div>
```csharp
using var db = new ExemploContext();

var itens = db
    .Itens
    .AsNoTracking()
    .PerformIdentityResolution() // Aqui está a solução
    .Include(p => p.Pedido)
    .ToList()
```
Observe que agora usamos o seguinte metódo (<b>PerformIdentityResolution</b>) ele é o responsável por resolver esse pequeno problema de alocação de objetos em memória.
## Twitter
<div class="notice--info">
 Fico por aqui! 😄 <br />
 Me siga no twitter: <a alt="" href="https://twitter.com/RalmsDeveloper">@ralmsdeveloper</a><br />
</div> 

<br>