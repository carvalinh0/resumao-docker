# Resumão de Docker + Docker compose + Kubernetes (k8s)

## O que caralhos é Docker?

Vamos lá seus porras, imagina que tu tem um código de API no github que usa npm + postgres + PrismaORM com um processo de compilação com esbuild + um client que usa Python e precisa compartilhar esses códigos com teu amante ou precisa subir num servidorzinho da Hostinger, você já imagina que terá que criar uma documentação de como rodar o projeto e como usar/inicializar o código e suas dependências (aqui seria o banco de dados postgres). Fazer isso é um trabalhão, cada um dos devs/servidores teriam que fazer `npm install` + `npx prisma generate` + `npm build` + `npm run dev/prod`, e pior ainda para o python onde seus usuários teriam que usar a versão exata do python que você está usando no projeto e ter que ficar usando o venv a todo momento. O Docker resolve isso tudo e o famoso "na minha máquina funciona", mas como?

O Docker é muito parecido com uma VM, a diferença é que um VM cria um sistema operacional inteiro junto com sua aplicação e usa um tamanho fixo de memória, disco e CPU, isso cria N limitações em grande escala. O Docker resolve isso empacotando toda a sua aplicação numa imagem, uma imagem pode ser interpretada como um template da sua aplicação, esperando para ser executado como um processo (container). Dentro da imagem tem tudo que sua aplicação precisa para rodar seu projeto e nada mais. Isso resolve o problema de memória pois como se trata de um processo então ele "não tem um limite de memória", disco pois ele pode criar discos virtuais que chamamos de volumes, que veremos mais pra frente e de CPU pois não roda como um processo qualquer.

O Docker roda principalmente em cima do sistema operacional, em especifico do linux. Se você já baixou o Docker Desktop no seu Windows você pode ter percebido que ele cria uma Maquina virtual do Docker, que por baixo dos panos seria uma maquina virtual linux. No linux ele pode já até vir instalado por padrão no OS como é o caso do Fedora.

## O BASICO de Docker

Primeiramente, supomos que você está usando pastas diferentes para o seu projeto de API e para o projeto do Cliente, veja a estrutura de pastas que estamos usando:

```txt
📦projeto
 ┣ 📂api
 ┃ ┗ 📂src
 ┃ ┗ 📜package-lock.json
 ┃ ┗ 📜package.json
 ┃ ┗ 📜Dockerfile
 ┗ 📂aplicacao
 ┃ ┗ 📜main.py
 ┃ ┗ 📜Dockerfile
 ┗ 📜docker-compose.yml
```

Agora vamos começar pela API e explicar os comandos passo a passo.

