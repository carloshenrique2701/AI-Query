# AIQuery

Aplicacao web que transforma perguntas em linguagem natural em consultas SQL somente de leitura. O sistema coleta os metadados do banco configurado pelo usuario, envia a estrutura para o Google Gemini, valida a consulta gerada e apresenta o resultado em HTML.

O projeto foi desenvolvido como trabalho de conclusao de curso. A arquitetura usa JDBC para conversar com diferentes bancos, mas a presenca de um driver JDBC nao significa que todos os recursos estejam igualmente validados em cada banco.

## Status de suporte

| Banco | Driver no projeto | Nivel atual | URL JDBC recomendada | Observacoes |
| --- | --- | --- | --- | --- |
| MySQL | `com.mysql:mysql-connector-j` | Principal | `jdbc:mysql://host:3306/banco` | E o banco usado pelo ambiente Docker e pelo banco interno da aplicacao. |
| MariaDB | `org.mariadb.jdbc:mariadb-java-client` | Principal | `jdbc:mariadb://host:3306/banco` | O driver esta incluido e o formato curto `mariadb://` e normalizado. |
| PostgreSQL | `org.postgresql:postgresql` | Principal | `jdbc:postgresql://host:5432/banco` | O fluxo JDBC esta preparado; schemas diferentes de `public` devem ser validados no ambiente alvo. |
| SQLite | `org.xerial:sqlite-jdbc` | Parcial | `jdbc:sqlite:/caminho/arquivo.db` | Use a URL JDBC completa. Normalmente nao ha usuario ou senha. |
| Oracle | `com.oracle.database.jdbc:ojdbc11` | Parcial | `jdbc:oracle:thin:@//host:1521/servico` | Use os formatos oficiais do driver. Catalogos, schemas e sintaxe de limite exigem validacao especifica. |
| SQL Server | `com.microsoft.sqlserver:mssql-jdbc` | Parcial | `jdbc:sqlserver://host:1433;databaseName=banco` | O driver e a normalizacao existem, mas este nao e um dos bancos principais testados. |

Essa tabela descreve o estado do codigo, nao uma garantia de compatibilidade universal. O funcionamento tambem depende do driver, das permissoes, do formato de metadados e da sintaxe SQL aceita pelo banco.

## Como funciona

1. O usuario configura a conexao do banco externo.
2. A aplicacao normaliza a URL, carrega o driver informado quando necessario e testa a conexao com `DriverManager`.
3. Depois da validacao, as credenciais sao armazenadas de forma criptografada.
4. Antes de responder uma pergunta, a aplicacao le tabelas e colunas usando `DatabaseMetaData`.
5. O modelo de IA recebe o dialeto identificado e um dicionario de metadados, mas nao recebe acesso direto ao banco.
6. A consulta retornada pela IA e analisada pelo JSQLParser.
7. Somente instrucoes cujo tipo principal seja `SELECT` podem ser executadas.
8. O resultado e enviado a uma segunda etapa de IA para formatacao em HTML.

O filtro de `SELECT` reduz o risco de operacoes de escrita, mas nao substitui controle de permissao no banco, limites de custo e isolamento de credenciais.

## Tecnologias

- Java 25, Spring Boot e Maven.
- Spring Data JPA e Spring JDBC para o banco interno da aplicacao.
- JDBC e `DatabaseMetaData` para conexao e leitura de metadados dos bancos externos.
- JSQLParser para verificar se a consulta gerada e uma consulta de leitura.
- Google Gemini API para interpretar perguntas, gerar SQL e formatar resultados.
- Spring Security e JWT para autenticacao.
- BCrypt para senhas de usuarios e AES para credenciais dos bancos externos.
- HTML, CSS, JavaScript e Thymeleaf na interface.
- Docker Compose para a aplicacao e os bancos MySQL de desenvolvimento.

## Arquitetura de dados

O ambiente Docker possui dois bancos com finalidades diferentes:

- `db_manager`: banco interno da aplicacao, usado para usuarios e credenciais.
- `db_analise`: banco de demonstracao usado como banco externo de analise.
- `app`: aplicacao Spring Boot.

O banco interno usa MySQL por padrao. Em testes automatizados, o perfil de teste usa H2 em memoria. Os bancos externos configurados pelos usuarios sao acessados diretamente pelo JDBC e nao precisam ser adicionados ao Compose.

## Pre-requisitos

- Java 25.
- Docker Desktop com Docker Compose.
- Uma chave da API Google Gemini.
- Portas `8080`, `3306` e `3307` disponiveis quando o ambiente Docker completo for usado.

## Configuracao com Docker

Defina as variaveis abaixo no ambiente que executara o Compose ou em um arquivo `.env`:

```env
GOOGLE_GEMINI_API_KEY=sua-chave-do-gemini
JWT_SECRET=um-segredo-com-pelo-menos-32-caracteres
SECRET=outro-segredo-com-pelo-menos-32-caracteres
```

O `docker-compose.yml` define estas variaveis para a aplicacao:

```text
DB_URL=jdbc:mysql://db_manager:3306/aiquery_manager
DB_USERNAME=root
DB_PASSWORD=(atualmente vazio no Compose; deve ser corrigido para root123)
DB_ANALISE_URL=jdbc:mysql://db_analise:3306/rede_lojas_roupas
```

Dentro da rede Docker, a aplicacao deve usar os nomes `db_manager` e `db_analise` e a porta interna `3306`. Do computador hospedeiro, o banco interno fica em `localhost:3306` e o banco de analise em `localhost:3307`.

