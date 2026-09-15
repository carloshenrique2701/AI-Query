# AIQuery

Aplicação web que transforma perguntas em linguagem natural em consultas SQL somente de leitura. O sistema coleta os metadados do banco configurado pelo usuário, envia a estrutura para o Google Gemini, valida a consulta gerada e apresenta o resultado em HTML.

O projeto foi desenvolvido como trabalho de conclusão de curso. A arquitetura usa JDBC para conversar com diferentes bancos, mas a presença de um driver JDBC não significa que todos os recursos estejam igualmente validados em cada banco.

![Tela principal do AIQuery](contents/img1.png)

## Status de suporte

| Banco | Driver no projeto | Nível atual | URL JDBC recomendada | Observações |
| --- | --- | --- | --- | --- |
| MySQL | `com.mysql:mysql-connector-j` | Principal | `jdbc:mysql://host:3306/banco` | É o banco usado pelo ambiente Docker e pelo banco interno da aplicação. |
| MariaDB | `org.mariadb.jdbc:mariadb-java-client` | Principal | `jdbc:mariadb://host:3306/banco` | O driver está incluído e o formato curto `mariadb://` é normalizado. |
| PostgreSQL | `org.postgresql:postgresql` | Principal | `jdbc:postgresql://host:5432/banco` | O fluxo JDBC está preparado; schemas diferentes de `public` devem ser validados no ambiente-alvo. |
| SQLite | `org.xerial:sqlite-jdbc` | Parcial | `jdbc:sqlite:/caminho/arquivo.db` | Use a URL JDBC completa. Normalmente não há usuário ou senha. |
| Oracle | `com.oracle.database.jdbc:ojdbc11` | Parcial | `jdbc:oracle:thin:@//host:1521/servico` | Use os formatos oficiais do driver. Catálogos, schemas e sintaxe de limite exigem validação específica. |
| SQL Server | `com.microsoft.sqlserver:mssql-jdbc` | Parcial | `jdbc:sqlserver://host:1433;databaseName=banco` | O driver e a normalização existem, mas este não é um dos bancos principais testados. |

Essa tabela descreve o estado do código, não uma garantia de compatibilidade universal. O funcionamento também depende do driver, das permissões, do formato de metadados e da sintaxe SQL aceita pelo banco.

## Como funciona

1. O usuário configura a conexão do banco externo.
2. A aplicação normaliza a URL, carrega o driver informado quando necessário e testa a conexão com `DriverManager`.
3. Depois da validação, as credenciais são armazenadas de forma criptografada.
4. Antes de responder uma pergunta, a aplicação lê tabelas e colunas usando `DatabaseMetaData`.
5. O modelo de IA recebe o dialeto identificado e um dicionário de metadados, mas não recebe acesso direto ao banco.
6. A consulta retornada pela IA é analisada pelo JSQLParser.
7. Somente instruções cujo tipo principal seja `SELECT` podem ser executadas.
8. O resultado é enviado a uma segunda etapa de IA para formatação em HTML.

O filtro de `SELECT` reduz o risco de operações de escrita, mas não substitui o controle de permissões no banco, os limites de custo e o isolamento de credenciais.

## Tecnologias

- Java 25, Spring Boot e Maven.
- Spring Data JPA e Spring JDBC para o banco interno da aplicacao.
- JDBC e `DatabaseMetaData` para conexao e leitura de metadados dos bancos externos.
- JSQLParser para verificar se a consulta gerada e uma consulta de leitura.
- Google Gemini API para interpretar perguntas, gerar SQL e formatar resultados.
- Spring Security e JWT para autenticação.
- BCrypt para senhas de usuários e AES para credenciais dos bancos externos.
- HTML, CSS, JavaScript e Thymeleaf na interface.
- Docker Compose para a aplicacao e os bancos MySQL de desenvolvimento.

## Arquitetura de dados

O ambiente Docker possui dois bancos com finalidades diferentes:

- `db_manager`: banco interno da aplicação, usado para usuários e credenciais.
- `db_analise`: banco de demonstração usado como banco externo de análise.
- `app`: aplicação Spring Boot.

O banco interno usa MySQL por padrão. Em testes automatizados, o perfil de teste usa H2 em memória. Os bancos externos configurados pelos usuários são acessados diretamente pelo JDBC e não precisam ser adicionados ao Compose.

## Pré-requisitos

- Java 25.
- Docker Desktop com Docker Compose.
- Uma chave da API Google Gemini.
- Portas `8080`, `3306` e `3307` disponíveis quando o ambiente Docker completo for usado.

## Configuracao com Docker

Defina as variaveis abaixo no ambiente que executara o Compose ou em um arquivo `.env`:

```env
GOOGLE_GEMINI_API_KEY=sua-chave-do-gemini
JWT_SECRET=um-segredo-com-pelo-menos-32-caracteres
SECRET=outro-segredo-com-pelo-menos-32-caracteres
```

O `docker-compose.yml` define estas variáveis para a aplicação:

```text
DB_URL=jdbc:mysql://db_manager:3306/aiquery_manager
DB_USERNAME=root
DB_PASSWORD=(atualmente vazio no Compose; deve ser corrigido para root123)
DB_ANALISE_URL=jdbc:mysql://db_analise:3306/rede_lojas_roupas
```

Dentro da rede Docker, a aplicação deve usar os nomes `db_manager` e `db_analise` e a porta interna `3306`. Do computador hospedeiro, o banco interno fica em `localhost:3306` e o banco de análise em `localhost:3307`.