```dockerfile
# ./api/Dockerfile
 
# FROM - Usado para definir qual imagem você quer usar para rodar os comandos seguintes,
#   veja que usamos node, mas poderia ser Bun ou Python para diferentes aplicações. As imagens
#   não se limitam a runtimes, mas normalmente usamos imagens de runtimes para aplicações mais
#   simples e diretas, como é o nosso caso.
# AS - Usado para nomear (alias) esse estágio do build, nos comandos seguintes veremos o porquê.
FROM node:20-alpine AS build
 
# WORKDIR - Usado para definir em qual pasta estará nosso código. Se você abrir um container
# docker enquanto ele está ativo, verá que ele se assemelha muito às pastas que vemos no
# diretório "/" do Linux, mas não se engane, isso é uma cópia dos diretórios e não um OS (lembra
# que rodamos o Docker em cima do OS e não com um OS independente como uma VM). Mesmo assim
#   você normalmente consegue acessar o container como se fosse uma máquina Linux e executar
#   comandos como ls, pwd, grep e etc. Em imagens mais leves como é o caso de imagens slim ou
#   alpine (ex: python:3.14-alpine3.24) esses comandos podem não funcionar pois imagens como
#   essas cortam diversos recursos para deixar a imagem final mais leve e quanto menos passos
#   e mais leve a sua imagem final melhor.
WORKDIR /app
 
# COPY - Copia arquivos do seu projeto para a sua imagem, começamos primeiro com arquivos de
#    libs como o package.json. Note que você pode copiar mais de um arquivo para ./ e que
#    nesse caso o ./ vai apontar para /app pois definimos no comando anterior que o nosso
#    projeto ficará armazenado no /app.
COPY package.json package-lock.json ./
 
# RUN - Usado para executar comandos durante o build da imagem. E qual a diferença entre RUN,
#   CMD e ENTRYPOINT? CMD é usado para definir qual é o comando para inicializar nossa imagem,
# sendo o primeiro item do array o binário executável e que normalmente sempre será o npm
# e depois o "run start" no caso de aplicações com Node.js, sempre teremos apenas um CMD
# no final do build. O ENTRYPOINT é usado para definir um comando que é usado sempre 
#   antes de todos os outros comandos, então se eu definir o ENTRYPOINT como "npm" então
#   eu poderia rodar RUN ci diretamente pois ele injetaria o npm antes.
RUN npm ci
 
# Depois de instalar as dependências, copiamos o resto dos arquivos do projeto. Por que não
# copiamos tudo de uma vez? Bem, por dois motivos:
#   1. Ao usar o COPY separado, estamos aproveitando o cache do Docker para acelerar o build da
# imagem, pois se as dependências não alterarem, então ele copia da última versão da imagem e
# isso deixa o tempo de build muito mais curto.
#   2. Nem sempre você terá o node_modules na sua pasta local, então rodar o build sem ela
#   causaria um erro, pois você não tem as dependências necessárias para rodar sua aplicação.
COPY . .
 
# Fazemos o build da API (comando específico do projeto)
RUN npm run build
 
# Agora note que usamos uma imagem mais leve (alpine) para montar a imagem para produção, isso
# que estamos fazendo se chama Multi Stage Build e é uma boa prática para deixar sua imagem
# final ultra leve.
FROM node:20-alpine AS production
 
# Definimos novamente a pasta onde vamos deixar nosso código. Note que agora estamos usando outra imagem e, portanto, outro ambiente.
WORKDIR /app
 
# Copia apenas o que precisamos para rodar a imagem em produção e instala as dependências.
# Deve estar se perguntando agora o porquê instalamos as dependências novamente, veja, o
# build da aplicação precisa de pacotes do ambiente de desenvolvimento mas rodar o que o build
# gera não, então instalamos apenas os pacotes de produção com --omit=dev e reduzimos o
# tamanho do node_modules que será gerado na instalação.
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
 
# Copiamos o build gerado na imagem "build" para /app/dist
COPY --from=build /app/dist ./dist
 
# EXPOSE - Usamos esse comando para expor uma porta do container, no caso dessa API ela usa a
#   porta 3000 então abrimos a porta 3000 do container para ele ser acessado por fora.
EXPOSE 3000
 
# VOLUME - Por padrão o Docker armazena os arquivos que você cria quando o container é
#   executado dentro do próprio container, isso acaba sendo um problema quando o container
#   morre, pois quando isso acontece todos os dados morrem junto com o container.
#   Ao usar um volume o Docker irá criar um volume flexível no disco e ele será montado
#   (https://linux.die.net/man/8/mount) num diretório dentro do container e tudo que é salvo
#   dentro desse diretório é salvo no volume (como se fosse um pendrive). Você pode definir o
#   volume dentro do Dockerfile ou do docker-compose que veremos no próximo tópico.
VOLUME /data /logs /config
 
# USER - Criamos um usuário para rodar a aplicação, isso é recomendado em produção pois se
#   tivermos o container invadido o hacker não consegue extrair muito desse container e nem
#   dos outros no qual podemos estar conectado, assim como um usuário sem permissões deveria
#   executar por padrão.
USER node
 
# Aqui o comando de inicialização que comentamos antes.
CMD ["npm", "run", "start"]
```

Dica: Normalmente chamamos o build da imagem de Dockerfile, mas isso não é regra.

Agora vamos ver um build simples da aplicação que serve para a maioria dos casos de build.

```dockerfile
FROM python:3.14-alpine
 
WORKDIR /app
 
COPY requirements.txt .
 
RUN pip install -r requirements.txt
 
COPY . .
 
CMD ["python", "main.py"]
```

Ambos os arquivos podem ser executados usando o `docker build -t nome-da-imagem .` e depois `docker run nome-da-imagem` mas quando temos banco de dados e múltiplos serviços, usamos o Docker Compose.

## Docker compose

Imagine o Dockerfile como uma receita de "como montar sua imagem" agora imagine o Docker compose como um manual de "como rodar a sua imagem" ou de "como rodar múltiplos serviços juntos". Isso porque você pode sim usar o Docker Compose para apenas um serviço.

Vamos ver como ficaria o arquivo docker-compose.yml e explicar os comandos passo a passo (aqui o nome do arquivo também não precisa ser o mesmo, mas o padrão é esse, pois assim não precisamos definir o arquivo ao rodar o comando do compose que veremos).

