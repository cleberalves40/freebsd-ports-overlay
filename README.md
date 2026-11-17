# freebsd-ports-overlay

Ports do FreeBSD mantidos por Cleber Alves do Nascimento, fora da árvore oficial. A
estrutura é a de uma árvore de ports (`<categoria>/<port>`), para uso como overlay do
poudriere ou com `make` direto. Os distfiles que não existem em outro lugar (como o cache
do npm do Thinker) ficam nas [releases](https://github.com/cleberalves40/freebsd-ports-overlay/releases)
deste repositório.

| Port | Versão | Aplicativo (upstream) |
|---|---|---|
| [`www/thinker`](www/thinker) | 0.41.2 | [Thinker](https://github.com/patrickbrandao/thinker): gateway de LLM para agentes de IA (MIT) |

```sh
git clone https://github.com/cleberalves40/freebsd-ports-overlay.git
poudriere ports -c -p overlay -m null -M "$PWD/freebsd-ports-overlay"
poudriere bulk -j <jail> -p default -O overlay www/thinker
# ou, sem poudriere:
make -C freebsd-ports-overlay/www/thinker install clean
```

Os ports deste repositório usam a licença BSD-2-Clause (como a árvore oficial). Cada
aplicativo empacotado mantém a sua própria licença.

---

# www/thinker

Pacote nativo do [Thinker](https://github.com/patrickbrandao/thinker) para o FreeBSD,
mantido neste repositório em [`www/thinker`](www/thinker).

| Papel | Quem | Onde |
|---|---|---|
| Aplicativo (autor e fonte) | Patrick Brandão e contribuidores do Thinker, licença MIT | [patrickbrandao/thinker](https://github.com/patrickbrandao/thinker) |
| Port (empacotamento para FreeBSD) | Cleber Alves do Nascimento, `MAINTAINER` do port | [cleberalves40/freebsd-ports-overlay](https://github.com/cleberalves40/freebsd-ports-overlay), `www/thinker` |

O port não altera o código do Thinker: compila a tag publicada pelo upstream e empacota o
resultado. Defeitos do aplicativo vão para o upstream; defeitos de empacotamento (rc.d,
lançador, plist, dependências) ficam aqui.

### O que o pacote instala

| Caminho | Conteúdo |
|---|---|
| `/usr/local/libexec/thinker` | Aplicação compilada: backend, painel, CLI e `node_modules` de produção (JavaScript puro) |
| `/usr/local/libexec/thinker-run` | Lançador: monta o ambiente a partir do `rc.conf`, lê os segredos e faz `exec node` |
| `/usr/local/bin/thinker` | CLI oficial do Thinker |
| `/usr/local/etc/rc.d/thinker_db` | Cluster PostgreSQL dedicado (socket Unix, autenticação peer); `initdb` como comando extra |
| `/usr/local/etc/rc.d/thinker` | Servidor sob `daemon(8)`, reiniciado se cair |
| `/usr/local/etc/thinker/thinker.env` | Variáveis extras do Thinker (e-mail, OAuth, timeouts) |
| `/usr/local/etc/newsyslog.conf.d/thinker.conf` | Rotação de `/var/log/thinker/*.log` |

Usuário e grupo `thinker` (UID/GID 1006, de `files/UIDs` e `files/GIDs` do port). Dados em
`/var/db/thinker`. Dependências: `node24` e `postgresql18-server`. O pacote não depende da
arquitetura (`NO_ARCH`); o build é limitado a amd64 e aarch64, as plataformas com binários
das ferramentas de build (rolldown, lightningcss, tailwind oxide).

### Instalação

```sh
pkg add thinker-0.41.2.pkg            # ou pkg install thinker, quando houver repositório
sysrc thinker_db_enable=YES thinker_enable=YES
service thinker_db initdb             # diretórios, segredos, cluster, papel e banco
service thinker_db start
service thinker start
```

Configuração no `rc.conf`:

| Variável | Padrão |
|---|---|
| `thinker_port` | `2099` |
| `thinker_bind` | `127.0.0.1` |
| `thinker_url` | `http://localhost:${thinker_port}` (vira `BETTER_AUTH_URL`; com `http:` o HSTS é desligado) |
| `thinker_trusted_proxies` | `127.0.0.1,::1` |
| `thinker_catalog_sync` | `true` |
| `thinker_db_port`, `thinker_db_socket`, `thinker_db_data` | `5433`, `/var/run/thinker-db`, `/var/db/thinker/pgdata` |

### Manutenção do port

O build não acessa a rede. As dependências npm vêm de um segundo distfile, o cache do npm
(`thinker-npm-cache-<versão>.tar.xz`), publicado nas releases deste repositório com a tag
`thinker-npm-cache`. O cache cobre amd64 e aarch64.

Nova versão do upstream:

```sh
cd www/thinker
# 1. DISTVERSION= <nova versão> no Makefile (PORTREVISION volta a 0)
make makesum DISTFILES=patrickbrandao-thinker-v<nova>_GH0.tar.gz   # hash da tag
make make-npm-cache            # npm ci para freebsd x64 e arm64; grava em DISTDIR
make makesum                   # distinfo com os dois distfiles
# 2. publicar o cache na release thinker-npm-cache do repositório:
gh release upload thinker-npm-cache "$(make -V DISTDIR)/thinker-npm-cache-<nova>.tar.xz" \
    -R cleberalves40/freebsd-ports-overlay
# 3. compilar e conferir
make package stage-qa check-plist
poudriere testport -j 15-amd64 -p default -O overlay www/thinker
```

No `do-build`, os workspaces são compilados na ordem do `turbo.json` (o `turbo` não tem
binário FreeBSD): `thinker-shared`, `thinker-backend`, `thinker-frontend`, `thinker-cli`.
A árvore de produção repete o estágio `prod-deps` do Dockerfile oficial
(`npm ci --omit=dev --omit=optional --ignore-scripts`), com a mesma limpeza, mas mantendo
os arquivos de licença dos módulos. O build falha se aparecer um `.node` ou `.so` no
runtime, porque aí o `NO_ARCH` deixaria de valer.

Pontos a conferir a cada versão nova:

- `docker/Dockerfile` do upstream: mudanças nos estágios `build` e `prod-deps` (pastas
  copiadas, módulos removidos) precisam ser refletidas no `do-build`/`do-install`.
- Workspaces novos em `package.json` e a ordem de dependência entre eles.
- Versão do Node exigida (`engines`) e do PostgreSQL.

### Validação (2026-11-17, FreeBSD 15.1, Thinker 0.41.2)

- `make package`, `stage-qa` e `check-plist` sem erros; pacote de 13 MB (126 MB instalado),
  com os mesmos módulos de primeiro nível da imagem oficial e nenhum binário nativo.
- Jail limpo: `pkg add`, `initdb`, `/api/v1/health` saudável, painel, CLI, 53 tabelas
  criadas pelas migrations; `kill -9` no node com retorno em 8 s; `pkg delete` e
  reinstalação preservando `/var/db/thinker`.
- Pendentes: `poudriere testport` (a árvore `default` do poudriere estava fora de sincronia
  com o repositório de pacotes) e teste em aarch64.
