# DIO - Trilha .NET - Nuvem com Microsoft Azure
www.dio.me

---

## 🔧 Desafio de projeto
Implementar uma Web API em .NET para um sistema de RH que realize operações CRUD (criar, ler, atualizar e excluir) de funcionários. A cada alteração (inclusão, atualização ou exclusão), um log completo do funcionário deve ser armazenado em uma Azure Table.

---

## ✅ Objetivo

Construir uma **Web API** que:

1. Realize operações **CRUD** em funcionários.
2. Armazene **logs de alterações** (criação, atualização, exclusão) em uma **Azure Table**.
3. Seja implantada no **Azure App Service**, com **SQL Database** e **Azure Storage Account**.

---

## 📁 Estrutura de Arquivos (Já Pronta!)

Você já tem os arquivos essenciais:

- `Models/Funcionario.cs`
- `Models/FuncionarioLog.cs`
- `Models/TipoAcao.cs`
- `Controllers/FuncionarioController.cs` ← com `TODOs` a serem completados
- `appsettings.json` ← com `ConnectionStrings`

---

## 🔧 Passo 1: Complete o Modelo `Funcionario.cs`

O schema exigido é:

```json
{
  "nome":"Nome funcionario",
  "endereco":"Rua 1234",
  "ramal":"1234",
  "emailProfissional":"email@email.com",
  "departamento":"TI",
  "salario": 1000,
  "dataAdmissao":"2022-06-23T02:58:36.345Z"
}
```

### ✅ Código Final (`Funcionario.cs`)

```csharp
using System.ComponentModel.DataAnnotations;

namespace TrilhaNetAzureDesafio.Models;

public class Funcionario
{
    [Key]
    public int Id { get; set; }

    [Required]
    public string Nome { get; set; } = string.Empty;

    [Required]
    public string Endereco { get; set; } = string.Empty;

    [Required]
    public string Ramal { get; set; } = string.Empty;

    [Required]
    public string EmailProfissional { get; set; } = string.Empty;

    [Required]
    public string Departamento { get; set; } = string.Empty;

    [Required]
    public decimal Salario { get; set; }

    [Required]
    public DateTime DataAdmissao { get; set; }
}
```

> ⚠️ Certifique-se de que o campo `Id` seja auto-incrementado pelo banco (EF Core faz isso automaticamente).

---

## 📝 Passo 2: Implemente `FuncionarioLog.cs` (Filha de `Funcionario` + `ITableEntity`)

Como exigido:  
> *“A classe FuncionarioLog é filha da classe Funcionario, pois o log terá as mesmas informações da Funcionario.”*

Ela precisa implementar `ITableEntity` para funcionar com Azure Table Storage.

### ✅ Código Final (`FuncionarioLog.cs`)

```csharp
using Azure.Data.Tables;

namespace TrilhaNetAzureDesafio.Models;

public class FuncionarioLog : Funcionario, ITableEntity
{
    public string PartitionKey { get; set; } = string.Empty;
    public string RowKey { get; set; } = string.Empty;
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public TipoAcao TipoAcao { get; set; }

    // Construtor padrão necessário para o Azure Table
    public FuncionarioLog() { }

    public FuncionarioLog(Funcionario funcionario, TipoAcao tipoAcao, string departamento, string rowKey)
    {
        // Copiar todos os campos do Funcionario
        Id = funcionario.Id;
        Nome = funcionario.Nome;
        Endereco = funcionario.Endereco;
        Ramal = funcionario.Ramal;
        EmailProfissional = funcionario.EmailProfissional;
        Departamento = funcionario.Departamento;
        Salario = funcionario.Salario;
        DataAdmissao = funcionario.DataAdmissao;

        TipoAcao = tipoAcao;

        // Configurar chaves do Azure Table
        PartitionKey = departamento; // Ex: "TI", "RH" — usado para agrupar logs por departamento
        RowKey = rowKey; // ID único para cada registro de log
    }
}
```

---

## 🧭 Passo 3: Defina o Enum `TipoAcao.cs`

Crie um enum simples para identificar a ação realizada:

### ✅ Código Final (`TipoAcao.cs`)

```csharp
namespace TrilhaNetAzureDesafio.Models;

public enum TipoAcao
{
    Inclusao,
    Atualizacao,
    Remocao
}
```

---

## 🛠️ Passo 4: Complete o `FuncionarioController.cs` (Resolva os TODOs)

Aqui está o código **completo e corrigido** do controlador, com todos os `TODOs` implementados:

```csharp
using Azure.Data.Tables;
using Microsoft.AspNetCore.Mvc;
using TrilhaNetAzureDesafio.Context;
using TrilhaNetAzureDesafio.Models;

namespace TrilhaNetAzureDesafio.Controllers;

[ApiController]
[Route("[controller]")]
public class FuncionarioController : ControllerBase
{
    private readonly RHContext _context;
    private readonly string _connectionString;
    private readonly string _tableName;

    public FuncionarioController(RHContext context, IConfiguration configuration)
    {
        _context = context;
        _connectionString = configuration.GetValue<string>("ConnectionStrings:SAConnectionString");
        _tableName = configuration.GetValue<string>("ConnectionStrings:AzureTableName");
    }

    private TableClient GetTableClient()
    {
        var serviceClient = new TableServiceClient(_connectionString);
        var tableClient = serviceClient.GetTableClient(_tableName);

        tableClient.CreateIfNotExists();
        return tableClient;
    }

    [HttpGet("{id}")]
    public IActionResult ObterPorId(int id)
    {
        var funcionario = _context.Funcionarios.Find(id);

        if (funcionario == null)
            return NotFound();

        return Ok(funcionario);
    }

    [HttpPost]
    public IActionResult Criar(Funcionario funcionario)
    {
        _context.Funcionarios.Add(funcionario);
        _context.SaveChanges(); // ✅ Salva no SQL

        var tableClient = GetTableClient();
        var funcionarioLog = new FuncionarioLog(funcionario, TipoAcao.Inclusao, funcionario.Departamento, Guid.NewGuid().ToString());

        tableClient.UpsertEntity(funcionarioLog); // ✅ Loga no Azure Table

        return CreatedAtAction(nameof(ObterPorId), new { id = funcionario.Id }, funcionario);
    }

    [HttpPut("{id}")]
    public IActionResult Atualizar(int id, Funcionario funcionario)
    {
        var funcionarioBanco = _context.Funcionarios.Find(id);

        if (funcionarioBanco == null)
            return NotFound();

        // ✅ Atualiza TODOS os campos
        funcionarioBanco.Nome = funcionario.Nome;
        funcionarioBanco.Endereco = funcionario.Endereco;
        funcionarioBanco.Ramal = funcionario.Ramal;
        funcionarioBanco.EmailProfissional = funcionario.EmailProfissional;
        funcionarioBanco.Departamento = funcionario.Departamento;
        funcionarioBanco.Salario = funcionario.Salario;
        funcionarioBanco.DataAdmissao = funcionario.DataAdmissao;

        _context.SaveChanges(); // ✅ Salva no SQL

        var tableClient = GetTableClient();
        var funcionarioLog = new FuncionarioLog(funcionarioBanco, TipoAcao.Atualizacao, funcionarioBanco.Departamento, Guid.NewGuid().ToString());

        tableClient.UpsertEntity(funcionarioLog); // ✅ Loga no Azure Table

        return Ok();
    }

    [HttpDelete("{id}")]
    public IActionResult Deletar(int id)
    {
        var funcionarioBanco = _context.Funcionarios.Find(id);

        if (funcionarioBanco == null)
            return NotFound();

        _context.Funcionarios.Remove(funcionarioBanco); // ✅ Marca para exclusão
        _context.SaveChanges(); // ✅ Salva no SQL

        var tableClient = GetTableClient();
        var funcionarioLog = new FuncionarioLog(funcionarioBanco, TipoAcao.Remocao, funcionarioBanco.Departamento, Guid.NewGuid().ToString());

        tableClient.UpsertEntity(funcionarioLog); // ✅ Loga no Azure Table

        return NoContent();
    }
}
```

---

## 🔄 Passo 5: Gere a Migration e Atualize o Banco

No **Package Manager Console**:

```bash
Add-Migration InitialCreate
Update-Database
```

Isso cria a tabela `Funcionarios` no banco SQL.

---

## ☁️ Passo 6: Configure e Publique no Azure

### 1. Crie os Recursos no Azure Portal:
- **App Service** → para hospedar sua Web API.
- **SQL Database** → para armazenar os funcionários.
- **Storage Account** → crie uma **Azure Table** chamada `FuncionarioLog`.

### 2. Configure as Strings de Conexão no App Service:
No portal do Azure, vá até seu App Service > **Configurações > Configuração** > **Cadeias de conexão**:

| Nome | Valor | Tipo |
|------|-------|------|
| `ConexaoPadrao` | String de conexão do SQL DB | SQLServer |
| `SAConnectionString` | String de conexão do Storage Account | Custom |
| `AzureTableName` | `FuncionarioLog` | Custom |

> 💡 Para obter a string do Storage Account:  
> Storage Account → **Chaves de acesso** → copie a `connection string`.

---

## 🧪 Teste Localmente com Swagger

Acesse:  
👉 `https://localhost:5001/swagger`

Teste os endpoints:
- `POST /Funcionario` → criar
- `GET /Funcionario/{id}` → buscar
- `PUT /Funcionario/{id}` → atualizar
- `DELETE /Funcionario/{id}` → deletar

Verifique se os logs estão sendo gravados na Azure Table (pode usar o **Azure Storage Explorer** ou o portal).

---

## ✅ Checklist Final

| Item | Status |
|------|--------|
| ✅ Modelos `Funcionario`, `FuncionarioLog`, `TipoAcao` criados corretamente | ✔️ |
| ✅ Todos os `TODOs` no controller resolvidos | ✔️ |
| ✅ Migration gerada e aplicada | ✔️ |
| ✅ `appsettings.json` com strings de conexão corretas | ✔️ |
| ✅ Aplicação publicada no Azure App Service | ✔️ |
| ✅ Logs salvos na Azure Table `FuncionarioLog` | ✔️ |
