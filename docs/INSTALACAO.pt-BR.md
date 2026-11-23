# Instalar e configurar o Termix

Guia em português deste fork. O Termix é uma plataforma gratuita, de código aberto e
auto-hospedada para gerenciar servidores (terminal SSH, área de trabalho remota, gerenciador de
arquivos, Docker, métricas e mais, tudo em plugins). Este repositório é uma cópia do projeto
original [Termix-SSH/Termix](https://github.com/Termix-SSH/Termix) (licença Apache 2.0, autoria de
Luke Gustafson e colaboradores). Os créditos são deles; a documentação oficial em inglês está em
<https://docs.termix.site/install>.

## Índice

1. [Escolha como instalar](#1-escolha-como-instalar)
2. [Instalar com Docker (recomendado)](#2-instalar-com-docker-recomendado)
3. [Primeiro acesso](#3-primeiro-acesso)
4. [Configuração por variáveis de ambiente](#4-configuração-por-variáveis-de-ambiente)
5. [HTTPS e proxy reverso](#5-https-e-proxy-reverso)
6. [Banco de dados](#6-banco-de-dados)
7. [Segredos e criptografia](#7-segredos-e-criptografia)
8. [Backup, atualização e remoção](#8-backup-atualização-e-remoção)
9. [Aplicativo de desktop e celular](#9-aplicativo-de-desktop-e-celular)
10. [Rodar a partir do código-fonte](#10-rodar-a-partir-do-código-fonte)
11. [Problemas comuns](#11-problemas-comuns)

## 1. Escolha como instalar

| Eu quero... | Use |
|---|---|
| Um servidor Termix acessível pelo navegador (a forma mais comum) | [Docker](#2-instalar-com-docker-recomendado) |
| Um aplicativo no meu computador (Windows, macOS, Linux) | [Desktop](#9-aplicativo-de-desktop-e-celular) |
| Usar no celular | [Celular](#9-aplicativo-de-desktop-e-celular) |
| Desenvolver ou alterar o código | [Código-fonte](#10-rodar-a-partir-do-código-fonte) |

## 2. Instalar com Docker (recomendado)

Requisitos: Docker com o plugin Compose (`docker compose version` deve responder). Funciona em
Linux, Windows (Docker Desktop) e macOS.

1. Crie uma pasta e, dentro dela, o arquivo `docker-compose.yml` (versão mínima):

   ```yaml
   services:
     termix:
       image: ghcr.io/termix-ssh/termix:latest
       container_name: termix
       restart: unless-stopped
       ports:
         - "8080:8080"
       volumes:
         - termix-data:/app/data
       environment:
         PORT: "8080"

   volumes:
     termix-data:
       driver: local
   ```

   Existe também uma versão endurecida (capacidades reduzidas, `no-new-privileges`, `tmpfs`) em
   [`docker/docker-compose.yml`](../docker/docker-compose.yml); prefira-a em servidores expostos.

2. Suba o serviço:

   ```sh
   docker compose up -d
   docker compose logs -f termix     # acompanhe a inicialização (Ctrl+C sai)
   ```

3. Abra `http://IP-DO-SERVIDOR:8080` no navegador.

Tudo o que importa (banco, chaves, plugins, uploads) fica no volume `termix-data`, montado em
`/app/data`. **Não apague esse volume**: é ele que guarda seus dados.

Variantes de imagem: `latest` traz os plugins padrão; as tags `-full` (por exemplo `latest-full`)
trazem todos os plugins oficiais. Os outros plugins podem ser instalados depois pela aba
*Plugins*.

### Pasta local em vez de volume nomeado

Se preferir ver os arquivos no disco, troque o volume por uma pasta e dê a ela o dono certo (o
contêiner roda com `PUID`/`PGID`, padrão 1000):

```yaml
    volumes:
      - ./data:/app/data
    environment:
      PORT: "8080"
      PUID: "1000"
      PGID: "1000"
```

```sh
mkdir -p data && sudo chown -R 1000:1000 data
```

## 3. Primeiro acesso

1. Abra a página inicial e crie a primeira conta. (Confirme na tela: em instalações novas, a conta
   inicial é a de administração.)
2. Cadastre um host: nome, endereço, porta SSH e a forma de autenticação (senha ou chave). Você pode
   guardar senhas e chaves em *Credenciais* e reutilizá-las em vários hosts.
3. Abra o host para obter o terminal SSH. Recursos como gerenciador de arquivos, Docker e métricas
   aparecem conforme os plugins ativos.
4. Em *Plugins* (administrador) ligue, desligue, instale ou atualize plugins.

Dica: `ALLOW_REGISTRATION=false` fecha o cadastro de novas contas depois que você criou a sua
(veja a próxima seção).

## 4. Configuração por variáveis de ambiente

Defina em `environment:` no compose. A lista completa, com descrição de cada uma, está em
[`env.catalog.json`](../env.catalog.json). As mais usadas:

| Variável | Padrão | Para que serve |
|---|---|---|
| `PORT` | `8080` | Porta da interface web dentro do contêiner. |
| `DATA_DIR` | `/app/data` | Onde ficam banco, chaves, plugins e uploads. Monte um volume aqui. |
| `PUID` / `PGID` | `1000` | Usuário e grupo com que o contêiner roda; deve ser o dono do volume. |
| `BASE_PATH` | vazio | Servir sob um subcaminho, como `/termix` (use o mesmo no proxy). |
| `ALLOW_REGISTRATION` | (config. do admin) | `true`/`false`: permite criar novas contas. |
| `ALLOW_PASSWORD_LOGIN` | (config. do admin) | `true`/`false`: login por usuário e senha. |
| `ALLOW_PASSWORD_RESET` | (config. do admin) | `true`/`false`: redefinir senha pela tela de login. |
| `LOG_LEVEL` | | `debug`, `info`, `warn` ou `error`. |
| `ENABLE_SSL` / `SSL_PORT` | `false` / `8443` | HTTPS direto no Termix (veja a seção 5). |
| `DATABASE_DIALECT` | `sqlite` | `sqlite`, `postgres` ou `mysql`. |
| `REDIS_URL` | vazio | Só para várias instâncias do backend compartilhando salas de colaboração. |

Depois de mudar variáveis: `docker compose up -d` recria o contêiner com os novos valores.

## 5. HTTPS e proxy reverso

**Opção A, proxy reverso (recomendado).** Coloque Nginx, Caddy ou Traefik na frente, com
certificado, e aponte para `http://termix:8080`. Se o proxy não enviar `X-Forwarded-Proto`, defina
`EXTERNAL_FORCE_HTTPS=true` para que os retornos de login (SSO) usem `https`. Se o Termix ficar
em um subcaminho, defina `BASE_PATH` e repita o caminho no proxy. Exemplo mínimo de Caddy:

```
termix.exemplo.com {
    reverse_proxy termix:8080
}
```

**Opção B, HTTPS no próprio Termix.** Defina `ENABLE_SSL=true` e publique a porta `8443`. Sem
certificado próprio, ele gera um autoassinado no primeiro início (o navegador vai avisar).
Certificado e chave próprios: `SSL_CERT_PATH` e `SSL_KEY_PATH` (padrão
`/app/data/ssl/termix.crt`; coloque os arquivos na pasta de dados).

**Login por proxy de autenticação** (Authelia, Authentik): `TRUSTED_PROXY_AUTH_ENABLED=true`,
mais `TRUSTED_PROXY_AUTH_TRUSTED_PROXIES` (IPs do proxy, mantenha estreito) e
`TRUSTED_PROXY_AUTH_ROLE_MAP`. Não exponha o contêiner diretamente quando isso estiver ligado.

## 6. Banco de dados

O padrão é SQLite, criptografado em disco (`DB_FILE_ENCRYPTION=true`), e não precisa de nada. Para
PostgreSQL ou MySQL:

```yaml
    environment:
      DATABASE_DIALECT: postgres
      DATABASE_URL: postgres://usuario:senha@db:5432/termix
      # DATABASE_SSL: require
```

Escolha o banco antes de começar a usar: mudar depois exige migrar os dados.

## 7. Segredos e criptografia

Na primeira execução o Termix **gera sozinho** `JWT_SECRET`, `DATABASE_KEY`, `ENCRYPTION_KEY` e
`INTERNAL_AUTH_TOKEN` e os guarda em `DATA_DIR/.env`. Isso significa que **quem tiver o volume tem
as chaves**, e que perder essa pasta torna as senhas e chaves SSH salvas irrecuperáveis.

Para ambientes mais rígidos, forneça os segredos de fora e proíba que sejam gravados em disco:

```yaml
    environment:
      TERMIX_REQUIRE_EXTERNAL_SECRETS: "true"
      JWT_SECRET_FILE: /run/secrets/jwt_secret
      DATABASE_KEY_FILE: /run/secrets/database_key
      ENCRYPTION_KEY_FILE: /run/secrets/encryption_key
      INTERNAL_AUTH_TOKEN_FILE: /run/secrets/internal_auth_token
```

Formatos (do catálogo de variáveis): `JWT_SECRET` com pelo menos 64 caracteres;
`DATABASE_KEY` e `ENCRYPTION_KEY` com 64 caracteres hexadecimais. Para gerar:

```sh
openssl rand -hex 32     # DATABASE_KEY e ENCRYPTION_KEY
openssl rand -hex 48     # JWT_SECRET (96 caracteres)
```

Nunca troque `ENCRYPTION_KEY` sem seguir o processo de rotação (`ENCRYPTION_KEY_PREVIOUS` guarda as
chaves antigas).

## 8. Backup, atualização e remoção

**Backup:** copie o volume de dados com o contêiner parado.

```sh
docker compose stop termix
docker run --rm -v termix_termix-data:/dados -v "$PWD":/backup alpine \
  tar czf /backup/termix-dados-$(date +%F).tar.gz -C /dados .
docker compose start termix
```

(O nome do volume é `<pasta-do-compose>_termix-data`; confira com `docker volume ls`.) Guarde o
backup em local seguro: ele contém as chaves.

**Atualizar:**

```sh
docker compose pull
docker compose up -d
```

Antes de uma atualização que altera o banco, o Termix já faz uma cópia automática da pasta de
dados (`DATABASE_LAYER_PREUPGRADE_BACKUP_KEEP`, padrão: guarda 3). Mesmo assim, faça o seu backup.

**Remover:** `docker compose down` mantém os dados; `docker compose down -v` **apaga o volume**.

## 9. Aplicativo de desktop e celular

Baixe na [página de Releases do projeto original](https://github.com/Termix-SSH/Termix/releases):

| Sistema | Formatos |
|---|---|
| Windows (x64/arm64) | Portátil, instalador EXE e MSI, EXE "full" (todos os plugins), Chocolatey |
| Linux (x64/arm64) | Portátil, AppImage, Deb, versões "full", AUR, Flatpak |
| macOS (12.0+) | DMG, DMG "full", Homebrew, App Store |
| iOS/iPadOS (15.1+) | App Store, IPA |
| Android (7.0+) | Google Play, APK |

O aplicativo de desktop funciona sozinho e também pode se ligar a um servidor Termix para manter
uma cópia offline da sua conta (plugin *Sync*). No celular, informe o endereço do seu servidor.

## 10. Rodar a partir do código-fonte

Requisitos: Node.js 22.12 ou mais novo (o arquivo `.nvmrc` indica a versão usada: 24) e npm 11 ou
mais novo.

```sh
git clone https://github.com/sidneyjfg/termix.git
cd termix
npm install
npm run dev            # backend + interface em http://localhost:5173/
npm run dev:electron   # o mesmo, com o app de desktop
```

Outros comandos: `npm run test`, `npm run type-check`, `npm run lint`, `npm run format`,
`npm run build`. Veja [CONTRIBUTING.md](../CONTRIBUTING.md). Fora do Docker, os dados ficam em
`./db/data`.

Para gerar a imagem Docker local: `docker build -f docker/Dockerfile -t termix .`

## 11. Problemas comuns

| Sintoma | O que verificar |
|---|---|
| Não abre em `:8080` | `docker compose ps` e `docker compose logs termix`; a porta pode estar em uso (mude o lado esquerdo de `8080:8080`); firewall do servidor. |
| `permission denied` na pasta de dados | O dono da pasta deve ser o `PUID`/`PGID` do contêiner (`chown -R 1000:1000 data`). |
| Aviso de certificado no navegador | É o certificado autoassinado do `ENABLE_SSL`; use um proxy com certificado válido. |
| Login por SSO volta para `http` | Defina `EXTERNAL_FORCE_HTTPS=true` atrás de proxy sem `X-Forwarded-Proto`. |
| Iniciou "vazio" depois de mudar de pasta | O Termix recusa iniciar com a pasta de dados vazia se achar dados antigos em outro lugar; só use `ALLOW_EMPTY_DATA_DIR=true` se quiser mesmo uma instalação nova. |
| Esqueci a senha do administrador | Existe uma ferramenta de recuperação no contêiner (`termix-recover`); veja `docker/termix-recover.sh` e a documentação oficial. |
| Hosts ou senhas sumiram depois de recriar o contêiner | O volume de dados não estava montado em `/app/data`. |

---

*Este guia foi escrito a partir dos arquivos do repositório (`docker/`, `env.catalog.json`,
`README.md`, `CONTRIBUTING.md`). Onde o texto diz "confirme", o comportamento não foi testado aqui.*
