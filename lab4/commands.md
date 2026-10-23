# Лабораторна робота 4. Робота з Docker Volumes
Студентка: Криловська Варвара
Група: 6.1214-пі-1
Варіант: 11

## 1. Підготовка
mkdir -p html
docker info

## 2. Bind mounts (Nginx та Apache)
docker run -d --name "lab4-krylovska-nginx" -p 8080:80 -v "$PWD/html:/usr/share/nginx/html:ro" nginx:1.28
docker run -d --name "lab4-krylovska-apache" -p 8081:80 -v "$PWD/html:/usr/local/apache2/htdocs:ro" httpd:2.4-alpine
curl -fsS http://localhost:8080
curl -fsS http://localhost:8081
docker inspect "lab4-krylovska-nginx" --format '{{json .Mounts}}'
docker inspect "lab4-krylovska-apache" --format '{{json .Mounts}}'

docker stop "lab4-krylovska-nginx" "lab4-krylovska-apache"
docker rm "lab4-krylovska-nginx" "lab4-krylovska-apache"

docker run -d --name "lab4-krylovska-nginx" -p 8080:80 --mount "type=bind,src=$PWD/html,dst=/usr/share/nginx/html,readonly" nginx:1.28
docker run -d --name "lab4-krylovska-apache" -p 8081:80 --mount "type=bind,src=$PWD/html,dst=/usr/local/apache2/htdocs,readonly" httpd:2.4-alpine
docker stop "lab4-krylovska-nginx" "lab4-krylovska-apache"
docker rm "lab4-krylovska-nginx" "lab4-krylovska-apache"

## 3. Дані MySQL та PostgreSQL
docker volume create "lab4-krylovska-mysql-data"
docker volume create "lab4-krylovska-postgres-data"

docker run -d --name "lab4-krylovska-mysql" --env-file mysql.env -v "lab4-krylovska-mysql-data:/var/lib/mysql" mysql:8.4
docker run -d --name "lab4-krylovska-postgres" --env-file postgres.env -v "lab4-krylovska-postgres-data:/var/lib/postgresql/data" postgres:17-alpine

docker exec -it "lab4-krylovska-mysql" mysql -uroot -p lab4
docker exec -it "lab4-krylovska-postgres" psql -U postgres -d lab4

docker inspect "lab4-krylovska-mysql" "lab4-krylovska-postgres" --format '{{.Name}} {{.Id}}'
docker volume inspect "lab4-krylovska-mysql-data" "lab4-krylovska-postgres-data"

## 4. Заміна контейнерів
docker stop "lab4-krylovska-mysql" "lab4-krylovska-postgres"
docker rm "lab4-krylovska-mysql" "lab4-krylovska-postgres"

docker run -d --name "lab4-krylovska-mysql" --env-file mysql.env --mount "type=volume,src=lab4-krylovska-mysql-data,dst=/var/lib/mysql" mysql:8.4
docker run -d --name "lab4-krylovska-postgres" --env-file postgres.env --mount "type=volume,src=lab4-krylovska-postgres-data,dst=/var/lib/postgresql/data" postgres:17-alpine

docker inspect "lab4-krylovska-mysql" "lab4-krylovska-postgres" --format '{{.Name}} {{.Id}}'
docker exec -it "lab4-krylovska-mysql" mysql -uroot -p -e "SELECT * FROM student_record;" lab4
docker exec -it "lab4-krylovska-postgres" psql -U postgres -d lab4 -c "SELECT * FROM student_record;"

## 5. VOLUME у власному образі
docker build -f Dockerfile.volume -t "rosei1/alpine-lighttpd:2.0" .

docker run -d --name "lab4-krylovska-auto" -p 8082:80 "rosei1/alpine-lighttpd:2.0"
docker inspect "lab4-krylovska-auto" --format '{{json .Mounts}}'
curl -fsS http://localhost:8082
docker stop "lab4-krylovska-auto"
docker rm -v "lab4-krylovska-auto"

docker volume create "lab4-krylovska-content"
docker run -d --name "lab4-krylovska-own" -p 8083:80 -v "lab4-krylovska-content:/var/www/localhost/htdocs" "rosei1/alpine-lighttpd:2.0"
docker inspect "lab4-krylovska-own" --format '{{json .Mounts}}'
curl -fsS http://localhost:8083

docker run -d --name "lab4-krylovska-bind" -p 8082:80 -v "$PWD/html:/var/www/localhost/htdocs:ro" "rosei1/alpine-lighttpd:2.0"
curl -fsS http://localhost:8082

docker push "rosei1/alpine-lighttpd:2.0"
docker pull "rosei1/alpine-lighttpd:2.0"
docker image inspect "rosei1/alpine-lighttpd:2.0" --format '{{json .RepoDigests}}'

## 6. Тимчасові файли tmpfs
docker run -d --name "lab4-krylovska-memory" --tmpfs /scratch:size=16m python:3.12-slim sleep infinity
docker exec "lab4-krylovska-memory" sh -c 'echo temporary > /scratch/check.txt'
docker exec "lab4-krylovska-memory" cat /scratch/check.txt
docker inspect "lab4-krylovska-memory" --format '{{.Id}}'
docker stop "lab4-krylovska-memory"
docker start "lab4-krylovska-memory"
docker inspect "lab4-krylovska-memory" --format '{{.Id}}'
docker exec "lab4-krylovska-memory" sh -c 'test ! -e /scratch/check.txt && echo "file absent"'

## 7. Резервне копіювання та відновлення
docker stop "lab4-krylovska-own"
mkdir -p backup
docker run --rm --mount "type=volume,src=lab4-krylovska-content,dst=/data,readonly" --mount "type=bind,src=$PWD/backup,dst=/backup" python:3.12-slim tar -C /data -cf /backup/html.tar .
docker volume create "lab4-krylovska-restored"
docker run --rm --mount "type=volume,src=lab4-krylovska-restored,dst=/data" --mount "type=bind,src=$PWD/backup,dst=/backup,readonly" python:3.12-slim tar -C /data -xf /backup/html.tar
MSYS_NO_PATHCONV=1 docker run -d --name "lab4-krylovska-restored-web" -p 8084:80 --mount "type=volume,src=lab4-krylovska-restored,dst=/var/www/localhost/htdocs" "rosei1/alpine-lighttpd:2.0"
curl -fsS http://localhost:8084

## 8. Безпечне завершення
docker ps -a --filter "name=lab4-krylovska-"
docker stop "lab4-krylovska-mysql" "lab4-krylovska-postgres" "lab4-krylovska-bind" "lab4-krylovska-memory" "lab4-krylovska-restored-web"
docker rm "lab4-krylovska-mysql" "lab4-krylovska-postgres" "lab4-krylovska-bind" "lab4-krylovska-memory" "lab4-krylovska-restored-web" "lab4-krylovska-own"
docker volume rm "lab4-krylovska-mysql-data" "lab4-krylovska-postgres-data" "lab4-krylovska-content" "lab4-krylovska-restored"
