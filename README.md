# Guia Passo a Passo para Configurar uma VPS Ubuntu 24.04 para Aplicações usando GitHub e Docker

Este é meu guia ensinando a configurar uma VPS para subir aplicações usando GitHub e Docker.

## Passo 1: Acesse sua VPS Usando SSH

O primeiro passo é ter uma VPS, mas suponho que você já tenha, se está lendo este guia.

use o comando a baixo, trocando `USER` pelo seu usuário na VPS e `IP` pelo endereço IPV4 que você recebeu quando adquiriu a VPS:

```bash
ssh USER@IP
```

Siga o passo a passo orientado pelo SSH para validar sua senha de usuário root da VPS e prontinho, estará logado no sistema

## Passo 2: Criando um novo usuário

Ter um usuário diretamente associado a aplicação garante algumas vantagens como:

- Segurança: Isola cada aplicação, limitando o impacto de uma possível vulnerabilidade ou ataque a uma única aplicação.
- Permissões: Facilita a gestão de permissões, garantindo que cada aplicação só tenha acesso aos recursos necessários.
- Organização: Mantém os arquivos e processos de cada aplicação separados, tornando a administração do sistema mais organizada.
- Auditoria: Simplifica o monitoramento e auditoria, permitindo rastrear atividades específicas de cada aplicação.
Facilidade de Backup e - Restauração: Facilita o backup e restauração de dados específicos de cada aplicação.

No terminal use o comando a baixo, trocando `NEWUSER` pelo usuário que irá criar:

```bash
adduser NEWUSER
```

Opcional: Se você quiser dar permissão de `super usuário` como `sudo` e afins, use o comando a baixo, lembrando de trocar `NEWUSER` pelo usuário que você criou anteriormente

```bash
usermod -aG sudo NEWUSER
```

Agora configure uma cópia da chave SSH do usuário `root` para o novo usuário, assim o mesmo poderá acessar via SSH. (obs: Lembre de trocar `NEWUSER` pelo usuário que criou, caso contrário vai dar erro!)

```bash
rsync --archive --chown=NEWUSER:NEWUSER ~/.ssh /home/NEWUSER
```

Prontinho, agora basta relogar e acessar usando ssh com o usuário que você criou

## Passo 3:  Configurando Git e GitHub

Adicione seu usuário e email git usando:

```bash
git config --global user.name "SEU_NOME"
```

```bash
git config --global user.email "SEU_EMAIL"
```

Agora, configure uma chave SSH parar conseguir acessar seus repositórios no GitHub usando o git. Siga este tutorial para fazer isso: [github-ssh-key-tutorial](https://github.com/mauriciobenjamin700/github-ssh-key-tutorial)

Ao finalizar o tutorial, você terá seu github pronto para clonar seu repositório do Github para o Servido VPS.

## Passo 4: Colocando o Projeto

Basta clonar seu projeto para o servidor VSP usando o comando a baixo, lembrando de trocar `USER` e `REPO` pelo seus respectivos usuário e repositório no GitHub:

```bash
git clone git@github.com:USER/REPO.git

```

## Configurando o Docker no Servidor

Vou pressupor que você já tem os DockerFiles configurados no seu projeto e o docker compose, então basta instalar o docker no seu servidor VPS. Para isso recomendo o meu tutorial sobre docker que está a baixo!

[Minha Jornada Aprendendo a Usar Docker](https://github.com/mauriciobenjamin700/docker-learning)

Pronto, basta rodar o seu docker compose e ser feliz agora.