> Importante: atualmente `DB_PASSWORD` está vazio no Compose, enquanto `MYSQL_ROOT_PASSWORD` é `root123`. Isso pode impedir a aplicação de autenticar no banco interno. Antes de usar o ambiente, defina `DB_PASSWORD: root123` no serviço `app` ou forneça um valor equivalente por variável de ambiente. Em produção, use segredos fora do arquivo e nunca reutilize essas credenciais de exemplo.

Para executar o ambiente completo em containers:

```powershell
docker compose up --build
```

Acesse `http://localhost:8080` depois que os serviços estiverem saudáveis. O banco de demonstração é inicializado com o arquivo `test_db.sql`.

Comandos úteis:

```powershell
docker compose ps
docker compose logs app
docker compose config
```

## Configurando um banco externo

Na tela de configuração, informe a URL, o usuário e a senha. O sistema testa a conexão antes de salvar as credenciais.

### URLs aceitas pelo normalizador

URLs JDBC completas são a opção recomendada:

```text
jdbc:mysql://localhost:3306/meubanco
jdbc:mariadb://localhost:3306/meubanco
jdbc:postgresql://localhost:5432/meubanco
jdbc:sqlite:/var/dados/app.db
jdbc:oracle:thin:@//localhost:1521/meuservico
jdbc:sqlserver://localhost:1433;databaseName=meubanco
```

Para MySQL, MariaDB, PostgreSQL e SQL Server, o sistema também adiciona `jdbc:` quando recebe estes formatos curtos:

```text
mysql://host:3306/banco
mariadb://host:3306/banco
postgresql://host:5432/banco
sqlserver://host:1433;databaseName=banco
```

SQLite deve ser informado com a URL JDBC completa. Para Oracle, use os formatos oficiais do driver, como `jdbc:oracle:thin:@host:1521:SID` ou `jdbc:oracle:thin:@//host:1521/servico`; não use `oracle://` como formato recomendado.

O sistema também consegue extrair usuário e senha de uma URL com informações de autoridade, por exemplo:

```text
postgresql://usuario:senha@host:5432/banco
```

Por segurança, prefira sempre os campos separados. Senhas colocadas em URLs podem aparecer no histórico do navegador, logs, mensagens de erro ou ferramentas de monitoramento antes de serem removidas pela aplicação.

## JDBC e compatibilidade entre bancos

JDBC fornece uma API comum para abrir conexões e consultar metadados, mas não elimina as diferenças entre bancos. Neste projeto:

- `DriverManager` abre a conexão usando a URL, o usuário e a senha normalizados.
- `DatabaseMetaData` é usado para descobrir tabelas e colunas.
- O nome do banco é obtido preferencialmente de `Connection.getCatalog()` e, quando necessário, de `Connection.getSchema()` ou da URL.
- Oracle costuma depender mais de schemas do que de catálogos; por isso a extração de estrutura precisa ser validada em cada ambiente Oracle.
- SQLite é orientado a arquivo e não segue o mesmo modelo de catálogo, usuário e schema dos servidores relacionais.
- Uma consulta válida em um banco pode não ser válida em outro. O uso de `LIMIT 20`, por exemplo, não é universal e não deve ser assumido para Oracle.
- O SQL gerado pela IA ainda depende de nomes reais de tabelas, colunas, permissões e recursos do dialeto identificado.

Portanto, a aplicação é adaptável a bancos com drivers JDBC compatíveis, mas não oferece compatibilidade automática com qualquer banco sem testes de conexão, metadados e consultas no ambiente-alvo.

## Limitações conhecidas

- O fluxo de consulta aceita somente `SELECT` como operação principal.
- A validação com JSQLParser não garante que uma consulta seja barata, correta ou segura contra todos os cenários de abuso.
- Consultas geradas podem usar sintaxe específica de outro dialeto.
- A descoberta de schemas e tabelas pode variar entre MySQL, PostgreSQL, Oracle e SQLite.
- Oracle e SQLite possuem suporte parcial e devem ser tratados como integrações experimentais até receberem testes de integração dedicados.
- A qualidade da resposta depende dos metadados extraídos e da interpretação do modelo de IA.
- O banco interno e o banco externo de análise do Compose usam credenciais de desenvolvimento e não devem ser expostos em produção.

## Desenvolvimento e testes

Use o Maven Wrapper para executar a aplicação e os testes:

```powershell
.\mvnw.cmd test -Dspring.profiles.active=test
```

O perfil `test` usa H2 em memória. Para conferir os drivers declarados:

```powershell
.\mvnw.cmd dependency:tree "-Dincludes=com.mysql:mysql-connector-j,org.mariadb.jdbc:mariadb-java-client,org.postgresql:postgresql,org.xerial:sqlite-jdbc,com.oracle.database.jdbc:ojdbc11,com.microsoft.sqlserver:mssql-jdbc"
```

Ainda são recomendados testes de integração separados para:

- normalização das URLs e credenciais embutidas;
- conexão real com MySQL, MariaDB e PostgreSQL;
- extração de schemas em SQLite e Oracle;
- geração de limite de resultados conforme o dialeto;
- confirmação de que as credenciais do Compose permanecem sincronizadas.

## Segurança

- Mantenha `GOOGLE_GEMINI_API_KEY`, `JWT_SECRET` e `SECRET` fora do controle de versao.
- Troque os segredos padrão antes de qualquer ambiente compartilhado ou de produção.
- Use um usuário de banco com as menores permissões necessárias para consultas externas.
- Não coloque senhas em URLs quando os campos separados estiverem disponíveis.
- Considere limites de tempo, quantidade de linhas e custo das consultas antes de disponibilizar a aplicação para dados reais.

## Licença

Nenhuma licença foi definida no projeto até o momento.