```yml
services:
  # SERVIÇO 1: API
  api:
    # build - Em vez de usar uma imagem pronta do Docker Hub, dizemos para o Compose 
    # compilar o Dockerfile que está dentro da pasta ./api
    build: ./api

    # restart - Define a política de reinicialização. "on-failure" significa que se a API 
    # quebrar por um erro (crash), o Docker tenta reiniciar o container automaticamente.
    restart: on-failure

    # environment - Injeta variáveis de ambiente no container.
    # Repare na DATABASE_URL: o hostname do banco não é "localhost", é "postgres"!
    # O Docker Compose cria uma rede interna onde os containers se comunicam pelo nome do serviço.
    environment:
      - DATABASE_URL=postgresql://postgres:prisma@postgres:5432/postgres
      - PORT=3000
    
    # ports - Mapeamento de portas na sintaxe "SUA_MÁQUINA:CONTAINER".
    # A porta 3000 da sua máquina física vai redirecionar para a porta 
    # 3000 do container da API.
    ports:
      - 3000:3000
    
    # depends_on - Define dependências de inicialização.
    # Com "condition: service_healthy", a API vai ESPERAR o PostgreSQL passar no teste de
    # healthcheck antes de tentar rodar. Isso evita que a API quebre tentando conectar em
    # um banco que ainda está subindo.
    depends_on:
      postgres: service_healthy
    
    # volume - Cria um volume para o compose.
    # Note que o volume aqui é diferente do volume declarado no Dockerfile, isso porque se
    # criarmos um volume no Dockerfile e usar o compose esse volume criado será um "fantasma"
    # e não vamos conseguir usá-lo, por isso, ao usar compose certifique-se de retirar o
    # volume declarado no Dokerfile e declare-o aqui.
    volumes:
      - ./logs:/app/logs

  aplicacao:
    build: ./aplicacao
    restart: on-failure
  
  postgres:
    # image - Baixa diretamente a imagem oficial do PostgreSQL do Docker Hub.
    image: postgres:17

    # restart - "always" garante que o banco SEMPRE reinicie em caso de queda ou se o servidor reiniciar.
    restart: always

    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=prisma
    
    ports:
      - "5432:5432"
    
    # healthcheck - Testa periodicamente se o banco de dados já está pronto para receber conexões.
    healthcheck:
      # Executa o utilitário nativo do Postgres "pg_isready" dentro do container
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
      interval: 5s # Roda o teste a cada 5 segundos
      timeout: 2s # Aguarda no máximo 2 segundos pela resposta
      retries: 20 # Tenta até 20 vezes antes de considerar o container "não saudável"
    
    # volumes - Mapeia o volume nomeado "postgres_data" (definido na raiz) para a pasta interna 
    # do Postgres onde os arquivos do banco de fato ficam salvos no Linux.
    volumes:
      - postgres_data:/var/lib/postgresql/data

# VOLUMES GLOBAIS
# Aqui declaramos os volumes gerenciados pelo Docker Compose.
# Isso garante que mesmo se você der `docker compose down`, seus dados do banco de dados 
# NÃO serão apagados do HD da sua máquina.
volumes:
  postgres_data:
```

Para subir todo esse ambiente com apenas um comando: `docker compose up -d`

Dica: Comandos úteis

```bash
# Subir tudo em segundo plano
docker compose up -d

# Ver logs de todos os containers em tempo real
docker compose logs -f

# Parar todos os containers sem apagar os dados do banco
docker compose down

# Parar tudo E APAGAR os volumes
docker compose down -v
```

## Kubernetes (k8s)

Aqui não nos aprofundaremos muito: Kubernetes é uma ferramenta de orquestração avançada e que você provavelmente não precisará no início da sua carreira, mas é fundamental saber que existe e qual problema ela resolve.

### Que problema o Kubernetes resolve?

O Docker Compose é ótimo para rodar containers numa única máquina ou numa VM simples na nuvem. Mas e se a sua aplicação crescer e você precisar de 100 instâncias da sua API distribuídas em 10 servidores diferentes, com balanceamento de carga automático, atualizações sem tirar o sistema do ar (zero downtime) e reinício automático se um servidor físico queimar no meio da madrugada?

O Docker Compose sozinho não gerencia múltiplos servidores. É aí que entra o Kubernetes (k8s): ele é o "maestro" responsável por gerenciar clusters (grupos de máquinas) rodando containers.

### Conceitos-chave do K8s

1. Node (Nó): É uma máquina individual (física ou virtual) que faz parte do cluster.
2. Pod: A menor unidade do Kubernetes. Pense nele como uma "cápsula" que envolve um ou mais containers que compartilham a mesma rede e armazenamento.
3. Deployment: Define o estado desejado da aplicação. Você diz ao Kubernetes: "Quero 5 réplicas da minha API rodando a imagem v1". Se um Pod cair, o Deployment cria outro automaticamente (self-healing).
4. Service: Como os Pods são criados e destruídos o tempo todo e têm IPs dinâmicos, o Service fornece um endereço IP/DNS fixo interno para rotear o tráfego até os Pods certos.
5. Ingress: É a porta de entrada HTTP/HTTPS do cluster para o mundo externo, fazendo o papel de roteador e proxy reverso (como o Nginx).

### Quando usar Kubernetes?

Para a imensa maioria dos projetos pequenos e médios: NÃO use.

Adotar Kubernetes sem necessidade traz uma complexidade gigante de infraestrutura (overengineering). Prefira usar Docker Compose, Docker Swarm, serviços gerenciados como AWS ECS ou plataformas PaaS (como Render, Fly.io e Railway) até que a sua demanda exija escalar dezenas de microsserviços em múltiplos nós.
