# Baixando e iniciando

Um dos caminhos mais simples de inicializar com Spring Boot é acessando o site do configurador

https://start.spring.io/

1. Selecione Maven em Project
2. Selecione Java em Language
3. Selecione a versão desejada
4. Em metadata preencha com os dados do seu projeto
5. Em configuration selecione Properties
6. Em java selecione a versão 21 ou posterior
7. Em dependências selecione: Spring BootDevTools, Spring Web, Postgres Driver, Spring Data JPA e qualquer outro que tu queira
8. Clique em Generate

Pronto, agora você baixa o arquivo para sua máquina

9. Faremos um dockerfile que chamaremos de Dockerfile.dev com o seguinte conteúdo
> lembre que o arquivo deve ficar na raiz do projeto

```
FROM eclipse-temurin:21-jdk

RUN apt-get update && apt-get -y upgrade

RUN apt-get install -y inotify-tools dos2unix
ENV HOME=/app
RUN mkdir -p $HOME
WORKDIR $HOME

CMD [ "./run.sh" ]
```

> este Dockerfile permite que ao alterar um código na IDE, o Java atualize automaticamente o build

10. Para funcionar certinho nossa estratégia temos que criar o arquivo run.sh na raiz da aplicação, que será utilizado pelo docker para notificar as alterações na IDE

```
dos2unix mvnw
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005" &
while true; do
  inotifywait -e modify,create,delete,move -r ./src/ && ./mvnw compile
done

```

11. Faremos agora um dockerfile que será nosso builder de produção

```
FROM maven:3.9.8-eclipse-temurin-21-jammy AS builder

WORKDIR /app

COPY . /app

RUN mvn package -DskipTests


FROM eclipse-temurin:21-jdk AS runner

ENV TZ=America/Sao_Paulo
ENV JAVA_TOOL_OPTIONS="-Duser.timezone=America/Sao_Paulo"

COPY --from=builder /app/target/raiz-api.jar /raiz-api.jar

ENTRYPOINT ["java","-Dhttps.protocols=TLSv1.1,TLSv1.2","-Djavax.net.debug=ssl", "-jar","/raiz-api.jar"]

EXPOSE 80

```

> Compare os dois dockerfiles e veja o que tem de diferente neles

12. Criaremos agora nosso compose file, na raiz do projeto

```
services:  
  pgsql:
    image: postgres:15    
    env_file:
      - .env      
    ports:
      - "54329:5432"
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_BASE}   
    volumes:
      - ./.docker/init.sql:/docker-entrypoint-initdb.d/init.sql
  web:
    build:
      context: .
      dockerfile: ./.docker/Dockerfile.dev
    working_dir: /app
    volumes:
      - ./:/app
    depends_on:
      - pgsql      
    ports:    
      - "8080:8080"
      - "35729:35729"
      - "5005:5005"
    env_file:
      - .env    

```

13. Veja que temos um arquivo .env, esse arquivo nós versionamos com .env.example com dados de exemplo, nunca com os reais. O arquivo original será .env, você normalmente renomeia esse arquivo example e muda as variáveis

```
DB_BASE=devops
DB_HOST=pgsql
DB_PASSWORD=devops
DB_USER=devops
SERVICE_URL=http://localhost:8080
PORT=8080
```

# agora vamos brincar

1. Gerando os build

```
docker compose --profile dev build
```

2. Subindo tudo

```
docker comopse --profile dev up
```