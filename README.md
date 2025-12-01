# Docker-EF_CORE-SQL-C---Esempio-"libreria-libri-autore"
applicazione C# completa con:  
- Entity Framework Core ORM 
- MySQL su Docker 
- CRUD completo (Create, Read, Update, Delete)
- Menu interattivo console 
- Database con dati di esempio
////////////////////////////////////////////////////////////////////////////////////////////////////
🔧 PREREQUISITI
Software Necessari:
✅ .NET SDK 8.0+

✅ Docker Desktop (Windows/Mac) o Docker Engine (Linux)

✅ Visual Studio Code o Visual Studio 2022

PASSO 1: SETUP PROGETTO

Attenzione se usate bash o powershell perchè cambiano semplicemente i comandi in alcuni casi!
Creaiamo la struttura cartelle:
mkdir LibreriaEF
cd LibreriaEF
mkdir Models Data Services

nella cartella LibreriaEF da powershell di visual studio code, digitiamo : 'dotnet new console'
poi:
Configura file progetto (LibreriaEF.csproj) : importante fare un controllo delle versioni e specificare bene quale usare:
//////////////////////////////////////////////////////////////////////////////////////////////////
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageReference Include="Pomelo.EntityFrameworkCore.MySql" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>
</Project>
///////////////////////////////////////////////////////////////////////////////////////////////////
Creiamo un file docker-compose.yml
///////////////////////////////////////////////////////////////////////////////////////////////////
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql_libreria
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: LibreriaDB
      MYSQL_USER: csharp_user
      MYSQL_PASSWORD: Csharp123!
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
volumes:
  mysql_data:

  
  ///////////////////////////////////////////////////////////////////////////////////////////////////
  Crea init.sql : qui creiamo la nostra tabella e la popoliamo
  ///////////////////////////////////////////////////////////////////////////////////////////////////
  CREATE DATABASE IF NOT EXISTS LibreriaDB;
USE LibreriaDB;

CREATE TABLE Autori (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    Nome VARCHAR(100) NOT NULL,
    Cognome VARCHAR(100) NOT NULL,
    Email VARCHAR(150)
);

CREATE TABLE Libri (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    Titolo VARCHAR(200) NOT NULL,
    ISBN VARCHAR(13) UNIQUE NOT NULL,
    Prezzo DECIMAL(10,2),
    AutoreId INT,
    FOREIGN KEY (AutoreId) REFERENCES Autori(Id)
);

INSERT INTO Autori (Nome, Cognome, Email) VALUES
('Umberto', 'Eco', 'umberto.eco@example.com'),
('Italo', 'Calvino', 'italo.calvino@example.com');

INSERT INTO Libri (Titolo, ISBN, Prezzo, AutoreId) VALUES
('Il Nome della Rosa', '9788804680385', 15.90, 1),
('Il Barone Rampante', '9788804667669', 12.99, 2);

 ///////////////////////////////////////////////////////////////////////////////////////////////////
  Avvia MySQL con Docker
 ///////////////////////////////////////////////////////////////////////////////////////////////////
 # Terminale nella cartella LibreriaEF
docker-compose up -d   //qui stiamo verificando

# Verifica che sia in esecuzione
docker ps

# Dovresti vedere:
# CONTAINER ID   IMAGE     PORTS                    NAMES
# abc123        mysql:8.0  0.0.0.0:3306->3306/tcp   mysql_libreria
 ///////////////////////////////////////////////////////////////////////////////////////////////////
 Crea appsettings.json : qui specifichiamo la connessione con il database
 ///////////////////////////////////////////////////////////////////////////////////////////////////
 {
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Port=3306;Database=LibreriaDB;User=csharp_user;Password=Csharp123!;CharSet=utf8mb4"
  }
}
///////////////////////////////////////////////////////////////////////////////////////////////////
Crea Models/Autore.cs
///////////////////////////////////////////////////////////////////////////////////////////////////
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace LibreriaEF.Models
{
    [Table("Autori")]
    public class Autore
    {
        [Key]
        [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
        public int Id { get; set; }

        [Required]
        [MaxLength(100)]
        public string Nome { get; set; } = string.Empty;

        [Required]
        [MaxLength(100)]
        public string Cognome { get; set; } = string.Empty;

        [MaxLength(150)]
        public string? Email { get; set; }

        // Navigazione
        public virtual ICollection<Libro> Libri { get; set; } = new List<Libro>();

        [NotMapped]
        public string NomeCompleto => $"{Nome} {Cognome}";
    }
}
///////////////////////////////////////////////////////////////////////////////////////////////////
Crea Models/Libro.cs
///////////////////////////////////////////////////////////////////////////////////////////////////
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace LibreriaEF.Models
{
    [Table("Libri")]
    public class Libro
    {
        [Key]
        [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
        public int Id { get; set; }

        [Required]
        [MaxLength(200)]
        public string Titolo { get; set; } = string.Empty;

        [Required]
        [MaxLength(13)]
        public string ISBN { get; set; } = string.Empty;

        [Column(TypeName = "decimal(10,2)")]
        public decimal Prezzo { get; set; }

        public int? AutoreId { get; set; }

        // Navigazione
        [ForeignKey("AutoreId")]
        public virtual Autore? Autore { get; set; }
    }
}
///////////////////////////////////////////////////////////////////////////////////////////////////
Crea Data/ApplicationDbContext.cs
///////////////////////////////////////////////////////////////////////////////////////////////////
using Microsoft.EntityFrameworkCore;
using LibreriaEF.Models;

namespace LibreriaEF.Data
{
    public class ApplicationDbContext : DbContext
    {
        public DbSet<Autore> Autori { get; set; }
        public DbSet<Libro> Libri { get; set; }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            if (!optionsBuilder.IsConfigured)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();

                var connectionString = configuration.GetConnectionString("DefaultConnection");
                optionsBuilder.UseMySql(connectionString, 
                    ServerVersion.AutoDetect(connectionString));
            }
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Configura relazione uno-a-molti
            modelBuilder.Entity<Libro>()
                .HasOne(l => l.Autore)
                .WithMany(a => a.Libri)
                .HasForeignKey(l => l.AutoreId)
                .OnDelete(DeleteBehavior.SetNull);
        }
    }
}
///////////////////////////////////////////////////////////////////////////////////////////////////
Crea Program.cs
///////////////////////////////////////////////////////////////////////////////////////////////////
using Microsoft.EntityFrameworkCore;
using LibreriaEF.Data;
using LibreriaEF.Models;