> Importante: atualmente `DB_PASSWORD` esta vazio no Compose, enquanto `MYSQL_ROOT_PASSWORD` e `root123`. Isso pode impedir a aplicacao de autenticar no banco interno. Antes de usar o ambiente, defina `DB_PASSWORD: root123` no servico `app` ou forneca um valor equivalente por variavel de ambiente. Em producao, use segredos fora do arquivo e nunca reutilize essas credenciais de exemplo.

Para executar o ambiente completo em containers:

```powershell
docker compose up --build
```

Acesse `http://localhost:8080` depois que os servicos estiverem saudaveis. O banco de demonstracao e inicializado com o arquivo `test_db.sql`.

Comandos uteis:

```powershell
docker compose ps
docker compose logs app
docker compose config
```

## Configurando um banco externo

Na tela de configuracao, informe a URL, o usuario e a senha. O sistema testa a conexao antes de salvar as credenciais.

### URLs aceitas pelo normalizador

URLs JDBC completas sao a opcao recomendada:

```text
jdbc:mysql://localhost:3306/meubanco
jdbc:mariadb://localhost:3306/meubanco
jdbc:postgresql://localhost:5432/meubanco
jdbc:sqlite:/var/dados/app.db
jdbc:oracle:thin:@//localhost:1521/meuservico
jdbc:sqlserver://localhost:1433;databaseName=meubanco
```

Para MySQL, MariaDB, PostgreSQL e SQL Server, o sistema tambem adiciona `jdbc:` quando recebe estes formatos curtos:

```text
mysql://host:3306/banco
mariadb://host:3306/banco
postgresql://host:5432/banco
sqlserver://host:1433;databaseName=banco
```

SQLite deve ser informado com a URL JDBC completa. Para Oracle, use os formatos oficiais do driver, como `jdbc:oracle:thin:@host:1521:SID` ou `jdbc:oracle:thin:@//host:1521/servico`; nao use `oracle://` como formato recomendado.

O sistema tambem consegue extrair usuario e senha de uma URL com informacoes de autoridade, por exemplo:

```text
postgresql://usuario:senha@host:5432/banco
```

Por seguranca, prefira sempre os campos separados. Senhas colocadas em URLs podem aparecer no historico do navegador, logs, mensagens de erro ou ferramentas de monitoramento antes de serem removidas pela aplicacao.

## JDBC e compatibilidade entre bancos

JDBC fornece uma API comum para abrir conexoes e consultar metadados, mas nao elimina as diferencas entre bancos. Neste projeto:

- `DriverManager` abre a conexao usando a URL, o usuario e a senha normalizados.
- `DatabaseMetaData` e usado para descobrir tabelas e colunas.
- O nome do banco e obtido preferencialmente de `Connection.getCatalog()` e, quando necessario, de `Connection.getSchema()` ou da URL.
- Oracle costuma depender mais de schemas do que de catalogs; por isso a extracao de estrutura precisa ser validada em cada ambiente Oracle.
- SQLite e orientado a arquivo e nao segue o mesmo modelo de catalogo, usuario e schema dos servidores relacionais.
- Uma consulta valida em um banco pode nao ser valida em outro. O uso de `LIMIT 20`, por exemplo, nao e universal e nao deve ser assumido para Oracle.
- O SQL gerado pela IA ainda depende de nomes reais de tabelas, colunas, permissoes e recursos do dialeto identificado.

Portanto, a aplicacao e adaptavel a bancos com drivers JDBC compativeis, mas nao oferece compatibilidade automatica com qualquer banco sem testes de conexao, metadados e consultas no ambiente alvo.

## Limitacoes conhecidas

- O fluxo de consulta aceita somente `SELECT` como operacao principal.
- A validacao com JSQLParser nao garante que uma consulta seja barata, correta ou segura contra todos os cenarios de abuso.
- Consultas geradas podem usar sintaxe especifica de outro dialeto.
- A descoberta de schemas e tabelas pode variar entre MySQL, PostgreSQL, Oracle e SQLite.
- Oracle e SQLite possuem suporte parcial e devem ser tratados como integracoes experimentais ate receberem testes de integracao dedicados.
- A qualidade da resposta depende dos metadados extraidos e da interpretacao do modelo de IA.
- O banco interno e o banco externo de analise do Compose usam credenciais de desenvolvimento e nao devem ser expostos em producao.

## Desenvolvimento e testes

Use o Maven Wrapper para executar a aplicacao e os testes:

```powershell
.\mvnw.cmd test -Dspring.profiles.active=test
```

O perfil `test` usa H2 em memoria. Para conferir os drivers declarados:

```powershell
.\mvnw.cmd dependency:tree "-Dincludes=com.mysql:mysql-connector-j,org.mariadb.jdbc:mariadb-java-client,org.postgresql:postgresql,org.xerial:sqlite-jdbc,com.oracle.database.jdbc:ojdbc11,com.microsoft.sqlserver:mssql-jdbc"
```

Ainda sao recomendados testes de integracao separados para:

- normalizacao das URLs e credenciais embutidas;
- conexao real com MySQL, MariaDB e PostgreSQL;
- extracao de schemas em SQLite e Oracle;
- geracao de limite de resultados conforme o dialeto;
- confirmacao de que as credenciais do Compose permanecem sincronizadas.

## Seguranca

- Mantenha `GOOGLE_GEMINI_API_KEY`, `JWT_SECRET` e `SECRET` fora do controle de versao.
- Troque os segredos padrao antes de qualquer ambiente compartilhado ou de producao.
- Use um usuario de banco com as menores permissoes necessarias para consultas externas.
- Nao coloque senhas em URLs quando os campos separados estiverem disponiveis.
- Considere limites de tempo, quantidade de linhas e custo das consultas antes de disponibilizar a aplicacao para dados reais.

## Licenca

Nenhuma licenca foi definida no projeto ate o momento.
