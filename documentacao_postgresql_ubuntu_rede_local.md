# Documentação: PostgreSQL no Ubuntu Server com acesso à rede local

**Objetivo:** instalar e configurar o PostgreSQL no Ubuntu Server para
aceitar conexões de computadores da rede doméstica e permitir a criação
de bancos de dados independentes para diferentes aplicações.

> **Estado registrado neste documento:** Ubuntu Server 26.04 LTS,
> PostgreSQL 18.6, IP local `192.168.1.201`, rede doméstica
> `192.168.1.0/24` e porta TCP `5432`.

------------------------------------------------------------------------

## 1. Arquitetura

-   **Servidor:** Ubuntu Server.
-   **Banco de dados:** PostgreSQL 18.
-   **IP local do servidor:** `192.168.1.201`.
-   **Porta TCP:** `5432`.
-   **Gateway da rede:** `192.168.1.1`.
-   **Interface de rede ativa observada:** `wlx007811009e6c` (Wi-Fi).
-   **Banco inicial:** `appdb`.
-   **Usuário inicial da aplicação:** `app_user`.

O PostgreSQL pode hospedar vários bancos na mesma instância. Eles
compartilham CPU, memória, armazenamento e a mesma porta, mas podem ter
usuários e permissões diferentes.

**Segurança:** esta configuração destina-se à rede local. Não crie
redirecionamento da porta `5432` no roteador para a internet. O endereço
IP pode mudar se o roteador não reservar o endereço para o servidor.

------------------------------------------------------------------------

## 2. Verificar a versão do Ubuntu e a rede

``` bash
lsb_release -a
ip -br addr
ip route
```

Exemplo dos valores identificados:

``` text
Ubuntu 26.04 LTS
IP:      192.168.1.201/24
Gateway: 192.168.1.1
Rede:    192.168.1.0/24
```

-   `ip -br addr` mostra as interfaces e os endereços IP.
-   `ip route` mostra as rotas e o gateway padrão.
-   `192.168.1.0/24` representa a sub-rede local observada; confirme
    esses valores se a rede mudar.

------------------------------------------------------------------------

## 3. Instalar o PostgreSQL

Atualize os índices de pacotes e instale o PostgreSQL:

``` bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Confira o serviço principal:

``` bash
sudo systemctl status postgresql
```

Em sistemas Ubuntu com gerenciamento de clusters, o serviço principal
pode aparecer como `active (exited)`. Isso, por si só, não significa
falha. Verifique o cluster real:

``` bash
pg_lsclusters
```

O resultado esperado para esta instalação é semelhante a:

``` text
Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

Confira a versão:

``` bash
sudo -u postgres psql -c "SELECT version();"
```

O usuário administrativo local `postgres` é acessado pelo
`sudo -u postgres`, sem precisar habilitar login remoto para ele.

------------------------------------------------------------------------

## 4. Configurar o endereço em que o PostgreSQL escuta

Arquivo:

``` bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

Localize `listen_addresses` com `Ctrl + W` no Nano. Configure a linha a
seguir, removendo o `#` inicial caso esteja comentada:

``` conf
listen_addresses = 'localhost,192.168.1.201'
```

Isso permite escuta no loopback local e no endereço IPv4 do servidor. Se
o IP do servidor mudar, atualize a configuração. Não use `'*'` sem
entender e configurar adequadamente as restrições de rede e
autenticação.

No Nano: 1. `Ctrl + O` para salvar. 2. `Enter` para confirmar o nome. 3.
`Ctrl + X` para sair.

------------------------------------------------------------------------

## 5. Criar um usuário e o primeiro banco

Abra o console administrativo:

``` bash
sudo -u postgres psql
```

Execute os comandos **um por vez**:

``` sql
CREATE ROLE app_user LOGIN;
```

Defina a senha de forma interativa:

``` sql
\password app_user
```

Digite a senha duas vezes. **Espere o comando `\password` terminar antes
de inserir outro comando SQL.** Se vários comandos forem colados
enquanto o prompt estiver pedindo a senha, eles podem ser interpretados
como argumentos de `\password` e não serão executados.

Depois, crie o banco:

``` sql
CREATE DATABASE appdb OWNER app_user;
```

Saia do console:

``` sql
\q
```

Se o usuário já existir, não execute `CREATE ROLE app_user LOGIN;`
novamente; use `\password app_user` para redefinir a senha, se
necessário.

Para listar bancos:

``` bash
sudo -u postgres psql -l
```

Para conferir as roles:

``` bash
sudo -u postgres psql -c "\du"
```

------------------------------------------------------------------------

## 6. Autorizar a conexão pela rede no `pg_hba.conf`

Arquivo:

``` bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Adicione ao final a regra para o usuário e banco iniciais:

``` conf
host    appdb    app_user    192.168.1.0/24    scram-sha-256
```

Significado dos campos:

  Campo              Significado
  ------------------ -------------------------------------
  `host`             Conexão TCP/IP
  `appdb`            Banco autorizado por esta regra
  `app_user`         Usuário autorizado
  `192.168.1.0/24`   Endereços IPv4 da rede local
  `scram-sha-256`    Autenticação por senha usando SCRAM

A regra é específica: não autoriza automaticamente todos os bancos nem
todos os usuários. A ordem das regras do `pg_hba.conf` importa; o
PostgreSQL usa a primeira regra correspondente.

Salve o arquivo e recarregue a configuração ou reinicie o serviço:

``` bash
sudo systemctl restart postgresql
```

Confirme o cluster:

``` bash
pg_lsclusters
```

------------------------------------------------------------------------

## 7. Verificar se a porta está escutando

Execute:

``` bash
sudo ss -ltnp | grep 5432
```

No estado observado, o resultado incluía:

``` text
192.168.1.201:5432
127.0.0.1:5432
```

Isso confirma que existe um processo escutando nesses endereços. **Não
confirma sozinho** que qualquer cliente conseguirá entrar: o banco
precisa existir, a autenticação precisa ser válida e a regra de acesso
deve permitir a conexão.

------------------------------------------------------------------------

## 8. Testar localmente usando o IP da rede

Com o banco e a role criados, teste:

``` bash
psql -h 192.168.1.201 -p 5432 -U app_user -d appdb
```

Digite a senha quando solicitada. Se conectar, execute:

``` sql
SELECT current_database(), current_user;
```

Resultado esperado:

``` text
 current_database | current_user
------------------+------------
 appdb            | app_user
```

Para sair:

``` sql
\q
```

A conexão de teste realizada nesta configuração apresentou TLS 1.3. A
mensagem SSL do `psql` confirma que aquela conexão foi criptografada.

### Erro encontrado e solução

Erro:

``` text
FATAL: database "appdb" does not exist
```

Causa: o banco não tinha sido criado. Durante a primeira tentativa, os
comandos `CREATE DATABASE ...` foram digitados enquanto `\password`
ainda estava aguardando a senha, então não foram executados como SQL.

Solução aplicada:

``` bash
sudo -u postgres psql
```

``` sql
CREATE DATABASE appdb OWNER app_user;
\q
```

Depois disso, a conexão ao `appdb` funcionou.

------------------------------------------------------------------------

## 9. Criar bancos adicionais para outros projetos

É possível criar diversos bancos na mesma instância. Uma boa prática é
criar uma role de aplicação para cada projeto e atribuir a ela apenas o
banco necessário.

Exemplo de um segundo projeto chamado `loja_db`.

Entre como administrador:

``` bash
sudo -u postgres psql
```

Crie o usuário:

``` sql
CREATE ROLE loja_user LOGIN;
```

Defina a senha e aguarde a conclusão:

``` sql
\password loja_user
```

Crie o banco:

``` sql
CREATE DATABASE loja_db OWNER loja_user;
```

Saia:

``` sql
\q
```

### Autorizar esse segundo banco na rede

Edite:

``` bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Adicione uma regra específica:

``` conf
host    loja_db    loja_user    192.168.1.0/24    scram-sha-256
```

Recarregue a configuração:

``` bash
sudo systemctl reload postgresql
```

Teste a conexão:

``` bash
psql -h 192.168.1.201 -p 5432 -U loja_user -d loja_db
```

A regra do `appdb` não autoriza automaticamente `loja_db`; cada
combinação de banco, usuário e rede precisa corresponder a uma regra
válida.

### Criar mais bancos

Repita o mesmo padrão, trocando os nomes. Por exemplo:

``` sql
CREATE ROLE projeto2_user LOGIN;
```

Em seguida, execute `\password projeto2_user`, espere terminar e só
então execute:

``` sql
CREATE DATABASE projeto2_db OWNER projeto2_user;
```

Adicione a regra correspondente ao `pg_hba.conf`:

``` conf
host    projeto2_db    projeto2_user    192.168.1.0/24    scram-sha-256
```

Depois, recarregue a configuração. Escolha nomes simples, consistentes e
sem espaços.

------------------------------------------------------------------------

## 10. Conectar a partir de outro computador

No computador cliente, que deve estar conectado à mesma rede doméstica,
use estes parâmetros:

  Campo      Valor para o banco inicial
  ---------- -------------------------------------
  Host       `192.168.1.201`
  Porta      `5432`
  Database   `appdb`
  Usuário    `app_user`
  Senha      A senha configurada para `app_user`

Com o cliente `psql` instalado:

``` bash
psql -h 192.168.1.201 -p 5432 -U app_user -d appdb
```

No Windows PowerShell, o comando é o mesmo, desde que `psql` esteja
instalado e disponível no `PATH`. Também é possível usar um cliente
gráfico, como DBeaver.