class Program
{
    static async Task Main(string[] args)
    {
        Console.WriteLine("=== GESTIONE LIBRERIA ===");

        using var db = new ApplicationDbContext();
        
        // Crea database se non esiste
        await db.Database.EnsureCreatedAsync();
        Console.WriteLine("Database pronto!");

        bool continua = true;
        while (continua)
        {
            Console.WriteLine("\n=== MENU ===");
            Console.WriteLine("1. Visualizza libri");
            Console.WriteLine("2. Aggiungi libro");
            Console.WriteLine("3. Cerca libro");
            Console.WriteLine("4. Esci");
            Console.Write("Scelta: ");

            switch (Console.ReadLine())
            {
                case "1":
                    await VisualizzaLibri(db);
                    break;
                case "2":
                    await AggiungiLibro(db);
                    break;
                case "3":
                    await CercaLibro(db);
                    break;
                case "4":
                    continua = false;
                    break;
                default:
                    Console.WriteLine("Scelta non valida!");
                    break;
            }
        }
    }

    static async Task VisualizzaLibri(ApplicationDbContext db)
    {
        var libri = await db.Libri
            .Include(l => l.Autore)
            .ToListAsync();

        Console.WriteLine("\n=== LIBRI IN CATALOGO ===");
        foreach (var libro in libri)
        {
            Console.WriteLine($"{libro.Titolo} - {libro.Autore?.NomeCompleto} - €{libro.Prezzo}");
        }
    }

    static async Task AggiungiLibro(ApplicationDbContext db)
    {
        Console.Write("Titolo: ");
        var titolo = Console.ReadLine();
        
        Console.Write("ISBN: ");
        var isbn = Console.ReadLine();
        
        Console.Write("Prezzo: ");
        var prezzo = decimal.Parse(Console.ReadLine());

        var libro = new Libro
        {
            Titolo = titolo,
            ISBN = isbn,
            Prezzo = prezzo
        };

        db.Libri.Add(libro);
        await db.SaveChangesAsync();
        
        Console.WriteLine($"Libro '{titolo}' aggiunto!");
    }

    static async Task CercaLibro(ApplicationDbContext db)
    {
        Console.Write("Cerca titolo: ");
        var ricerca = Console.ReadLine();

        var libri = await db.Libri
            .Where(l => l.Titolo.Contains(ricerca))
            .Include(l => l.Autore)
            .ToListAsync();

        Console.WriteLine($"\nTrovati {libri.Count} libri:");
        foreach (var libro in libri)
        {
            Console.WriteLine($"- {libro.Titolo}");
        }
    }
}
///////////////////////////////////////////////////////////////////////////////////////////////////

4.1 Comandi terminale: ESECUZIONE PROGRAMMA

///////////////////////////////////////////////////////////////////////////////////////////////////
# 1. Assicurati che MySQL sia attivo
docker-compose up -d

# 2. Installa pacchetti NuGet
dotnet restore

# 3. Compila progetto
dotnet build

# 4. Esegui applicazione
dotnet run

# 5. Dovresti vedere:
# === GESTIONE LIBRERIA ===
# Database pronto!
# 
# === MENU ===
# 1. Visualizza libri
# 2. Aggiungi libro
# 3. Cerca libro
# 4. Esci
# Scelta:
  

