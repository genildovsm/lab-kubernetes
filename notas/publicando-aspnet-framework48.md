# Estrutura do cenário

Aplicação WebApp com Biblioteca de classe referenciada.

~~~
MinhaSolution.sln
 ├── MinhaWebApp (ASP.NET Framework 4.8)
 └── MinhaBiblioteca (Class Library)
~~~

- `MinhaBiblioteca` é referenciada por `MinhaWebApp`
- Não é uma aplicação executável
- Gera apenas uma `.dll`
  
### Publish no ASP.NET Framework

~~~cmd
msbuild MinhaWebApp.csproj ^
  /p:Configuration=Release ^
  /p:DeployOnBuild=true ^
  /p:PublishProfile=FolderProfile ^
  /p:PublishUrl=C:\out
~~~

O `^` apenas **continua a linha no CMD.**. O MSBuild recebe tudo **como um único comando.**

~~~
Linux images → Bash → \
Windows images → CMD → ^
Windows com PowerShell → `
~~~

O MSBuild faz automaticamente:

1. Compila MinhaBiblioteca
2. Gera MinhaBiblioteca.dll
3. Copia a DLL para: **C:\out\bin**
4. Publica somente a WebApp, **já com a biblioteca incluída**

~~~
C:\out\
 ├── MinhaWebApp.dll
 ├── web.config
 ├── bin\
 │   ├── MinhaBiblioteca.dll
 │   ├── Newtonsoft.Json.dll
 │   └── outras dependências
 ├── Views\
 └── ...
~~~

### Resumo

| Parâmetro                 | Função                   |
| --- | --- |
| `msbuild MinhaApi.csproj` | Compila o projeto        |
| `Configuration=Release`   | Build otimizado          |
| `DeployOnBuild=true`      | Habilita publish         |
| `PublishProfile`          | Define o tipo de publish |
| `PublishUrl`              | Pasta final do artefato  |


### Em Docker

Stage do build

~~~dockerfile
RUN msbuild MinhaWebApp.csproj ^
    /p:Configuration=Release ^
    /p:DeployOnBuild=true ^
    /p:PublishProfile=FolderProfile ^
    /p:PublishUrl=C:\out
~~~

### Stage final

~~~dockerfile
COPY --from=build C:/out/ /inetpub/wwwroot
~~~

### Dockerfile — Multi-stage (.NET Framework 4.8)

~~~dockerfile
# ================================
# STAGE 1 — BUILD / PUBLISH
# ================================
FROM mcr.microsoft.com/dotnet/framework/sdk:4.8 AS build

WORKDIR /src

# Copia tudo
COPY . .

# Restaura dependências
RUN nuget restore

# Publica em Release
RUN msbuild MinhaApi.csproj ^
    /p:Configuration=Release ^
    /p:DeployOnBuild=true ^
    /p:PublishProfile=FolderProfile ^
    /p:PublishUrl=C:\out

# ================================
# STAGE 2 — RUNTIME (IIS)
# ================================
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8

WORKDIR /inetpub/wwwroot

# Remove site padrão
RUN powershell -Command `
    Remove-Item -Recurse -Force * 

# Copia somente o publish
COPY --from=build C:/out/ .

# Porta padrão do IIS
EXPOSE 80
~~~

### Observação importante

- ASP.NET Framework 4.8 só funciona em Windows containers
- Multi-stage aqui reduz lixo de build, não o tamanho final (IIS é obrigatório)

## Informações sobre Build e Publish

### Usando Solution (.sln) — mais comum

~~~sh
dotnet build MinhaSolution.sln -c Release
dotnet test MinhaSolution.sln
dotnet publish MinhaWebApp/MinhaWebApp.csproj -c Release
~~~

O que acontece:

- build → compila todos os projetos
- test → executa somente projetos de teste
- publish → publica apenas a WebApp

📌 Boa prática:  
👉 Build/Test na solution  
👉 Publish apenas no projeto web  

`dotnet test` é o comando que **aciona a execução dos testes unitários**.  
A solution (.sln) apenas informa onde estão os projetos de teste.

Quando apontar para o `.csproj`

~~~sh
dotnet test tests/MinhaWebApp.Tests/MinhaWebApp.Tests.csproj
~~~

- Usar quando:
  - Quer rodar um conjunto específico
  - Pipeline mais rápido
  - Debug de falha específica

### Usando caminho do projeto (.csproj)

~~~sh
dotnet build src/MinhaWebApp/MinhaWebApp.csproj -c Release
dotnet test tests/MinhaWebApp.Tests/MinhaWebApp.Tests.csproj
dotnet publish src/MinhaWebApp/MinhaWebApp.csproj -c Release
~~~

✔ Mais controle  
✔ Ideal para CI/CD  

### Usando diretório atual (sem informar caminho)

~~~sh
cd src/MinhaWebApp
dotnet build -c Release
dotnet test
dotnet publish -c Release
~~~

- A CLI procura automaticamente:
  - .csproj  
  - .sln

⚠️ Perigoso em pipelines, pois depende do cwd  

### Informando pasta de saída do publish

Recomendado para Docker / CI.

~~~sh
dotnet publish src/MinhaWebApp/MinhaWebApp.csproj \
  -c Release \
  -o ./out
~~~

Resultado

~~~
out/
 ├── MinhaWebApp.dll
 ├── web.config
 ├── bin/
 └── ...
~~~

### Com `--no-build` (pipeline otimizado)

~~~sh
dotnet build MinhaSolution.sln -c Release
dotnet test MinhaSolution.sln
dotnet publish src/MinhaWebApp/MinhaWebApp.csproj -c Release --no-build -o ./out
~~~

✔ Evita recompilação  
✔ Build já validado  

## Dockerfile para ASP.NET Core 8 Web API 

Deploy em pod do kubernetes.

~~~yaml
# ==============================
# Build
# ==============================
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copia csproj e restaura dependências
COPY *.csproj .
RUN dotnet restore

# Copia o restante do código
COPY . .
RUN dotnet publish -c Release -o /app/publish

# ==============================
# Runtime
# ==============================
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app

# Configura a porta correta para Kubernetes
ENV ASPNETCORE_URLS=http://+:5000

# Copia os binários publicados
COPY --from=build /app/publish .

# Porta informativa (não expõe nada sozinho)
EXPOSE 5000

# Executa a aplicação
ENTRYPOINT ["dotnet", "MinhaApi.dll"]
~~~

### Diferenças importantes 

| ASP.NET Core       | ASP.NET Framework |
| ------------------ | ----------------- |
| Linux ou Windows   | Apenas Windows    |
| Kestrel            | IIS obrigatório   |
| Porta configurável | Porta fixa (80)   |
| ASPNETCORE_URLS    | não existe        |
| Leve               | Mais pesado       |