Se a conexão falhar, verifique: 1. Se os dois dispositivos estão na
mesma rede e conseguem alcançar o servidor. 2. Se o IP do servidor
continua sendo `192.168.1.201`. 3. Se `pg_lsclusters` mostra `online`.
4. Se `ss -ltnp` mostra a porta `5432` no IP de rede. 5. Se o banco e a
role existem e a senha está correta. 6. Se a regra correspondente está
no `pg_hba.conf`. 7. Se o roteador tem isolamento de clientes Wi-Fi ou
se outro firewall bloqueia a conexão.

------------------------------------------------------------------------

## 11. Firewall e exposição à internet

Na verificação inicial, o comando:

``` bash
sudo ufw status
```

retornou:

``` text
Status: inactive
```

Isso significa que o UFW estava inativo naquele momento. Outros
controles de rede ainda podem existir. Em vez de deixar o servidor sem
firewall, planeje habilitá-lo com regras apropriadas para sua forma de
administração.

**Atenção antes de ativar o UFW:** se você administra o Ubuntu por SSH,
permita primeiro o acesso SSH da sua rede para não se bloquear para fora
do servidor. Uma regra para SSH pode ser, por exemplo:

``` bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

Para permitir o PostgreSQL apenas a dispositivos da sub-rede local:

``` bash
sudo ufw allow from 192.168.1.0/24 to 192.168.1.201 port 5432 proto tcp
```

Depois de revisar as regras e garantir que não perderá acesso
administrativo:

``` bash
sudo ufw enable
sudo ufw status verbose
```

Adapte a regra SSH se o acesso administrativo vier de outra rede ou
interface. Não ative o firewall sem considerar outras portas e serviços
de que você depende.

**Não faça redirecionamento da porta 5432 no roteador para a internet.**
Para acesso remoto futuro, prefira uma VPN, como WireGuard ou Tailscale,
e mantenha a autenticação e as permissões do banco bem restritas.

------------------------------------------------------------------------

## 12. Boas práticas de organização

-   **Um banco por aplicação independente:** ajuda a separar dados e
    permissões.
-   **Uma role por aplicação:** evite compartilhar o mesmo usuário entre
    projetos.
-   **Privilégio mínimo:** a aplicação não deve usar o superusuário
    `postgres`.
-   **Senhas fortes:** não coloque senhas em repositórios Git nem as
    compartilhe em mensagens ou logs.
-   **IP estável:** configure uma reserva DHCP no roteador para
    `192.168.1.201`, ou atualize a configuração se o IP mudar.
-   **Backups:** configure cópias de segurança e teste a restauração
    periodicamente.
-   **Monitoramento:** acompanhe espaço em disco, memória, logs e
    disponibilidade.
-   **Atualizações:** mantenha o Ubuntu e os pacotes PostgreSQL
    atualizados.

Bancos diferentes na mesma instância **não isolam completamente os
recursos**: compartilham CPU, memória, disco, configuração e o serviço
PostgreSQL. Separação de bancos também não substitui backups.

------------------------------------------------------------------------

## 13. Comandos úteis de consulta

  Objetivo                  Comando
  ------------------------- ------------------------------------------------
  Ver clusters              `pg_lsclusters`
  Ver serviço               `sudo systemctl status postgresql`
  Reiniciar PostgreSQL      `sudo systemctl restart postgresql`
  Recarregar configuração   `sudo systemctl reload postgresql`
  Ver escuta da porta       `sudo ss -ltnp \| grep 5432`
  Listar bancos             `sudo -u postgres psql -l`
  Listar roles              `sudo -u postgres psql -c "\du"`
  Ver firewall UFW          `sudo ufw status verbose`
  Ver versão                `sudo -u postgres psql -c "SELECT version();"`

### Consultas úteis dentro do `psql`

``` sql
SELECT current_database(), current_user;
```

Lista os bancos:

``` sql
\l
```

Lista as roles:

``` sql
\du
```

Sai do console:

``` sql
\q
```

------------------------------------------------------------------------

## 14. Checklist de validação

-   [x] PostgreSQL 18 instalado no Ubuntu Server.
-   [x] Cluster `18/main` em estado `online`.
-   [x] IP local identificado como `192.168.1.201`.
-   [x] PostgreSQL escutando em `192.168.1.201:5432` e `127.0.0.1:5432`.
-   [x] Role `app_user` criada.
-   [x] Banco `appdb` criado com proprietário `app_user`.
-   [x] Regra de acesso à rede configurada para `appdb` e `app_user`.
-   [x] Conexão de teste ao `appdb` realizada com sucesso pelo próprio
    servidor, usando o IP de rede.
-   [ ] Confirmar conexão a partir de outro computador da rede
    doméstica.
-   [ ] Reservar o IP do servidor no roteador.
-   [ ] Configurar e validar as regras do firewall.
-   [ ] Preparar backups automáticos e testar restauração.

**Observação:** o acesso pelo próprio servidor via `192.168.1.201` foi
validado. A conexão a partir de outro computador ainda precisa ser
testada para confirmar o caminho completo pela rede doméstica.
