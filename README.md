# 🐳 Docker e Docker Compose: Do Caos à Orquestração em 45 Minutos

Bem-vindo(a) ao repositório oficial da nossa aula prática sobre orquestração de contêineres!

Este material foi desenhado para levar você dos conceitos fundamentais de isolamento de software até a implantação de uma infraestrutura multi-serviços completa utilizando **Docker Compose**. Tudo isso em 45 minutos, rodando 100% na nuvem, sem precisar instalar nada na sua máquina.

> Esta versão do guia foi ampliada com mais contexto teórico, comparações práticas, boas práticas de mercado e uma seção de perguntas frequentes — material de sobra para preencher os 45 minutos com segurança, mesmo que a turma tenha muitas dúvidas pelo caminho. **O laboratório prático (Seção 8) permanece exatamente como testado e validado — nada nele foi alterado.**

---

## 📚 Sumário
1. [Fundamentos: O Fim do "Na minha máquina funciona"](#2-fundamentos-o-fim-do-na-minha-máquina-funciona)
2. [Containers vs. Máquinas Virtuais: Por Baixo do Capô](#3-containers-vs-máquinas-virtuais-por-baixo-do-capô)
3. [O Problema: Gerenciamento Imperativo](#4-o-problema-gerenciamento-imperativo)
4. [A Solução: Docker Compose e Infraestrutura como Código (IaC)](#5-a-solução-docker-compose-e-infraestrutura-como-código-iac)
5. [Anatomia do Compose YAML](#6-anatomia-do-compose-yaml)
6. [Ambiente Prático: Killercoda](#7-ambiente-prático-killercoda)
7. [Laboratório Passo a Passo (Hands-on)](#8-laboratório-passo-a-passo-hands-on)
8. [Boas Práticas e Erros Comuns](#9-boas-práticas-e-erros-comuns)
9. [O Próximo Nível: Escala e Kubernetes](#10-o-próximo-nível-escala-e-kubernetes)
10. [Perguntas Frequentes (FAQ)](#11-perguntas-frequentes-faq)
11. [Cheat Sheet (Comandos Úteis)](#12-cheat-sheet-comandos-úteis)

---

## 1. Fundamentos: O Fim do "Na minha máquina funciona"

No desenvolvimento de software tradicional, configurar laboratórios ou servidores do zero exige instalar dependências complexas. Se você já precisou subir máquinas virtuais completas (com sistemas operacionais inteiros) apenas para rodar um servidor web ou um banco de dados, sabe que o processo consome muito disco, memória e tempo de configuração.

**A Solução (Docker):** O Docker utiliza a tecnologia de **contêineres**. Ele empacota o código da sua aplicação (por exemplo, scripts em Python), as bibliotecas necessárias e as configurações de sistema em uma única unidade padronizada e imutável.

Diferente de uma Máquina Virtual que emula o hardware e carrega um sistema operacional pesado (Guest OS), os contêineres compartilham o *kernel* do sistema hospedeiro, inicializando em milissegundos e consumindo uma fração dos recursos.

### 🚢 Uma analogia que ajuda

Pense no transporte marítimo antes e depois do contêiner de aço padronizado. Antes, cada tipo de carga exigia um método de embarque diferente — sacas eram empilhadas manualmente, máquinas eram amarradas com cordas, e cada porto tinha seu próprio jeito de lidar com cada tipo de carga. O contêiner padronizado resolveu isso: não importa o que está dentro, ele tem o mesmo tamanho e os mesmos encaixes, e qualquer guindaste do mundo sabe como movê-lo.

O Docker faz exatamente isso com software: não importa se sua aplicação é Python, Node.js ou Java — ela vira uma "caixa padronizada" que roda da mesma forma em qualquer máquina que tenha o Docker instalado.

### 🏗️ Arquitetura do Docker: quem fala com quem

O Docker segue um modelo cliente-servidor, mesmo quando tudo roda na mesma máquina:

* **Docker Client (`docker`)**: o comando que você digita no terminal.
* **Docker Daemon (`dockerd`)**: o processo em segundo plano que efetivamente cria, executa e gerencia os contêineres. O client conversa com o daemon através de uma API REST.
* **Docker Registry (ex.: Docker Hub)**: o "repositório" de onde as imagens prontas são baixadas — é de lá que virá o `redis:7-alpine` que usaremos no laboratório.

### 📦 Imagem vs. Container: a confusão mais comum

Esses dois termos costumam ser usados como sinônimos, mas não são a mesma coisa:

* **Imagem**: um template somente-leitura e imutável, feito de camadas empilhadas (cada instrução do `Dockerfile` — `FROM`, `RUN`, `COPY` — gera uma camada nova). É a "planta" do que vai rodar.
* **Container**: uma **instância em execução** de uma imagem, com uma fina camada gravável adicionada por cima. É efêmero: se você apagar o container, essa camada gravável some junto — daí a necessidade de `volumes` para dados que precisam sobreviver (voltamos a isso na Seção 6).

Essa distinção entre "o que é permanente" (imagem, volume) e "o que é descartável" (container) é a base conceitual de tudo o que vamos construir hoje.

---

## 2. Containers vs. Máquinas Virtuais: Por Baixo do Capô

A pergunta que sempre aparece nesse ponto da aula é: "isso não é só uma VM mais rápida?" Não — a diferença é estrutural, não só de performance.

| Característica | Máquina Virtual | Container |
| :--- | :--- | :--- |
| Isolamento | Hypervisor emula hardware completo | Compartilha o kernel do host (namespaces + cgroups) |
| Sistema operacional | Guest OS completo por VM | Nenhum — usa o kernel do hospedeiro |
| Tamanho típico | Gigabytes | Megabytes |
| Tempo de boot | Minutos | Milissegundos a segundos |
| Densidade por host | Poucas dezenas | Centenas |
| Caso de uso ideal | Isolar kernels/SOs diferentes | Empacotar e distribuir aplicações |

Dois mecanismos do kernel Linux tornam isso possível, e vale citá-los em aula:

* **Namespaces**: isolam o que cada processo "enxerga" — cada container tem sua própria visão de rede, processos (PID), sistema de arquivos e hostname, mesmo compartilhando o mesmo kernel.
* **cgroups (control groups)**: limitam quanto de CPU, memória e I/O cada container pode consumir, evitando que um container "faminto" derrube a máquina inteira.

> 🔎 Não é preciso entrar em detalhes de implementação com a turma — o ponto pedagógico é: **containers são processos isolados, não máquinas**. Isso explica tanto a velocidade de inicialização quanto a "Regra de Ouro da Rede" que veremos na Seção 6.

---

## 3. O Problema: Gerenciamento Imperativo

Antes da orquestração automatizada, subíamos a infraestrutura comando por comando. Esse modelo gera vários problemas:

* **Risco Humano:** Esquecer *flags* de rede ou variáveis de ambiente quebra o sistema.
* **Ordem de Execução:** É difícil garantir que a aplicação web só inicie após o banco de dados estar 100% pronto.
* **Trabalho Manual:** É praticamente impossível documentar, versionar no Git ou compartilhar a infraestrutura com a equipe de forma simples.

### 😩 Exemplo prático: subindo a mesma stack sem Compose

Para deixar a dor bem concreta, eis o que seria necessário para colocar no ar, **manualmente**, a mesma aplicação Flask + Redis que vamos orquestrar daqui a pouco:

```bash
# 1. Criar a rede manualmente
docker network create frontend-net

# 2. Criar o volume para persistência
docker volume create redis-data

# 3. Subir o Redis, lembrando de conectar na rede e no volume certos
docker run -d --name redis --network frontend-net -v redis-data:/data redis:7-alpine

# 4. Torcer para o Redis estar pronto a tempo (sem healthcheck, sem garantias)
sleep 5

# 5. Construir a imagem da aplicação web
docker build -t laboratorio-web .

# 6. Subir o container web, lembrando de mapear porta, rede e variável de ambiente
docker run -d --name web --network frontend-net -p 5000:5000 -e REDIS_HOST=redis laboratorio-web
```

Repare nos problemas: são **6 comandos manuais**, cada um com flags fáceis de esquecer (`--network`, `-v`, `-e`), nenhuma garantia real de que o Redis estará pronto antes do `sleep 5` acabar (em uma máquina sobrecarregada, 5 segundos pode não bastar), e nada disso fica documentado ou versionado — na próxima vez, alguém vai ter que redescobrir essa sequência exata.

É exatamente esse cenário que a função `get_hit_count()` do nosso `app.py` tenta mitigar com suas tentativas (`retries`) — mas depender de *retry* no código para compensar uma orquestração frágil é remendo, não solução.

---

## 4. A Solução: Docker Compose e Infraestrutura como Código (IaC)

A evolução natural na engenharia de sistemas é não dizer *como* o computador deve fazer (passo a passo), mas sim declarar *o que* queremos. Chamamos isso de **Infraestrutura como Código (IaC)**.

Para garantir a máxima compatibilidade com diversos laboratórios e servidores (incluindo o nosso ambiente de testes), utilizaremos o comando tradicional `docker-compose` (com hífen) e o arquivo padrão `docker-compose.yml` declarando a versão da sintaxe.

### Os quatro pilares do IaC

Quando declaramos infraestrutura como código, ganhamos quatro propriedades que o modelo imperativo não oferece:

* **Idempotência**: rodar `docker-compose up` dez vezes seguidas produz o mesmo resultado final — o Compose só recria o que mudou.
* **Versionamento**: o arquivo `docker-compose.yml` vai para o Git como qualquer outro código-fonte, com histórico completo de mudanças via `git log`.
* **Reprodutibilidade**: o mesmo arquivo sobe o mesmo ambiente na sua máquina, na do colega e no servidor de produção.
* **Documentação viva**: o arquivo *é* a documentação. Não existe "documentação desatualizada", porque o arquivo e o comportamento real nunca podem divergir.

### 🧭 Nota técnica: `docker-compose` vs. `docker compose`

Você vai notar que usamos o comando com hífen, `docker-compose`, ao longo deste laboratório — é a forma mais compatível com a maior variedade de ambientes e máquinas de laboratório, incluindo o Killercoda. Mas vale registrar o contexto para a turma:

* **Compose V1** (`docker-compose`, com hífen): ferramenta independente escrita em Python. Está oficialmente **descontinuada (end-of-life) desde julho de 2022** — só recebe correções de segurança críticas.
* **Compose V2** (`docker compose`, sem hífen): reescrita em Go e integrada diretamente à CLI do Docker como um plugin. É o padrão atual e recomendado para qualquer ambiente novo.

Na prática, a maioria das instalações modernas do Docker responde aos dois comandos (o V1 costuma estar "apelidado" para o V2 por trás dos panos). Se o seu ambiente pessoal usar uma instalação recente do Docker, sinta-se à vontade para usar `docker compose` (sem hífen) em vez de `docker-compose` — a sintaxe do arquivo YAML é idêntica.

> 📌 Outra mudança relevante: o campo `version:` no topo do `docker-compose.yml` (que usaremos na Seção 6) é hoje considerado **obsoleto** pela especificação atual do Compose — o Compose V2 ignora esse campo e sempre usa a especificação mais recente. Mantemos `version: '3.8'` neste guia por clareza didática e compatibilidade com ambientes mais antigos, mas não se assuste se, no seu Docker pessoal, aparecer um aviso (*warning*) dizendo que o atributo está obsoleto — é só um aviso, não um erro.

---

## 5. Anatomia do Compose YAML

O arquivo `docker-compose.yml` é a "planta baixa" da nossa infraestrutura. Ele define os "prédios" (serviços) que vamos construir. Vamos destrinchar cada bloco com mais profundidade do que cabe em um slide:

### `version:`

Define a versão da sintaxe do arquivo — utilizaremos a `3.8` para garantir suporte a *healthchecks* avançados no maior número possível de ambientes (veja a nota técnica da Seção 5 sobre sua obsolescência nas versões mais recentes do Compose).

### `build:` vs. `image:`

* **`build: .`** — o tijolo. Compila a imagem **localmente**, a partir do `Dockerfile` presente no diretório indicado (`.` = diretório atual). É o que faremos com o serviço `web`.
* **`image: redis:7-alpine`** — o pré-fabricado. Baixa uma imagem já pronta do Docker Hub, sem precisar buildar nada. É o que faremos com o serviço `redis`.

Um serviço pode até usar os dois juntos (`build` + `image`): nesse caso, o Compose builda a imagem localmente mas a "marca" (tag) com o nome definido em `image`, facilitando publicá-la depois em um registry.

### `ports:`

O túnel. Mapeia `"porta_do_host:porta_do_container"`. No nosso `web`, `"5000:5000"` significa "quem acessar a porta 5000 da máquina hospedeira cai na porta 5000 dentro do container". Repare que o `redis` só declara `"6379"` (sem a porta do host) — isso expõe uma porta efêmera aleatória ao hospedeiro, já que **ninguém de fora precisa falar com o Redis diretamente**; só o serviço `web` conversa com ele, via rede interna.

### `environment:` e `env_file:`

Injeção de variáveis de ambiente para alterar configurações sem modificar o código-fonte — é assim que `REDIS_HOST=redis` chega até o `os.environ.get('REDIS_HOST', 'redis')` do nosso `app.py`. Para poucas variáveis, `environment:` direto no YAML já é suficiente (nosso caso). Quando a lista cresce — ou quando há segredos, como senhas de banco — o mais comum é externalizar para um arquivo `env_file: .env`, que **nunca deve ser commitado no Git** (adicione-o ao `.gitignore`).

### `volumes:`

O Cofre-Forte. Existem dois tipos, e vale diferenciá-los em aula:

* **Volume nomeado** (o que usaremos: `redis-data:/data`): gerenciado pelo próprio Docker, vive fora do container e sobrevive a um `docker-compose down` sem `-v` — a forma recomendada para dados de banco de dados.
* **Bind mount** (ex.: `./app:/code`): mapeia uma pasta real do seu hospedeiro diretamente para dentro do container. Muito usado em desenvolvimento, para ver mudanças de código refletidas sem rebuildar a imagem — mas menos portável entre máquinas.

### `networks:`

Isolamento e organização. Serviços na mesma `network:` se enxergam pelo nome; serviços em redes diferentes ficam isolados entre si por padrão — útil, por exemplo, para impedir que um serviço de frontend acesse diretamente um banco de dados que só deveria falar com o backend.

> **A Regra de Ouro da Rede:** Nunca decore ou fixe IPs em contêineres. O próprio nome do serviço definido no YAML torna-se o hostname oficial, sendo resolvido automaticamente pelo DNS interno do Docker.

### `depends_on:` — simples vs. com `condition`

Existe uma pegadinha importante aqui que vale reforçar em aula: `depends_on` **sem** `condition` só garante que o container dependente foi *iniciado* — não que a aplicação dentro dele já está *pronta* para receber conexões. Um banco de dados pode "iniciar" em 200ms e ainda levar alguns segundos para aceitar conexões. É por isso que, no nosso laboratório, usamos a forma estendida:

```yaml
depends_on:
  redis:
    condition: service_healthy
```

Isso diz ao Compose: "só inicie o `web` depois que o `healthcheck` do `redis` reportar `healthy`" — resolvendo de vez o problema de corrida que vimos na Seção 4.

### `healthcheck:`

O próprio orquestrador roda um comando de teste dentro do container periodicamente para decidir se ele está saudável:

* **`test:`** o comando de verificação (`redis-cli ping`, que responde `PONG` se o Redis estiver operacional).
* **`interval:`** de quanto em quanto tempo testar (a cada 5s).
* **`timeout:`** quanto tempo esperar por uma resposta antes de considerar falha (3s).
* **`retries:`** quantas falhas seguidas até marcar o serviço como `unhealthy` (5).

### `restart:` (bônus — não usado no laboratório, mas essencial em produção)

Define o que o Docker faz se um container cair inesperadamente:

| Valor | Comportamento |
| :--- | :--- |
| `no` (padrão) | Nunca reinicia sozinho |
| `always` | Sempre reinicia, mesmo após reboot da máquina |
| `on-failure` | Só reinicia se o processo sair com erro (código ≠ 0) |
| `unless-stopped` | Reinicia sempre, exceto se foi parado manualmente |

### Indo além (mencione — não é necessário no lab de hoje)

* **`profiles:`** marca serviços como "opcionais", que só sobem quando explicitamente ativados (`docker-compose --profile debug up`) — útil para ferramentas de debug que não devem rodar em todo `up`.
* **`deploy.replicas`** (ou a flag `--scale`) permite rodar múltiplas cópias do mesmo serviço — veremos isso na Seção 10 como ponte para o Kubernetes.

---

## 6. Ambiente Prático: Killercoda

Para o laboratório de hoje, usaremos uma solução sem fricção: o **Killercoda**.

* Fornece uma VM Ubuntu nativa com sessão de 60 minutos ininterruptos.
* O Docker Engine já vem instalado e pronto para uso.
* **⚠️ Aviso Crítico:** Nunca pressione `F5` ou recarregue a aba do navegador durante o laboratório. Isso destrói a máquina virtual instantaneamente.

👉 **Link de Acesso:** [Ubuntu Playground no Killercoda](https://killercoda.com/playgrounds/scenario/ubuntu)

> 🧪 **Quer praticar depois da aula?** O Killercoda não é a única opção sem instalação local. O [Play with Docker](https://labs.play-with-docker.com/) oferece algo parecido, e muitos templates do GitHub Codespaces já vêm com Docker pré-instalado. Para uso contínuo, o Docker Desktop (Mac/Windows) ou o pacote `docker.io`/`docker-ce` (Linux) resolvem localmente.

---

## 7. Laboratório Passo a Passo (Hands-on)

Nosso projeto prático consiste em um Frontend Web (API em Python/Flask) e um Backend (Banco de dados Redis). A aplicação conta o número de visitas e armazena esse dado no banco.

### Passo 1: Preparando o Terreno

No terminal do Killercoda, verifique se o Compose está rodando e crie a pasta do projeto:

```bash
docker-compose --version
mkdir ~/laboratorio-compose && cd ~/laboratorio-compose
```
> 💡 **Dica de Produtividade:** Copie os blocos de código abaixo, cole inteiros no terminal e pressione `ENTER`. O comando `cat << 'EOF'` criará os arquivos automaticamente sem precisarmos abrir editores de texto no terminal.

### Passo 2: O Código da Aplicação (`app.py`)

Antes de executarmos o código, é fundamental entender o comportamento dessa aplicação. Esta é uma API Web construída com o microframework Flask em Python. 
O grande diferencial aqui é a **resiliência da conexão**: em ambientes distribuídos, o banco de dados pode demorar alguns segundos a mais para inicializar. Por isso, implementamos a função `get_hit_count()`, que possui um laço de repetição (`while True`) com um limite de tentativas (`retries`). Se o banco não estiver pronto, a aplicação aguarda meio segundo e tenta de novo, evitando que o contêiner falhe (crash) logo na inicialização. A conexão com o banco é feita dinamicamente através da variável de ambiente `REDIS_HOST`[cite: 1].

```bash
cat << 'EOF' > app.py
import time
import os
import redis
from flask import Flask

app = Flask(__name__)
redis_host = os.environ.get('REDIS_HOST', 'redis')
cache = redis.Redis(host=redis_host, port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def get_index():
    count = get_hit_count()
    return f'<h1>Laboratório Docker Compose Multi-Serviços</h1><p>Esta página foi visualizada <strong>{count}</strong> vezes.</p>'

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
```

### Passo 3: A Receita do Contêiner (`Dockerfile`)

O `Dockerfile` é a "receita de bolo" que diz ao motor do Docker como empacotar nossa aplicação. 
Vamos destrinchar cada instrução:
* `FROM python:3.10-alpine`: Utiliza uma versão minimalista do Linux (Alpine) com Python 3.10, resultando em uma imagem final extremamente leve e segura.
* `WORKDIR /code`: Define o diretório de trabalho padrão dentro do contêiner.
* `RUN pip install...`: Executa a instalação das dependências (Flask e Redis) durante a fase de construção (build).
* `COPY app.py .`: Move o nosso código fonte do sistema hospedeiro para dentro do contêiner.
* `CMD`: Especifica o comando padrão que manterá o contêiner em execução (rodando o servidor Python).

```bash
cat << 'EOF' > Dockerfile
FROM python:3.10-alpine
WORKDIR /code
RUN pip install flask redis
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

### Passo 4: O Maestro (`docker-compose.yml`)

Este é o coração da orquestração e o principal objetivo da nossa aula. Em vez de rodarmos múltiplos comandos imperativos propensos a falhas manuais, declaramos o estado desejado da nossa infraestrutura[cite: 2].
Pontos cruciais deste manifesto:
1. **Rede Interna (`frontend-net`)**: Garante que o serviço `web` e o banco `redis` consigam se comunicar de forma isolada do mundo externo.
2. **Dependência Inteligente (`depends_on` e `healthcheck`)**: O serviço `web` só será iniciado quando o Redis estiver com o status "saudável". O próprio orquestrador fará um ping no banco a cada 5 segundos; ao receber a resposta afirmativa, ele libera o início da aplicação web[cite: 1, 2].
3. **Persistência (`volumes`)**: O `redis-data` ancora os dados do banco de dados no disco físico da máquina hospedeira, garantindo que o ciclo de vida efêmero do contêiner não destrua nossas informações em caso de reinicialização.

```bash
cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - frontend-net

  redis:
    image: redis:7-alpine
    ports:
      - "6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - frontend-net
    volumes:
      - redis-data:/data

networks:
  frontend-net:
    driver: bridge

volumes:
  redis-data:
EOF
```

### Passo 5: Fazendo a Mágica Acontecer

Agora que temos a infraestrutura descrita como código (IaC), podemos levantar todo o ambiente com um único comando declarativo. A *flag* `-d` (detached mode) é utilizada para rodar o processo em segundo plano, liberando o nosso terminal para continuar operando[cite: 1, 2]. A flag -d também previne que em ambientes remotos via ssh ou fechar um terminal sem querer derrube toda a infraestrutura

```bash
docker-compose up -d
```

Para inspecionar o comportamento da arquitetura em tempo real e visualizar os logs combinados do Python e do Redis, execute:

```bash
docker-compose logs -f
```
*(Para sair da tela de logs em tempo real sem desligar os contêineres, pressione `Ctrl + C`)*.
A flag -f no comando acima segue os logs em tempo reais, sem elas só seria impresso os logs e atuais e pararia. O docker compose unifica os logs dos conteiners para facilitar o rastreamento de erros. Por exemplo um erro que começa no front-end e estoura no back.

**Testando a Aplicação:**
No terminal, faça requisições simulando acessos de usuários para ver a contagem subir iterativamente[cite: 1]:
```bash
curl http://localhost:5000
```
Para ver a interface no navegador de forma gráfica pelo Killercoda, clique na opção **Traffic / Ports** localizada no menu superior, digite a porta `5000` e acesse a página[cite: 1].

### Passo 6: O Teste de Resiliência (Simulação de Caos)

Uma infraestrutura madura deve ser capaz de se recuperar de falhas. E se o nosso banco de dados sofrer um *crash* ou precisar ser reiniciado? Vamos perder o histórico de contagens de acessos[cite: 1]? Vamos colocar isso à prova:

1. Vamos forçar a reinicialização apenas do contêiner do banco de dados:
```bash
docker-compose restart redis
```
2. Acesse a aplicação novamente pelo terminal ou atualize a página no navegador:
```bash
curl http://localhost:5000
```
**Resultado Prático:** O contador continuará exatamente de onde parou! Isso comprova na prática o conceito de **desacoplamento**: a execução (que é volátil e efêmera) foi separada do armazenamento de estado (que está ancorado e seguro no volume `redis-data` do hospedeiro)[cite: 1]. Essa propriedade do redis e a configuração do volume é o que permite classifica-lo como um banco de dados stateful, que é um banco de dados que armazena informações importantes diferente de um stateless onde subir um igual não afetaria o usuário. 

### Passo 7: Demolição Limpa

A engenharia de software eficiente também envolve a gestão correta de recursos. Deixar contêineres e redes fantasmas rodando consome memória e processamento do servidor de forma desnecessária[cite: 2]. A flag `-v` é essencial aqui para garantir que os volumes também sejam destruídos ao encerrarmos o laboratório[cite: 2].

```bash
docker-compose down -v
```

---

## 8. Boas Práticas e Erros Comuns

A aula de hoje usa um exemplo enxuto de propósito, mas ele já segue várias boas práticas de mercado — vale apontar isso durante a explicação. Aqui vai um checklist rápido, útil tanto para revisar o que já fizemos quanto para evitar armadilhas comuns em projetos reais:

* ✅ **Fixe (pin) a tag da imagem.** Usamos `redis:7-alpine` e `python:3.10-alpine`, não `redis:latest`. Tags fixas evitam que um `docker-compose up` de amanhã baixe uma versão diferente (e possivelmente quebrada) sem avisar.
* ✅ **Prefira imagens `alpine`** quando possível — são drasticamente menores, o que acelera build e deploy (atenção: `alpine` usa `musl` em vez de `glibc`, o que raramente pode causar incompatibilidades com certas bibliotecas nativas).
* 🔒 **Nunca coloque senhas ou chaves de API direto no `environment:`** de um arquivo versionado no Git. Use `env_file: .env` com o `.env` no `.gitignore`, ou um gerenciador de segredos.
* 🌐 **Não exponha portas que não precisam ser públicas.** Nosso `redis` não mapeia porta fixa para o hospedeiro — só o `web` precisa ser público.
* 🧹 **Adicione um `.dockerignore`.** Sem ele, o `docker build` pode copiar `.git/`, `node_modules/` ou arquivos de ambiente sensíveis para dentro da imagem sem querer.
* 🩺 **Defina `healthcheck` em todo serviço do qual outros dependem** — como fizemos com o `redis`. Sem isso, `depends_on` sozinho é uma falsa sensação de segurança.
* 🗑️ **Rode limpezas periódicas.** Containers parados, imagens intermediárias e redes órfãs se acumulam com o tempo; `docker system prune` (com cautela) resolve.
* 📝 **Valide antes de subir.** `docker-compose config` renderiza o YAML final (após interpolar variáveis) sem executar nada — ótimo para pegar erros de sintaxe antes de rodar `up`.

---

## 9. O Próximo Nível: Escala e Kubernetes

Onde o Compose termina e a escala massiva começa?

* **Docker Compose (Single-Host):** É a ferramenta definitiva para o ciclo de desenvolvimento local, fluxos ágeis de automação de testes (CI/CD) e implantações pontuais em servidores únicos (como arquiteturas enxutas em nuvem).
* **Orquestradores de Cluster (Kubernetes / Swarm):** Foram projetados para ambientes de produção massivos, distribuindo a carga entre múltiplos servidores de hardware simultâneos, garantindo balanceamento de carga global e alta disponibilidade corporativa (*auto-healing*).

### Comparando as três opções

| Critério | Docker Compose | Docker Swarm | Kubernetes |
| :--- | :--- | :--- | :--- |
| Hosts suportados | 1 (single-host) | Múltiplos (cluster) | Múltiplos (cluster) |
| Curva de aprendizado | Baixa | Média | Alta |
| Auto-healing | ❌ Não | ✅ Sim | ✅ Sim |
| Auto-scaling | ❌ Manual (`--scale`) | ⚠️ Limitado | ✅ Avançado (HPA) |
| Load balancing | ❌ Não nativo | ✅ Sim | ✅ Sim (avançado) |
| Ecossistema/comunidade | Grande | Pequeno | Enorme (padrão de mercado) |
| Melhor para | Dev local, CI/CD, apps pequenas | Times que já usam Docker e querem simplicidade | Produção em escala, times de plataforma dedicados |

### Quando migrar do Compose?

Alguns sinais de que chegou a hora de considerar um orquestrador de cluster:

* Você precisa distribuir carga entre **mais de um servidor físico**.
* *Downtime* é inaceitável e você precisa de *failover* automático.
* O time cresceu e precisa de controles de acesso, quotas e isolamento mais sofisticados (multi-tenancy).
* Você já está fazendo *scaling* manual com `--scale` com frequência e sente falta de automação.

A mentalidade declarativa (IaC) e as abstrações de redes, dependências e volumes que praticamos hoje com o Compose são exatamente a mesma base arquitetural exigida para dominar tecnologias como o Kubernetes no seu futuro profissional.

---

## 10. Perguntas Frequentes (FAQ)

**O Docker Compose substitui o Dockerfile?**
Não. São ferramentas complementares. O `Dockerfile` define **como construir uma imagem** (uma receita). O `docker-compose.yml` define **como orquestrar múltiplos containers/serviços** — que podem usar imagens construídas via `Dockerfile` (como nosso `web`) ou baixadas prontas de um registry (como nosso `redis`).

**Por que meus dados sumiram depois de um `docker-compose down`?**
Provavelmente foi usada a flag `-v` (`docker-compose down -v`), que remove os volumes junto com os containers e redes. Sem `-v`, os volumes nomeados (como o `redis-data`) sobrevivem normalmente — foi exatamente isso que comprovamos no Passo 6 do laboratório.

**Devo usar `docker-compose` ou `docker compose` (sem hífen)?**
Para qualquer ambiente novo, prefira `docker compose` (sem hífen, Compose V2) — é o padrão atual e mantido ativamente. Usamos a forma com hífen neste guia apenas por compatibilidade ampla com o ambiente do Killercoda (veja a nota técnica na Seção 5).

**Posso rodar várias instâncias do mesmo serviço?**
Sim: `docker-compose up -d --scale web=3` sobe três instâncias do serviço `web`. Cuidado: se o serviço mapear uma porta fixa do host (como `"5000:5000"`), as instâncias vão brigar pela mesma porta — nesse caso, remova o mapeamento fixo e deixe o Docker escolher portas aleatórias, ou coloque um *load balancer* na frente.

**Se eu editar o `docker-compose.yml` com os containers já rodando, a mudança é aplicada na hora?**
Não automaticamente. É preciso rodar `docker-compose up -d` de novo — o Compose compara o estado desejado (arquivo) com o estado atual e recria apenas os serviços que mudaram.

**O Compose funciona em produção?**
Pode funcionar bem para aplicações pequenas ou médias rodando em um único servidor. Para produção crítica, multi-servidor ou de alta disponibilidade, o caminho natural é migrar para Kubernetes ou Docker Swarm (Seção 10).

---

## 11. Cheat Sheet (Comandos Úteis)

| Comando | Descrição |
| :--- | :--- |
| `docker-compose up -d` | Sobe todos os serviços em segundo plano (*detached*). |
| `docker-compose ps` | Lista os contêineres ativos do projeto e seus status. |
| `docker-compose logs -f` | Exibe e acompanha os logs consolidados em tempo real. |
| `docker-compose logs -f <serviço>` | Acompanha os logs de apenas um serviço específico. |
| `docker-compose restart <serviço>`| Reinicia um serviço específico (ex: `redis`). |
| `docker-compose stop` / `start` | Para/inicia os containers sem removê-los (mantém o estado). |
| `docker-compose exec <serviço> <cmd>` | Executa um comando dentro de um container já rodando (ex: abrir um shell). |
| `docker-compose build` | Reconstrói as imagens definidas com `build:`, sem subir os containers. |
| `docker-compose config` | Valida e imprime o YAML final (após interpolação de variáveis). |
| `docker-compose top` | Mostra os processos em execução dentro de cada container. |
| `docker-compose up -d --scale <serviço>=N` | Sobe N réplicas de um serviço específico. |
| `docker-compose down` | Para e remove os contêineres e a rede. |
| `docker-compose down -v` | Para e remove contêineres, rede e **volumes de dados**. |

> 📚 **Referência oficial:** para ir além do que cabe em 45 minutos, a [documentação oficial do Docker Compose](https://docs.docker.com/compose/) é o melhor próximo passo — está sempre atualizada com a especificação mais recente.
