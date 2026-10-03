# checkout — baixa o seu código para dentro da automação do GitHub

**Em uma frase:** é uma peça de automação (uma *GitHub Action*) que **copia o
código do seu repositório para a máquina temporária que roda os seus testes,
builds e deploys**. Sem ela, essa máquina começa vazia e não tem o que
executar. É uma das actions mais conhecidas e usadas do GitHub: a grande
maioria dos workflows de CI/CD começa com ela.

> **Sobre este repositório:** é um **fork** (cópia pessoal) do projeto oficial
> [actions/checkout](https://github.com/actions/checkout), mantido pelo GitHub.
> Foi criado em 08/09/2024 e **não possui nenhuma alteração própria**
> (0 commits à frente do original). O código-fonte, a documentação e a licença
> pertencem aos autores originais (GitHub, Inc.).

[![Build and Test](https://github.com/actions/checkout/actions/workflows/test.yml/badge.svg)](https://github.com/actions/checkout/actions/workflows/test.yml)

## Índice

**Parte 1 — Sobre este fork**

1. [O que é](#1-o-que-é)
2. [Por que isso importa para quem desenvolve](#2-por-que-isso-importa-para-quem-desenvolve)
3. [Por que existe este fork](#3-por-que-existe-este-fork)
4. [Glossário rápido](#4-glossário-rápido)
5. [Uso básico](#5-uso-básico)
6. [Como funciona (visão geral do código)](#6-como-funciona-visão-geral-do-código)
7. [Desenvolvimento local](#7-desenvolvimento-local)
8. [Manter o fork atualizado](#8-manter-o-fork-atualizado)

**Parte 2 — Documentação original (traduzida para português)**

- [Checkout V4](#checkout-v4)
- [Novidades](#novidades)
- [Uso (todas as opções)](#uso-todas-as-opções)
- [Cenários](#cenários)
- [Licença](#licença)

---

# Parte 1 — Sobre este fork

## 1. O que é

Uma **GitHub Action** que faz o *checkout* (clonagem) do seu repositório dentro
do *runner* do GitHub Actions (na pasta `$GITHUB_WORKSPACE`), para que os passos
seguintes do workflow consigam acessar o código: compilar, testar, publicar etc.

**Analogia:** imagine uma linha de montagem. Cada estação (compilar, testar,
publicar) precisa da matéria-prima na bancada. O `checkout` é a primeira
estação: ele não fabrica nada, só **coloca o seu código na bancada** para as
demais trabalharem. Sem ele, o servidor temporário que roda o workflow começa
completamente vazio.

Em resumo: é o passo que, em quase todo workflow, vem antes de qualquer outro.

## 2. Por que isso importa para quem desenvolve

Hoje, equipes de software não testam nem publicam o código "na mão". Elas
configuram **CI/CD** (integração e entrega contínuas): a cada `git push` ou
pull request, um robô roda os testes, verifica a qualidade e, se tudo passar,
publica a aplicação. O `checkout` é o que permite esse robô **enxergar o seu
código**.

**Sem o checkout**, o workflow é só um servidor vazio:

```yaml
steps:
  - run: npm test        # ERRO: não há código aqui, nada para testar
```

**Com o checkout**, o código chega antes dos demais passos:

```yaml
steps:
  - uses: actions/checkout@v4   # 1º: traz o código
  - run: npm test               # 2º: agora há o que testar
```

O que ele resolve na prática:

- **Padronização:** uma única linha, usada por um enorme número de projetos, em vez de
  cada equipe escrever seus próprios comandos `git clone`.
- **Autenticação resolvida:** cuida do token de acesso e o remove no final, o
  que reduz o risco de vazar credenciais.
- **Eficiência:** por padrão baixa só o commit mais recente, deixando a
  automação mais rápida e barata. Quando precisa, baixa o histórico completo.
- **Flexibilidade:** baixa só algumas pastas, outro branch, submódulos, arquivos
  grandes (LFS), repositórios privados ou vários repositórios no mesmo job.
- **Base para todo o resto:** testes, build, análise de segurança, deploy,
  geração de documentação e releases dependem dele.

**Quem usa:** qualquer pessoa ou equipe que automatize tarefas no GitHub, de
projetos pessoais a grandes empresas. Se você abrir os workflows de quase
qualquer projeto open source no GitHub, vai encontrar `actions/checkout` lá.

## 3. Por que existe este fork

Fork feito em setembro de 2024, na versão **4.1.7**, provavelmente para estudo ou
como cópia de segurança. Hoje está **defasado em ~52 commits** em relação ao
original.

> **Um fork é como uma fotocópia de um livro:** a cópia fica igual ao livro no
> dia em que foi tirada, mas, se o autor lançar uma edição nova, a sua
> fotocópia não muda sozinha. É por isso que o fork ficou para trás.

Se o objetivo é apenas **usar** a action em seus workflows, **não é necessário
este fork**: use direto `actions/checkout`.

## 4. Glossário rápido

| Termo | O que significa |
|---|---|
| **GitHub Actions** | Sistema de automação do GitHub: roda tarefas (testes, deploy etc.) quando algo acontece no repositório. |
| **Workflow** | Arquivo `.yml` em `.github/workflows/` que descreve essas tarefas. |
| **Action** | Peça reutilizável que você encaixa num workflow com `uses:`. |
| **Runner** | Máquina temporária (virtual) que executa o workflow. Nasce vazia e é descartada ao final. |
| **Checkout** | Baixar o código do repositório para dentro do runner. |
| **Fork** | Cópia de um repositório de outra pessoa para a sua conta. |
| **PAT** | *Personal Access Token*: uma "senha" com permissões limitadas, usada para acessar repositórios privados. |
| **Upstream** | O repositório original, de onde o fork foi copiado. |

## 5. Uso básico

```yaml
steps:
  - uses: actions/checkout@v4
```

Para usar este fork em vez do original, troque a referência:

```yaml
steps:
  - uses: PedalanteBranco/checkout@main
```

Exemplos comuns:

```yaml
# Histórico completo (todos os branches e tags)
- uses: actions/checkout@v4
  with:
    fetch-depth: 0

# Outro branch, tag ou SHA
- uses: actions/checkout@v4
  with:
    ref: meu-branch

# Outro repositório privado (exige um PAT)
- uses: actions/checkout@v4
  with:
    repository: minha-org/meu-repo-privado
    token: ${{ secrets.GH_PAT }}
    path: meu-repo
```

A lista completa de *inputs* (as "opções" da action) está na
[Parte 2](#uso-todas-as-opções) deste documento e no arquivo
[`action.yml`](action.yml).

## 6. Como funciona (visão geral do código)

- **Linguagem:** TypeScript, executado em **Node 20** (`runs.using: node20`).
- **Entrada:** `src/main.ts` lê os inputs, chama o provedor de código-fonte e
  define a saída `ref`. Um passo `post` faz a limpeza (remove token/chave SSH).
- **Núcleo:** `src/git-source-provider.ts` orquestra o checkout; os demais
  arquivos em `src/` são auxiliares (autenticação, comandos git, API REST como
  plano B quando o git é anterior à 2.18, validação de inputs, *retry* etc.).
- **Build:** o código é empacotado com `ncc` em `dist/index.js`, que é o arquivo
  que o GitHub realmente executa. Por isso a pasta `dist/` é versionada.
- **Testes:** Jest (`__test__/`).
- **CI:** workflows em `.github/workflows/` (testes, CodeQL, verificação do
  `dist/`, licenças).

## 7. Desenvolvimento local

```bash
npm install
npm run build         # compila (tsc) e empacota (ncc) em dist/
npm run format-check  # Prettier
npm run lint          # ESLint
npm test              # Jest
```

## 8. Manter o fork atualizado

```bash
git remote add upstream https://github.com/actions/checkout.git
git fetch upstream
git merge upstream/main
```

Ou use o botão **Sync fork** na página do repositório no GitHub.

---

# Parte 2 — Documentação original (traduzida para português)

> **Aviso:** tradução livre do README oficial de
> [actions/checkout](https://github.com/actions/checkout#readme), feita para
> facilitar o estudo. Em caso de dúvida ou divergência, vale o texto oficial em
> inglês, que é mantido pelos autores originais e pode estar mais atualizado do
> que esta cópia.

## Checkout V4

Esta action faz o checkout do seu repositório dentro de `$GITHUB_WORKSPACE`,
para que o seu workflow possa acessá-lo.

Por padrão, **apenas um commit é baixado**: o do ref/SHA que disparou o
workflow. Use `fetch-depth: 0` para baixar todo o histórico de todos os
branches e tags. Consulte
[aqui](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows)
para saber a qual commit o `$GITHUB_SHA` aponta em cada tipo de evento.

O token de autenticação fica **gravado na configuração local do git**. Isso
permite que seus scripts executem comandos git autenticados. O token é removido
na limpeza que acontece ao final do job. Use `persist-credentials: false` se
não quiser esse comportamento.

Quando o Git 2.18 ou superior não está no `PATH`, a action recorre à **API REST**
para baixar os arquivos.

> **Em linguagem simples:** por padrão ele baixa só a "foto" mais recente do seu
> código (rápido e leve), em vez do álbum inteiro com todo o histórico. Se você
> precisar do álbum completo, é só pedir com `fetch-depth: 0`.

## Novidades

Consulte a [página de releases](https://github.com/actions/checkout/releases/latest)
para ver as notas da versão mais recente.

## Uso (todas as opções)

Cada linha abaixo é uma opção que você pode passar dentro de `with:`. Todas são
**opcionais**. Quando você não informa nada, vale o valor padrão (`Default`).
Os comentários foram traduzidos; os nomes das opções permanecem em inglês, pois
é assim que a action os reconhece.

```yaml
- uses: actions/checkout@v4
  with:
    # Nome do repositório com o dono. Por exemplo: actions/checkout
    # Padrão: ${{ github.repository }}
    repository: ''

    # O branch, tag ou SHA que será baixado. Ao fazer checkout do repositório que
    # disparou o workflow, o padrão é o ref ou SHA daquele evento.
    # Caso contrário, usa o branch padrão.
    ref: ''

    # Token de acesso pessoal (PAT) usado para baixar o repositório. O PAT é
    # configurado no git local, o que permite que seus scripts executem comandos
    # git autenticados. O passo final do job remove o PAT.
    #
    # Recomendamos usar uma conta de serviço com o mínimo de permissões necessárias.
    # Ao gerar um novo PAT, também selecione o mínimo de escopos necessários.
    #
    # [Saiba mais sobre criar e usar segredos criptografados](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)
    #
    # Padrão: ${{ github.token }}
    token: ''

    # Chave SSH usada para baixar o repositório. A chave SSH é configurada no git
    # local, o que permite que seus scripts executem comandos git autenticados.
    # O passo final do job remove a chave SSH.
    #
    # Recomendamos usar uma conta de serviço com o mínimo de permissões necessárias.
    #
    # [Saiba mais sobre criar e usar segredos criptografados](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)
    ssh-key: ''

    # Hosts conhecidos, além do banco de chaves de hosts do usuário e do global.
    # As chaves públicas SSH de um host podem ser obtidas com o utilitário
    # `ssh-keyscan`. Por exemplo: `ssh-keyscan github.com`. A chave pública do
    # github.com é sempre adicionada implicitamente.
    ssh-known-hosts: ''

    # Se deve fazer a verificação rigorosa da chave do host. Quando true, adiciona
    # as opções `StrictHostKeyChecking=yes` e `CheckHostIP=no` à linha de comando
    # SSH. Use a opção `ssh-known-hosts` para configurar hosts adicionais.
    # Padrão: true
    ssh-strict: ''

    # O usuário usado ao conectar ao host SSH remoto. Por padrão, é usado 'git'.
    # Padrão: git
    ssh-user: ''

    # Se deve gravar o token ou a chave SSH na configuração local do git
    # Padrão: true
    persist-credentials: ''

    # Caminho relativo, dentro de $GITHUB_WORKSPACE, onde o repositório será colocado
    path: ''

    # Se deve executar `git clean -ffdx && git reset --hard HEAD` antes de baixar
    # Padrão: true
    clean: ''

    # Clone parcial usando um filtro. Se definido, sobrescreve o sparse-checkout.
    # Padrão: null
    filter: ''

    # Faz um sparse checkout (baixa só parte dos arquivos) nos padrões informados.
    # Cada padrão deve ser separado por uma nova linha.
    # Padrão: null
    sparse-checkout: ''

    # Define se o modo "cone" será usado no sparse checkout.
    # Padrão: true
    sparse-checkout-cone-mode: ''

    # Número de commits a baixar. 0 significa todo o histórico de todos os
    # branches e tags.
    # Padrão: 1
    fetch-depth: ''

    # Se deve baixar as tags, mesmo quando fetch-depth > 0.
    # Padrão: false
    fetch-tags: ''

    # Se deve mostrar o progresso durante o download.
    # Padrão: true
    show-progress: ''

    # Se deve baixar os arquivos do Git-LFS
    # Padrão: false
    lfs: ''

    # Se deve fazer checkout dos submódulos: `true` para baixar os submódulos ou
    # `recursive` para baixá-los recursivamente.
    #
    # Quando a opção `ssh-key` não é informada, URLs SSH que começam com
    # `git@github.com:` são convertidas para HTTPS.
    #
    # Padrão: false
    submodules: ''

    # Adiciona o caminho do repositório como safe.directory na configuração global
    # do Git, executando `git config --global --add safe.directory <caminho>`
    # Padrão: true
    set-safe-directory: ''

    # A URL base da instância do GitHub de onde você quer clonar. Se não for
    # informada, usa o padrão do ambiente, ou seja, a mesma instância em que o
    # workflow está rodando. Exemplos: https://github.com ou
    # https://meu-servidor-ghes.exemplo.com
    github-server-url: ''
```

## Cenários

- [Baixar apenas os arquivos da raiz](#baixar-apenas-os-arquivos-da-raiz)
- [Baixar a raiz e as pastas `.github` e `src`](#baixar-a-raiz-e-as-pastas-github-e-src)
- [Baixar apenas um único arquivo](#baixar-apenas-um-único-arquivo)
- [Baixar todo o histórico de todas as tags e branches](#baixar-todo-o-histórico-de-todas-as-tags-e-branches)
- [Fazer checkout de outro branch](#fazer-checkout-de-outro-branch)
- [Fazer checkout de HEAD^](#fazer-checkout-de-head)
- [Vários repositórios (lado a lado)](#vários-repositórios-lado-a-lado)
- [Vários repositórios (aninhados)](#vários-repositórios-aninhados)
- [Vários repositórios (privados)](#vários-repositórios-privados)
- [Commit HEAD do pull request em vez do commit de merge](#commit-head-do-pull-request-em-vez-do-commit-de-merge)
- [Pull request no evento `closed`](#pull-request-no-evento-closed)
- [Fazer push de um commit usando o token embutido](#fazer-push-de-um-commit-usando-o-token-embutido)

### Baixar apenas os arquivos da raiz

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: .
```

### Baixar a raiz e as pastas `.github` e `src`

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      .github
      src
```

### Baixar apenas um único arquivo

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      README.md
    sparse-checkout-cone-mode: false
```

### Baixar todo o histórico de todas as tags e branches

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

### Fazer checkout de outro branch

```yaml
- uses: actions/checkout@v4
  with:
    ref: my-branch
```

### Fazer checkout de HEAD^

`HEAD^` é o commit anterior ao atual. Por isso é preciso baixar pelo menos 2
commits (`fetch-depth: 2`).

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 2
- run: git checkout HEAD^
```

### Vários repositórios (lado a lado)

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    path: main

- name: Checkout do repositório de ferramentas
  uses: actions/checkout@v4
  with:
    repository: my-org/my-tools
    path: my-tools
```

> Se o seu segundo repositório for privado, você precisará adicionar a opção
> descrita em [Vários repositórios (privados)](#vários-repositórios-privados).

### Vários repositórios (aninhados)

```yaml
- name: Checkout
  uses: actions/checkout@v4

- name: Checkout do repositório de ferramentas
  uses: actions/checkout@v4
  with:
    repository: my-org/my-tools
    path: my-tools
```

> Se o seu segundo repositório for privado, você precisará adicionar a opção
> descrita em [Vários repositórios (privados)](#vários-repositórios-privados).

### Vários repositórios (privados)

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    path: main

- name: Checkout das ferramentas privadas
  uses: actions/checkout@v4
  with:
    repository: my-org/my-private-tools
    token: ${{ secrets.GH_PAT }} # `GH_PAT` é um segredo que contém o seu PAT
    path: my-tools
```

> O `${{ github.token }}` tem acesso apenas ao repositório atual. Por isso, se
> você quiser baixar **outro** repositório privado, precisará fornecer o seu
> próprio [PAT](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line).
>
> **Analogia:** o token padrão é um crachá que abre só a sala do seu próprio
> projeto. Para entrar na sala de outro projeto privado, você precisa de um
> crachá extra (o PAT).

### Commit HEAD do pull request em vez do commit de merge

Por padrão, em um pull request o GitHub baixa um commit de *merge* temporário
(a mistura do seu branch com o branch de destino). Para baixar exatamente o
último commit do seu branch:

```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

### Pull request no evento `closed`

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, closed]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

### Fazer push de um commit usando o token embutido

```yaml
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          date > generated.txt
          # Atenção: as informações de conta abaixo não funcionam no GHES
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "generated"
          git push
```

**Observação:** o e-mail do usuário segue o formato
`{user.id}+{user.login}@users.noreply.github.com`. Consulte a API de usuários:
https://api.github.com/users/github-actions%5Bbot%5D

---

## Licença

Os scripts e a documentação deste projeto são distribuídos sob a
[Licença MIT](LICENSE). O copyright pertence aos autores originais
(GitHub, Inc.); este repositório é apenas um fork.
