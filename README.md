<h1 align="center">SOLID com C#</h1>

| . | Nome do Principio               | Abr. | Descrição                                                                                                      |
|---|---------------------------------|------|----------------------------------------------------------------------------------------------------------------|
| S | Single Responsibility Principle | SRP  | Cada Classe ou modulo deve ter apenas uma razão para mudar                                                     |
| O | Open/Closed Principle           | OCP  | Entidades de Software (classes, módulos, funções), devem ser abertas a extensão, mas fechadas para modificação |
| L | Liskov Substitution Principle   | LSP  | Classes bases devem ser substituíveis pelas classes derivadas                                                  |
| I | Interface Segregation Principle | ISP  | Faça interfaces com alta granularidade, para cada cliente                                                      |
| D | Dependency Inversion Principle  | DIP  | Dependa de Abstrações, não de classes concretas                                                                |

## Requisitos
- Visual Studio
- Conhecimento em C# e .NET

Sigle Responsability Principle
==============================

O Principio da responsabilidade única diz que os elementos de software (módulos, classes, funções, etc...) devem ter apenas uma responsabilidade. Nesse contexto responsabilidade é definido como "uma razão para mudar".

Para exemplificar, no projeto, a class UsuárioService tinha 4 responsabilidades, como pode ser visto no método Login:

```C#
namespace solidInCsharp.Service
{
    public class UsuarioService
    {
        // 1 - Responsabilidade do service (lógica de negócio)
        public string Login(string Email,  string Password) {
            // 2 - Responsabilidade de lógica de banco de dados
            var user = (from u in this.repository.Query() where u.Email == Email select u).FirstOrDefault();
            // 3 - Responsabilidade de cripgrafia de senha
            if (user == null || !ValidarSenha(user.Senha, Password)) {
                throw new Exception("Erro, usuário ou senha incorreto");
            }
            // 4 - Responsabilidade de gerar Token JWT
            return GerarToken(user);
        }
    }
}
```

Para fazer a correção, separamos cada uma das responsabilidades em suas classes.
Nosso método de Login fica então:

```C#
namespace solidInCsharp.Service
{
    public class UsuarioService
    {
        // 1 - Responsabilidade do service (lógica de negócio)
        public string Login(string Email,  string Password) {
            // 2 - Responsabilidade de lógica de banco de dados
            var user = repository.ObterUsuario(Email);
            // 3 - Responsabilidade de cripgrafia de senha
            if (user == null || ! new CriptografiaService().ValidarSenha(user.Senha, Password)) {
                throw new Exception("Erro, usuário ou senha incorreto");
            }
            // 4 - Responsabilidade de gerar Token JWT
            return new JWTService().GerarToken(user);
        }
    }
}
```

Open/Closed Principle
=====================

O pricípio aberto/fechado, diz que um elemento de software deve ser aberto para extensão, mas fechado para modificações.

Nosso projeto de exemplo violava esse princípio na classe ProdutoReportService.cs, ao exigir que para criar um novo formato de relatório o método fosse completamente modificado.

Além disso, o método possui duas responsabilidades, uma com o formato de dados (formato do arquivo), e outra com o conteúdo (colunas e linhas), então este método viola também o [1-SRP.md](SRP).

```C#
namespace solidInCsharp.Service
{
    public class ProdutoReportService
    {
        //trecho omitido
        public string GerarRelatorio(TipoRelatorio tipo) {
            var produtos = this.repository.ListAll();
            if (tipo == TipoRelatorio.CSV) {
                string relatorio = "ID;Nome;Preço\r\n";
                foreach (var produto in produtos)
                {
                    relatorio += produto.Id + ";" + produto.Nome + ";" + produto.Preco + "\r\n";
                }
                return relatorio;
            } else if (tipo == TipoRelatorio.HTML) {
                string relatorio = "<html><body><table>\r\n<th><td>ID</td><td>Nome</td><td>Preço</td></th>\r\n";
                foreach (var produto in produtos)
                {
                    relatorio += "<tr><td>" + produto.Id + "</td><td>" + produto.Nome + "</td><td>" + produto.Preco + "</td></tr>\r\n";
                }
                relatorio += "</table></body><html>";
                return relatorio;
            } // Se quiser adicionar um novo formato, tenho que modificar o código.
            throw new Exception("tipo de relatorio invalido");
        }
    }
}
```

