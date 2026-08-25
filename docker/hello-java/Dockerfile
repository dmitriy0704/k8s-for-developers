# Присваиваем имя сборочному контейнеру, который отвечает за сборку кода
FROM eclipse-temurin:21-jdk as buildstage
COPY . /app
WORKDIR /app
RUN javac Hello.java

# В качестве основы для выполняемого контейнера
# используем сокращенную версию базового образа,
# которая не содержит инструментов компиляции
FROM eclipse-temurin:21-jre-jammy

# С помощью выражения --from= ссылаемся
# на файлы из сборочного контейнера
COPY --from=buildstage /app/Hello.class /app/
WORKDIR /app
CMD java Hello
