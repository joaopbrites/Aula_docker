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

* **O Padrão Atual:** O Docker Compose V2 substitui o antigo `docker-compose` (escrito em Python) por uma integração nativa (`docker compose`, sem hífen, escrito em Go).
* **Interpretação Dinâmica:** No padrão atual, não é mais necessário declarar a versão (`version: '3'`) no topo do arquivo YAML; o Docker moderno infere as features dinamicamente[cite: 2].

## 4. Anatomia do Compose YAML

O arquivo `compose.yaml` é a "planta baixa" da nossa infraestrutura. Ele define os "prédios" (serviços) que vamos construir[cite: 2]:

* `build:` O tijolo. Constrói a imagem localmente a partir de um arquivo `Dockerfile`[cite: 2].
* `image:` O pré-fabricado. Baixa uma imagem pronta diretamente do Docker Hub[cite: 2].
* `ports:` O túnel. Mapeia a porta pública do sistema hospedeiro para a porta privada do contêiner[cite: 2].
* `environment:` Injeção de variáveis de ambiente para alterar configurações sem modificar o código-fonte[cite: 2].
* `volumes:` O Cofre-Forte. Ancoragem de dados em disco físico, garantindo que informações importantes sobrevivam à destruição do contêiner[cite: 2].

> **A Regra de Ouro da Rede:** Nunca decore ou fixe IPs em contêineres. O próprio nome do serviço definido no YAML torna-se o hostname oficial, sendo resolvido automaticamente pelo DNS interno do Docker[cite: 2].

---

## 5. Ambiente Prático: Killercoda

Para o laboratório de hoje, usaremos uma solução sem fricção: o **Killercoda**[cite: 2].

* Fornece uma VM Ubuntu 24.04 nativa com sessão de 60 minutos ininterruptos[cite: 2].
* O Docker Engine e o plugin Compose V2 já vêm instalados.
* **⚠️ Aviso Crítico:** Nunca pressione `F5` ou recarregue a aba do navegador durante o laboratório. Isso destrói a máquina virtual instantaneamente[cite: 2].

👉 **Link de Acesso:** [Ubuntu Playground no Killercoda](https://killercoda.com/playgrounds/scenario/ubuntu)

---

## 6. Laboratório Passo a Passo (Hands-on)

Nosso projeto prático consiste em um Frontend Web (API em Python/Flask) e um Backend (Banco de dados Redis). A aplicação conta o número de visitas e armazena esse dado no banco.

### Passo 1: Preparando o Terreno

No terminal do Killercoda, verifique se o Compose V2 está rodando e crie a pasta do projeto:

```bash
docker compose version
mkdir ~/laboratorio-compose && cd ~/laboratorio-compose