Começamos nossa refatoração extraindo o comportamento comum dos relatórios tabularem, que possuem uma estrutura similar para todos os formatos:

* Início do Relatório
  * Inicio dos Headers
    * Colunas de Headers
  * Fim dos Headers
  * Inicio de Linha (N vezes)
    * Colunas das linhas
  * Fim da Linha
* Fim do Relatório

Para representar esse padrão, criamos a interface IReportGenerator:

```C#
namespace solidInCsharp.Service.Report
{
    public interface IReportGenerator
    {
        public void IniciarRelatorio();
        public void IniciarHeaders();
        public void AdicionarHeader(string valor);
        public void FinalizarHeaders();
        public void IniciarLinha();
        public void AdicionarColuna(string valor);
        public void FinalizarLinha();
        public string FinalizarRelatorio();
    }
}
```

Criamos também as implementações para CSV e HTML, e modificamos nosso método para gerar o relatório:

```C#
namespace solidInCsharp.Service
{
    public class ProdutoReportService
    {
        // trecho omitido
        public string GerarRelatorio(IReportGenerator generator) {
            var produtos = this.repository.ListAll();
            generator.IniciarRelatorio();
            generator.IniciarHeaders();
            generator.AdicionarHeader("ID");
            generator.AdicionarHeader("Nome");
            generator.AdicionarHeader("Preço");
            generator.FinalizarHeaders();
            foreach (var produto in produtos)
            {
                generator.IniciarLinha();
                generator.AdicionarHeader(produto.Id);
                generator.AdicionarHeader(produto.Nome);
                generator.AdicionarHeader(produto.Preco + "");
                generator.FinalizarLinha();
            }
            return generator.FinalizarRelatorio();
        }
        
    }
}
```

Com isso, nosso método de geração de relatório permite que novos formatos de relatórios sejam criados.

Para finalizar, criamos uma factory para construir os geradores de relatório e alteramos nosso Controller para a nova chamada.

Liskov Substitution Principle
=============================

O princípio da substituição de Liskov, diz que as classes bases devem poder ser substituídas pelas classes derivadas em todos os contextos, ou seja, uma classe derivada não pode modificar o "contrato" da classe base.

Nosso projeto viola esse princípio na classe ProdutoRepository:

```C#
namespace solidInCsharp.Repository
{
    public class ProdutoRepository: BaseRepository<Produto, ProdutoRepository>
    {

        // Código omitido

        public new void Add(Produto item) {
            throw new Exception("Products are readonly");
        }

        public new void Update(Produto item) {
            throw new Exception("Products are readonly");
        }

        public new void Remove(Produto item) {
            throw new Exception("Products are readonly");
        }
    }
}
```

Ao sobrescrever os métodos, e fazer eles lançarem excessão, a classe ProdutoRepository não pode ser substituir a classe BaseRepository em todas as suas atribuições.

Para corrigir, criamos uma nova classe BaseReadOnlyRepository, que somente possui os métodos de leitura e alteramos o ProdutoRepository para herdar desta classe.

```C#
namespace solidInCsharp.Repository
{
    public abstract class BaseReadOnlyRepository<T,K> : DbContext where T : class  where K : BaseReadOnlyRepository<T,K>
    {
        public BaseReadOnlyRepository(DbContextOptions<K> options)
            : base(options)
        { }

        protected DbSet<T> Items { get; set; }


        public IEnumerable<T> ListAll() {
            return this.Items.ToArray();
        }

        
        public IQueryable<T> Query() {
            return this.Items.AsQueryable();
        }


    }
}

using System;
using Microsoft.EntityFrameworkCore;
using solidInCsharp.Model;
using System.Linq;

namespace solidInCsharp.Repository
{
    public class ProdutoRepository: BaseReadOnlyRepository<Produto, ProdutoRepository>
    {

        public ProdutoRepository(DbContextOptions<ProdutoRepository> options)
            : base(options)
        {
            // trecho omitido
         }

    }
}
```

Com isso, nossa classe não mais ofende o LSP.

Interface Segregation Principle
===============================

O princípio de segregação de interface, diz que os clientes das nossas classes devem utilizar apenas as interfaces que eles precisam.

