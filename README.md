# MyAuth.Proxy

[![Docker image](https://img.shields.io/docker/v/ozzyext/myauth-proxy?sort=semver)](https://hub.docker.com/r/ozzyext/myauth-proxy)

## Обзор

Сервеное приложение, осуществляющее контроль доступа к HTTP-ресурсам. 

Позваляет осуществлять контроль:

* анонимного доступа;
* доступа с `Basic` аутентификацией;
* доступа по ключам с `JWT` токенами.

После успешной авторизации запрос к ресурсу подвергается следующим изменениям:

* удаляется заголовок `Authentication`;
* если авторизвция неанонимная, то добавляется заголовок `X-User-Id` с идентификатором пользователя;
* если авторизация с ключом доступа, то добавляется заголовок `X-User-Claims` содержащий `JSON` с утверждениями.

На схеме ниже отражена концепция работы сервиса.

![](./doc/my-auth-proxy.png)

## Развёртывание

В данном разделе описывается развёртывание с использованием `docker`-контейнеров. Образы сервиса зарегистрированы в реестре образов [docker-hub](https://hub.docker.com/r/ozzyext/myauth-proxy).

Для настройки работы сервиса необходимо определить следующие параметры:

* настройки авторизации: файл `/app/configs/auth-config.lua` в контейнере;
* настройки `nginx` локации по умолчанию: `/etc/nginx/snippets/default-location.conf`
* адрес целевого сервера `target-server`, куда будут перенаправляться авторизированные запросы.

Пример развёртывания сервиса:

```bash
docker run --rm \ 
	-p 80:80 \
	-v ./auth-config.lua:/app/configs/auth-config.lua \
	-v ./default-location.conf:/etc/nginx/snippets/default-location.conf \
	--add-host target-server:192.168.0.222 \
	ozzyext/myauth-proxy:latest
```

## Конфигурация локации по умолчанию

Файл, содержащий инструкции в формате `nginx` и загружается из пути `/etc/nginx/snippets/default-location.conf` контейнера. Встраивается в конфигурацию корневой локации сервера по умолчанию во внутреннем `nginx`:

 ```nginx
server {
	listen 80;
	server_name default_server;

	location / {

		proxy_pass http://target-server;

		# authorization here

		include snippets/default-location.conf # <-- HERE IS !!!
	}
}
 ```

## Настройки авторизации

Файл, содержащий настройи доступа к перечисленным `URL` в формате `LUA`. Содержание файла - присвоение переменным с фиксированными именами значений заданной структуры с данными об авторизации перечисленных `URL`.

Пример конфигурации:

```lua
black_list = {
    "/blocked"
}

white_list = {
    "/free-resource"
}

anon = {
	"/pub"
}

basic = {
	{
		id = "user-1",
		pass = "user-1-pass",
		urls = {
			"/basic-access-[%d]+",
			"/basic-access-a"
		}
	},
	{
		id = "user-2",
		pass = "user-2-pass",
		urls = {
			"/basic-access-[%d]+"			
		}
	},
	{
		id = "user-2",
		pass = "user-2-pass",
		urls = {
			"/basic-access-2"			
		}
	}
}

rbac = {
	secret = "qwerty",
    ignore_audience = false,
	rules = {
		{
			url = "/rbac-access-[%d]+",
			roles = { "role-1", "role-2" } 
		},
		{
			url = "/rbac-access-2",
			roles = { "role-3", "role-4" } 
		}
	}
}
```

* `black_list` -  массив шаблонов `URL`, к которым будет запрещён доступ;
* `white_list` -  массив шаблонов `URL`, к которым будет предоставлен доступ без авторизации;
* `anon` - массив шаблонов `URL`, к которым будет предоставлен анонимный доступ;
* `basic` - массив пользователей, для которых доступна `basic`-авторизация и параметры их доступа:
  * `id` - идентификатор пользователя, используемый при формировании значения заголовка `Authentication`;
  * `pass` - пароль пользователя в открытом виде, используемый при формировании значения заголовка `Authentication`;
  * `urls` - массив [шаблонов](https://www.lua.org/pil/20.2.html) `URL` целевых ресурсов;
* `rbac` - настройки доступа на основе ролей при аутенитфикации с токеном доступа:
  * `secret` - ключ для проверки подписи `JWT` токена по алгоритму `HS256`;
  * `ignore_audience` - флаг, отключающий проверку `audience` токена; `false` - по умолчанию;
  * `rules` - массив правил
    * `url` - [шаблон](https://www.lua.org/pil/20.2.html) `URL` целевого ресурса;
    * `roles` - массив ролей пользователей, которым будет предоставлен доступ к целевому ресурсу.

Шаблоны `URL` - не регулярные выражения, а имеют специальный LUA-формат для шаблонов. [Оф документация](https://www.lua.org/pil/20.2.html). 

Важной особенностью шаблонов конфигурации является возможность не экранировать дефис (`-`). Т.е. при написании шаблона для `/some-resource` пришлось бы писать `/some%-resource`, но именно в этом конфигурационном файле этого можно не делать. Это сделано для удобства, потому что дефис (`-`) часто встречается в `URL`.
