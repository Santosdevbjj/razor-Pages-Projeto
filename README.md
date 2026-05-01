# Agenda de Contatos — Razor Pages + .NET 8 + PostgreSQL

> **CRUD completo com Razor Pages:** agenda de contatos com cadastro, edição, exclusão e visualização de detalhes, construída com Repository Pattern, Entity Framework Core e PostgreSQL — demonstrando o padrão Razor Pages como alternativa ao MVC para aplicações web orientadas a página.

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![C#](https://img.shields.io/badge/C%23-12.0-239120?style=for-the-badge&logo=csharp&logoColor=white)](https://learn.microsoft.com/dotnet/csharp/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-14+-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![EF Core](https://img.shields.io/badge/EF_Core-8.0-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)](https://learn.microsoft.com/ef/core/)
[![License](https://img.shields.io/badge/Licença-MIT-green?style=for-the-badge)](LICENSE)

---

## 1. Problema de Negócio

Desenvolvedores .NET que aprendem ASP.NET Core via MVC tradicional frequentemente têm dificuldade para construir aplicações web orientadas a fluxos de página — onde cada URL corresponde a uma ação específica do usuário, como "criar contato", "editar contato" ou "confirmar exclusão". O padrão MVC foi projetado para APIs e aplicações com separação estrita de responsabilidades, mas pode ser excessivo quando o objetivo é uma aplicação web simples com interações diretas de formulário.

O problema concreto que este projeto endereça é duplo:

- **Acoplamento desnecessário entre controller e view em operações de página única:** no MVC, uma operação de CRUD de contatos exige um controller com múltiplos action methods, rotas configuradas manualmente e views separadas. Para cada página existe uma rota, um método, uma view — a navegação entre eles não é explícita no código.
- **Falta de coesão entre lógica de página e marcação:** Razor Pages coloca o `PageModel` e o `.cshtml` no mesmo diretório, com a mesma convenção de nome. Quem abre `Edit.cshtml` encontra `Edit.cshtml.cs` imediatamente ao lado. Esse alinhamento reduz o custo cognitivo de navegar pelo projeto.

**A questão central:** como construir uma aplicação web .NET com CRUD completo onde cada página seja autocontida — com sua lógica, validação e persistência — sem o overhead de um controller centralizado?

---

## 2. Contexto

Este projeto foi desenvolvido no contexto do estudo de Razor Pages como alternativa ao padrão MVC para aplicações web orientadas a formulário. O domínio — uma agenda de contatos com nome, telefone, email, data de nascimento e foto — foi escolhido por ser simples o suficiente para não obscurecer o padrão arquitetural, mas completo o suficiente para demonstrar todas as operações de CRUD com validação real.

A escolha do PostgreSQL como banco de dados reflete o ecossistema mais comum em aplicações .NET modernas que precisam de banco relacional open source — sem custo de licença, com suporte de primeira classe no EF Core via provider Npgsql, e compatível com ambientes cloud como Azure Database for PostgreSQL e Render.

O Repository Pattern foi adicionado deliberadamente para demonstrar que Razor Pages não implica abandono de boas práticas de separação de responsabilidades. O `PageModel` não acessa o `DbContext` diretamente — ele depende da interface `IContatoRepository`, o que mantém a testabilidade da lógica de página independente da implementação de persistência.

---

## 3. Premissas

Para estruturar este projeto, foram adotadas as seguintes premissas:

- Cada página Razor é autocontida: a lógica de negócio correspondente à página fica no `PageModel` da mesma pasta, não em um controller centralizado.
- A validação de dados é dupla: atributos de DataAnnotations no modelo (`[Required]`, `[EmailAddress]`, `[StringLength]`) garantem validação server-side; `asp-validation-summary` e tag helpers expõem os erros ao usuário sem JavaScript adicional.
- O repositório isola completamente o acesso ao banco de dados. Os `PageModels` dependem de `IContatoRepository`, não de `AppDbContext`. Isso permite substituir a implementação em testes sem modificar as páginas.
- O campo `Foto` armazena uma URL de imagem, não um arquivo binário. Isso simplifica o armazenamento inicial e deixa a evolução para upload real (Azure Blob Storage, por exemplo) como próximo passo natural.
- Connection strings nunca são versionadas com credenciais reais. O `appsettings.json` contém apenas credenciais de desenvolvimento local; em produção, variáveis de ambiente sobrescrevem os valores.

---

## 4. Estratégia da Solução

**Razor Pages como unidade de organização**

Em vez de agrupar lógica por tipo (`Controllers/`, `Views/`), Razor Pages agrupa por funcionalidade: `Pages/Contatos/` contém todas as páginas e seus respectivos `PageModels` para o domínio de contatos. Cada arquivo `.cshtml` tem seu `.cshtml.cs` ao lado — a correspondência é explícita pela convenção de nome, não por roteamento configurado separadamente.

As rotas são definidas diretamente nas páginas via `@page "{id:int}"`, com constraint de tipo que rejeita IDs inválidos antes de chegar ao `OnGetAsync`. Isso elimina uma classe inteira de bugs onde IDs inválidos chegam ao repositório.

**Repository Pattern com interface**

O `IContatoRepository` define cinco operações assíncronas: `GetAllAsync`, `GetByIdAsync`, `AddAsync`, `UpdateAsync` e `DeleteAsync`. O `ContatoRepository` implementa a interface usando o `AppDbContext` com EF Core. O `Program.cs` registra `IContatoRepository` com `AddScoped<IContatoRepository, ContatoRepository>()`, garantindo que cada request HTTP receba uma instância isolada do repositório e do contexto.

**Fluxo de dados em cada operação CRUD**

`OnGetAsync` carrega dados do repositório e os expõe via propriedades do `PageModel` para a view. `OnPostAsync` recebe dados via `[BindProperty]`, valida com `ModelState.IsValid` e persiste via repositório. Em caso de erro de validação, a página é reexibida com os erros inline. Em caso de sucesso, o redirecionamento para `Index` evita resubmissão do formulário no reload.

**EF Core com Code-First e migrations**

O `AppDbContext` expõe apenas `DbSet<Contato>`. As migrations são geradas pelo EF Core CLI a partir do modelo C# — o banco PostgreSQL é criado e versionado pelo código, não por scripts SQL manuais. O comando `dotnet ef database update` aplica as migrations e cria o banco `AgendaDb` automaticamente.

---

## 5. Decisões Técnicas e Aprendizados

### Por que Razor Pages e não MVC?

A escolha foi deliberada para aprender a diferença entre os dois padrões. MVC faz mais sentido quando múltiplas views compartilham a mesma lógica de controller ou quando a separação entre modelo, view e controller precisa ser explícita (APIs, por exemplo). Razor Pages faz mais sentido quando cada URL corresponde a uma ação específica do usuário e a lógica é coesa com a página. Para uma agenda de contatos, cada operação é uma página distinta — Razor Pages reflete essa realidade melhor do que MVC.

### Por que Repository Pattern em vez de acesso direto ao DbContext?

O `PageModel` poderia acessar o `AppDbContext` diretamente — é mais simples e o EF Core é testável com bancos em memória. A decisão de adicionar o repositório foi educacional: demonstrar que Razor Pages não implica abandono de padrões de design. O benefício prático aparece quando a implementação de persistência muda — trocar PostgreSQL por SQL Server, por exemplo, exige mudanças apenas no `ContatoRepository` e no provider, não nas páginas.

### O aprendizado sobre `[BindProperty]` e resubmissão de formulário

Na primeira versão, o `OnPostAsync` do `EditModel` não redirecionava após salvar — apenas retornava `Page()`. O resultado: ao recarregar a página após edição, o browser resubmetia o formulário e duplicava a operação de update. A solução foi o padrão Post-Redirect-Get: `OnPostAsync` salva e redireciona para `Index`, que é um GET. O reload do browser refaz o GET, não o POST. Esse padrão é fundamental em qualquer aplicação web com formulários.

### O que faria diferente

Substituiria o campo `Foto` de URL de string por upload real com `IFormFile`, armazenando os arquivos em `wwwroot/imagens/` com nome gerado por GUID para evitar colisões. Também adicionaria paginação no `Index` para não carregar todos os contatos de uma vez — o `GetAllAsync` atual não tem limite, o que se torna problema com volumes maiores.

---

## 6. Resultados

O projeto entregou uma aplicação web funcional com os seguintes resultados concretos:

**CRUD completo operacional:** 5 páginas Razor (Index, Create, Edit, Delete, Details) com seus respectivos `PageModels`, cobrindo todas as operações de gestão de contatos com validação server-side e feedback de erro inline.

**Validação robusta no modelo:** o `Contato` tem 5 campos validados com DataAnnotations — `[Required]`, `[StringLength]`, `[EmailAddress]` e `[DataType]`. Entradas inválidas são rejeitadas antes de chegar ao repositório, com mensagens de erro exibidas via `asp-validation-summary`.

**Repository Pattern testável:** `IContatoRepository` com 5 métodos assíncronos, implementado por `ContatoRepository` via EF Core. A interface permite substituir a implementação sem modificar as páginas.

**Banco PostgreSQL gerenciado por migrations:** o `AppDbContext` com `DbSet<Contato>` e as migrations do EF Core criam e versionam o banco `AgendaDb` automaticamente. Qualquer clone do repositório com PostgreSQL disponível pode rodar a aplicação com dois comandos.

**Armazenamento de fotos por URL:** o campo `Foto` aceita URLs de imagem externas, exibidas na página de detalhes via tag `<img>`. Simples, funcional, e com caminho claro para evolução a upload local.

---

## 7. Próximos Passos

- Implementar upload real de imagem com `IFormFile`, armazenando arquivos em `wwwroot/imagens/` com nome gerado por GUID.
- Adicionar paginação no `Index` com parâmetros `page` e `pageSize`, evitando carregamento irrestrito de contatos.
- Implementar busca por nome no `Index` com filtro no repositório (`GetByNameAsync`).
- Adicionar autenticação com ASP.NET Core Identity para proteger as operações de criação, edição e exclusão.
- Substituir DataAnnotations por FluentValidation para validações mais complexas (formato de telefone brasileiro, por exemplo).
- Adicionar testes unitários dos `PageModels` com mocks do `IContatoRepository`.

---

## Como Executar

### Pré-requisitos

- .NET SDK 8.0
- PostgreSQL 14+ em execução local
- Git

### Passo a passo

```bash
# 1. Clone o repositório
git clone https://github.com/Santosdevbjj/razor-Pages-Projeto.git
cd razor-Pages-Projeto

# 2. Configure a connection string em appsettings.json
# "PostgresConnection": "Host=localhost;Database=AgendaDb;Username=postgres;Password=SUA_SENHA"

# 3. Instale a CLI do EF Core (se necessário)
dotnet tool install --global dotnet-ef

# 4. Crie o banco e aplique as migrations
dotnet ef migrations add InitialCreate
dotnet ef database update

# 5. Execute a aplicação
dotnet run

# Acesse em: http://localhost:5000
```

---

## Estrutura do Projeto

```
razor-Pages-Projeto/
├── Program.cs                        # DI: DbContext, Repository, Razor Pages
├── appsettings.json                  # Connection string PostgreSQL
├── AgendaContatos.csproj             # EF Core, Npgsql
│
├── Models/
│   └── Contato.cs                    # Id, Nome, Telefone, Email, DataNascimento, Foto
│
├── Data/
│   └── AppDbContext.cs               # DbSet<Contato>
│
├── Services/
│   ├── IContatoRepository.cs         # Contrato: 5 operações assíncronas
│   └── ContatoRepository.cs          # Implementação via EF Core
│
├── Pages/
│   ├── Index.cshtml                  # Página inicial
│   └── Contatos/
│       ├── Index.cshtml              # Lista de contatos com ações
│       ├── Index.cshtml.cs           # OnGetAsync → carrega lista
│       ├── Create.cshtml             # Formulário de criação
│       ├── Create.cshtml.cs          # OnPostAsync → valida e salva
│       ├── Edit.cshtml               # Formulário de edição
│       ├── Edit.cshtml.cs            # OnGetAsync + OnPostAsync
│       ├── Delete.cshtml             # Confirmação de exclusão
│       ├── Delete.cshtml.cs          # OnGetAsync + OnPostAsync
│       ├── Details.cshtml            # Visualização completa
│       └── Details.cshtml.cs         # OnGetAsync → carrega por ID
│
└── wwwroot/
    └── imagens/                      # Diretório para imagens locais
```

---

## Fluxo CRUD

```
Usuário acessa /Contatos
    │
    ▼
Index.cshtml ←─── OnGetAsync() → IContatoRepository.GetAllAsync()
    │
    ├── [Novo Contato] → /Contatos/Create
    │       └── OnPostAsync() → AddAsync() → RedirectToPage("Index")
    │
    ├── [Editar] → /Contatos/Edit/{id}
    │       ├── OnGetAsync(id) → GetByIdAsync(id)
    │       └── OnPostAsync() → UpdateAsync() → RedirectToPage("Index")
    │
    ├── [Detalhes] → /Contatos/Details/{id}
    │       └── OnGetAsync(id) → GetByIdAsync(id)
    │
    └── [Excluir] → /Contatos/Delete/{id}
            ├── OnGetAsync(id) → GetByIdAsync(id) [confirmação]
            └── OnPostAsync(id) → DeleteAsync(id) → RedirectToPage("Index")
```

---

## Tecnologias Utilizadas

| Camada | Tecnologia | Justificativa |
|---|---|---|
| **Runtime** | .NET 8, C# 12 | LTS, nullable safety, records |
| **UI** | Razor Pages | Coesão página-lógica, convenção sobre configuração |
| **ORM** | EF Core 8 + Npgsql | Code-first, migrations automáticas, PostgreSQL nativo |
| **Banco** | PostgreSQL 14+ | Open source, sem licença, padrão em cloud |
| **Validação** | DataAnnotations + Tag Helpers | Server-side com feedback inline sem JavaScript |
| **DI** | ASP.NET Core built-in | `AddScoped` para DbContext e repositório |

---

## Autor

**Sergio Santos**  
Data Engineer & Cloud Architect — 15+ anos em sistemas críticos bancários (Bradesco)  
Campus Expert DIO · Bootcamp GFT Start #7 .NET

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)
[![GitHub](https://img.shields.io/badge/GitHub-Santosdevbjj-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Santosdevbjj)