Nosso projeto viola esse princípio ao expor todos os métodos do EntityFramework para os clientes dos repositórios, o que poderia fazer com que eles façam usos das classes de formas que não pretendemos.

Para ajustar isso, criamos as seguintes interfaces:

```C#
namespace solidInCsharp.Repository
{
    public interface IReadRepository<T> where T : class  
    {
        public IEnumerable<T> ListAll();
    }

    public interface IWriteRepository<T> where T : class  
    {
        public void Add(T item);
        public void Remove(T item);
        public void Update(T item);

    }

    public interface IReadWriteRepository<T>: IReadRepository<T>, IWriteRepository<T> where T : class  
    {
    }

    public interface IUsuarioRepository: IReadWriteRepository<Usuario>
    {
        public Usuario ObterUsuario(string Email);
    }

    public interface IProdutoRepository: IReadRepository<Produto>
    {

    }
}

```

Também alteramos as implementações de repositório para usarem essas interfaces, atualizamos os serviços para usarem as novas interfaces e atualizamos o Startup para permitir a injeção de dependencias.

Desta forma, os clientes dos nossos repositórios tem acesso apenas aos métodos que queremos expor para eles.

Dependency Inversion Principle
==============================

O princípio de inversão de dependencia diz que nossas classes sempre devem depender de abstrações e não de concretos.

Seguindo esse principio favorecemos vários pontos importantes nos projetos, como a melhoria de manutenibilidade, testabilidade e extensabilidade do código.

Nosso projeto viola esse princípio em vários pontos, que listamos a seguir:

* Controllers dependem de services diretamente
* Services tem dependencia de outros services diretamente
* Services constroem outros services

Para resolução destes ponteos, devemos criar interfaces e em .Net o meio mais fácil é fazer a injeção de dependencias. Por exemplo, na classe UsuarioService:

```C#
namespace solidInCsharp.Service
{
    public class UsuarioService
    {
        private IUsuarioRepository repository;

        public UsuarioService(IUsuarioRepository repository){ 
            this.repository = repository;
        }

        public void CriarUsuario(string Email, string Name, string Password) {
            var user = this.repository.ObterUsuario(Email);
            if (user != null) {
                throw new Exception("Erro, usuário já existe");
            }
            user = new Usuario() { Email = Email, Nome = Name, Senha = new CriptografiaService().CriptografarSenha(Password)};
            this.repository.Add(user);
        }

        public string Login(string Email,  string Password) {
            var user = this.repository.ObterUsuario(Email);
            if (user == null || !new CriptografiaService().ValidarSenha(user.Senha, Password)) {
                throw new Exception("Erro, usuário ou senha incorreto");
            }
            return new JWTService().GerarToken(user);
        }


    }
}
```

Nesta classe, precisamos isolar a criação do JWTService e CriptoGrafiaService, além de criar uma interface para ela que permitirá que os controllers não tenham uma dependencia direta da implementação.

A implementação desta classe fica desta forma:

```C#
namespace solidInCsharp.Service
{
    public class UsuarioService: IUsuarioService
    {
        private IUsuarioRepository repository;

        private ICriptografiaService criptografiaService;

        private IJWTService jWTService;

        public UsuarioService(IUsuarioRepository repository, ICriptografiaService criptografia, IJWTService jwt){ 
            this.repository = repository;
            this.criptografiaService = criptografia;
            this.jWTService = jwt;
        }

        public void CriarUsuario(string Email, string Name, string Password) {
            var user = this.repository.ObterUsuario(Email);
            if (user != null) {
                throw new Exception("Erro, usuário já existe");
            }
            user = new Usuario() { Email = Email, Nome = Name, Senha = criptografiaService.CriptografarSenha(Password)};
            this.repository.Add(user);
        }

        public string Login(string Email,  string Password) {
            var user = this.repository.ObterUsuario(Email);
            if (user == null || !criptografiaService.ValidarSenha(user.Senha, Password)) {
                throw new Exception("Erro, usuário ou senha incorreto");
            }
            return jWTService.GerarToken(user);
        }


    }
}
```

Por fim, fazemos ajuste na classe de Startup para permitir a injeção de dependencias, e nosso projeto está concluído, seguindo os 5 princípios do SOLID!
