# 🐳 Docker e Docker Compose: Do Caos à Orquestração em 45 Minutos

Bem-vindo(a) ao repositório oficial da nossa aula prática sobre orquestração de contêineres! 

Este material foi desenhado para levar você dos conceitos fundamentais de isolamento de software até a implantação de uma infraestrutura multi-serviços completa utilizando **Docker Compose**. Tudo isso em 45 minutos, rodando 100% na nuvem, sem precisar instalar nada na sua máquina.

---

## 📚 Sumário
1. [Fundamentos: O Fim do "Na minha máquina funciona"](#1-fundamentos-o-fim-do-na-minha-máquina-funciona)
2. [O Problema: Gerenciamento Imperativo](#2-o-problema-gerenciamento-imperativo)
3. [A Solução: Docker Compose e IaC](#3-a-solução-docker-compose-e-iac)
4. [Anatomia do Compose YAML](#4-anatomia-do-compose-yaml)
5. [Ambiente Prático: Killercoda](#5-ambiente-prático-killercoda)
6. [Laboratório Passo a Passo (Hands-on)](#6-laboratório-passo-a-passo-hands-on)
7. [O Próximo Nível: Escala e Kubernetes](#7-o-próximo-nível-escala-e-kubernetes)
8. [Cheat Sheet (Comandos Úteis)](#8-cheat-sheet-comandos-úteis)

---

## 1. Fundamentos: O Fim do "Na minha máquina funciona"

No desenvolvimento de software tradicional, configurar laboratórios ou servidores do zero exige instalar dependências complexas. Se você já precisou subir máquinas virtuais completas (com sistemas operacionais inteiros) apenas para rodar um servidor web ou um banco de dados, sabe que o processo consome muito disco, memória e tempo de configuração.

**A Solução (Docker):** O Docker utiliza a tecnologia de **contêineres**. Ele empacota o código da sua aplicação (por exemplo, scripts em Python), as bibliotecas necessárias e as configurações de sistema em uma única unidade padronizada e imutável.

Diferente de uma Máquina Virtual que emula o hardware e carrega um sistema operacional pesado (Guest OS), os contêineres compartilham o *kernel* do sistema hospedeiro, inicializando em milissegundos e consumindo uma fração dos recursos.

## 2. O Problema: Gerenciamento Imperativo

Antes da orquestração automatizada, subíamos a infraestrutura comando por comando. Esse modelo gera vários problemas:

* **Risco Humano:** Esquecer *flags* de rede ou variáveis de ambiente quebra o sistema.
* **Ordem de Execução:** É difícil garantir que a aplicação web só inicie após o banco de dados estar 100% pronto.
* **Trabalho Manual:** É praticamente impossível documentar, versionar no Git ou compartilhar a infraestrutura com a equipe de forma simples.

## 3. A Solução: Docker Compose e IaC

A evolução natural na engenharia de sistemas é não dizer *como* o computador deve fazer (passo a passo), mas sim declarar *o que* queremos. Chamamos isso de **Infraestrutura como Código (IaC)**.

Para garantir a máxima compatibilidade com diversos laboratórios e servidores (incluindo o nosso ambiente de testes), utilizaremos o comando tradicional `docker-compose` (com hífen) e o arquivo padrão `docker-compose.yml` declarando a versão da sintaxe.

## 4. Anatomia do Compose YAML

O arquivo `docker-compose.yml` é a "planta baixa" da nossa infraestrutura. Ele define os "prédios" (serviços) que vamos construir:

* `version:` Define a versão da sintaxe do arquivo (utilizaremos a `3.8` para garantir suporte a *healthchecks* avançados).
* `build:` O tijolo. Constrói a imagem localmente a partir de um arquivo `Dockerfile`[cite: 2].
* `image:` O pré-fabricado. Baixa uma imagem pronta diretamente do Docker Hub[cite: 2].
* `ports:` O túnel. Mapeia a porta pública do sistema hospedeiro para a porta privada do contêiner[cite: 2].
* `environment:` Injeção de variáveis de ambiente para alterar configurações sem modificar o código-fonte[cite: 2].
* `volumes:` O Cofre-Forte. Ancoragem de dados em disco físico, garantindo que informações importantes sobrevivam à destruição do contêiner[cite: 2].

> **A Regra de Ouro da Rede:** Nunca decore ou fixe IPs em contêineres. O próprio nome do serviço definido no YAML torna-se o hostname oficial, sendo resolvido automaticamente pelo DNS interno do Docker[cite: 2].

---

## 5. Ambiente Prático: Killercoda

Para o laboratório de hoje, usaremos uma solução sem fricção: o **Killercoda**[cite: 2].

* Fornece uma VM Ubuntu nativa com sessão de 60 minutos ininterruptos[cite: 2].
* O Docker Engine já vem instalado e pronto para uso.
* **⚠️ Aviso Crítico:** Nunca pressione `F5` ou recarregue a aba do navegador durante o laboratório. Isso destrói a máquina virtual instantaneamente[cite: 2].

👉 **Link de Acesso:** [Ubuntu Playground no Killercoda](https://killercoda.com/playgrounds/scenario/ubuntu)

---

## 6. Laboratório Passo a Passo (Hands-on)

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

Agora que temos a infraestrutura descrita como código (IaC), podemos levantar todo o ambiente com um único comando declarativo. A *flag* `-d` (detached mode) é utilizada para rodar o processo em segundo plano, liberando o nosso terminal para continuar operando[cite: 1, 2].

```bash
docker-compose up -d
```

Para inspecionar o comportamento da arquitetura em tempo real e visualizar os logs combinados do Python e do Redis, execute:

```bash
docker-compose logs -f
```
*(Para sair da tela de logs em tempo real sem desligar os contêineres, pressione `Ctrl + C`)*.

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
**Resultado Prático:** O contador continuará exatamente de onde parou! Isso comprova na prática o conceito de **desacoplamento**: a execução (que é volátil e efêmera) foi separada do armazenamento de estado (que está ancorado e seguro no volume `redis-data` do hospedeiro)[cite: 1].

### Passo 7: Demolição Limpa

A engenharia de software eficiente também envolve a gestão correta de recursos. Deixar contêineres e redes fantasmas rodando consome memória e processamento do servidor de forma desnecessária[cite: 2]. A flag `-v` é essencial aqui para garantir que os volumes também sejam destruídos ao encerrarmos o laboratório[cite: 2].

```bash
docker-compose down -v
```

---

## 7. O Próximo Nível: Escala e Kubernetes

Onde o Compose termina e a escala massiva começa[cite: 2]?

* **Docker Compose (Single-Host):** É a ferramenta definitiva para o ciclo de desenvolvimento local, fluxos ágeis de automação de testes (CI/CD) e implantações pontuais em servidores únicos (como arquiteturas enxutas em nuvem)[cite: 2].
* **Orquestradores de Cluster (Kubernetes / Swarm):** Foram projetados para ambientes de produção massivos, distribuindo a carga entre múltiplos servidores de hardware simultâneos, garantindo balanceamento de carga global e alta disponibilidade corporativa (*auto-healing*)[cite: 2].

A mentalidade declarativa (IaC) e as abstrações de redes, dependências e volumes que praticamos hoje com o Compose são exatamente a mesma base arquitetural exigida para dominar tecnologias como o Kubernetes no seu futuro profissional[cite: 2].

---

## 8. Cheat Sheet (Comandos Úteis)

| Comando | Descrição |
| :--- | :--- |
| `docker-compose up -d` | Sobe todos os serviços em segundo plano (*detached*). |
| `docker-compose ps` | Lista os contêineres ativos do projeto e seus status. |
| `docker-compose logs -f` | Exibe e acompanha os logs consolidados em tempo real. |
| `docker-compose restart <servico>`| Reinicia um serviço específico (ex: `redis`). |
| `docker-compose down` | Para e remove os contêineres e a rede. |
| `docker-compose down -v` | Para e remove contêineres, rede e **volumes de dados**. |
