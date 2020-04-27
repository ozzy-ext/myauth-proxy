# MyAuth.Proxy

[![Docker image](https://img.shields.io/docker/v/ozzyext/myauth-proxy?sort=semver)](https://hub.docker.com/r/ozzyext/myauth-proxy)

Ознакомьтесь с последними изменениями в [журнале изменений](/changelog.md).

## Обзор

Серверное приложение, осуществляющее контроль доступа к HTTP-ресурсам. 

Позволяет осуществлять контроль:

* анонимного доступа;
* доступа с `Basic` аутентификацией;
* доступа по ключам с `JWT` токенами.

После успешной авторизации запрос к ресурсу снабжается заголовком авторизации с открытой схемой аутентификации [MyAuth](https://github.com/ozzy-ext-myauth/specification), указанной в конфигурации. А так же, другими заголовками, если они предусмотрены выбранной схемой аутентификации.

На схеме ниже отражена концепция работы сервиса.

![](./doc/my-auth-proxy.png)

## Развёртывание

В данном разделе описывается развёртывание с использованием `docker`-контейнеров. Образы сервиса зарегистрированы в реестре образов [docker-hub](https://hub.docker.com/r/ozzyext/myauth-proxy).

Для настройки работы сервиса необходимо определить следующие параметры:

* настройки авторизации: файл `/app/configs/auth-config.lua` в контейнере;
* настройки `nginx` локации по умолчанию: `/etc/nginx/snippets/default-location.conf`
* адрес целевого сервера `TARGET_SERVER`, куда будут перенаправляться авторизированные запросы.

Опционально можно настроить:

* подпись сервиса в случае отрицательного ответа:
  * помещается в заголовок `X-Source` ответа;
  * `myauth-proxy` - по умолчанию
  * устанавливается через переменную окружения `SERVICE_SIGN`

Пример развёртывания сервиса:

```bash
docker run --rm \ 
	-p 80:80 \
	-v ./auth-config.lua:/app/configs/auth-config.lua \
	-v ./default-location.conf:/etc/nginx/snippets/default-location.conf \
	-e TARGET_SERVER=target-host.com \
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

### Файл

Файл, содержащий настройки доступа к перечисленным `URL` в формате `LUA`. Содержание файла - присвоение переменным с фиксированными именами значений заданной структуры с данными об авторизации перечисленных `URL`.

Пример конфигурации:

```lua
debug_mode=true

dont_apply_for = {
	"/free_for_access"
}

only_apply_for = {
	"/"
}

black_list = {
	"/blocked"
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
	ignore_audience = false,
	rules = {
		{
			url = "/rbac-access-[%d]+",
			allow = { "role-1", "role-2" },
			deny = { "role-3", "role-4" },
			allow_get = { "role-5"  },
			deny_post = { "role-1"  }
		},
		{
			url = "/rbac-access-2",
 			allow_for_all = true
		}
	}
}
```

* `debug_mode` - `true`/`false`, устанавливает режим отладки. Подробнее о режиме отладки - ниже, в разделе [Отладка](##Отладка).
* `output_scheme` - исходящая схема аутентификации, используемая для передачи утверждений пользователя при доступе к ресурсу. Доступные значения:
  * [MyAuth1](https://github.com/ozzy-ext-myauth/specification/blob/master/v1/myauth-authentication-1.md) (по умолчанию)
* `dont_apply_for` - массив [шаблонов](https://www.lua.org/pil/20.2.html) `URL`, к которым не будет применяться авторизация;
* `only_apply_for` - массив [шаблонов](https://www.lua.org/pil/20.2.html) `URL`, к которым будет применяться авторизация. По умолчанию - ко всем; 
* `black_list` -  массив [шаблонов](https://www.lua.org/pil/20.2.html) `URL`, к которым будет запрещён доступ;
* `anon` - массив [шаблонов](https://www.lua.org/pil/20.2.html)`URL`, к которым будет предоставлен анонимный доступ;
* `basic` - массив пользователей, для которых доступна `basic`-авторизация и параметры их доступа:
  * `id` - идентификатор пользователя, используемый при формировании значения заголовка `Authentication`;
  * `pass` - пароль пользователя в открытом виде, используемый при формировании значения заголовка `Authentication`;
  * `urls` - массив [шаблонов](https://www.lua.org/pil/20.2.html) `URL` целевых ресурсов;
* `rbac` - настройки доступа на основе ролей при аутентификации с токеном доступа:
  * `ignore_audience` - флаг, отключающий проверку `audience` токена; `false` - по умолчанию;
  * `rules` - массив правил
    * `url` - [шаблон](https://www.lua.org/pil/20.2.html) `URL` целевого ресурса;
    * `allow_for_all` - `true`/`false`, устанавливает разрешение доступа для любой роли; По сути - доступ для любого авторизованного пользователя;
    * `allow` - массив ролей пользователей, которым разрешён доступ к целевому ресурсу без учёта `http`-метода;
    * `deny` - массив ролей пользователей, которым запрещён доступ к целевому ресурсу без учёта `http`-метода;
    * `allow_get` - массив ролей пользователей, которым разрешён доступ к целевому ресурсу с `http`-методом `GET`. Аналогично, для любого `http`-метода;
    * `deny_get` - массив ролей пользователей, которым запрещён доступ к целевому ресурсу с `http`-методом `GET`. Аналогично, для любого `http`-метода;

### Секреты

Файл с секретами должен располагаться на хост-сервере и подключаться в контейнер по пути `/app/configs/auth-secrets.lua`

Формат файла - `LUA` скрипт с объявлением переменных.

Пример:

```lua
jwt_secret = "some-secret"
```

* `jwt_secret` - ключ для проверки подписи `JWT` токена по алгоритму `HS256`.

### Шаблоны 

Шаблоны `URL` - не регулярные выражения, а имеют специальный LUA-формат для шаблонов. [Оф документация](https://www.lua.org/pil/20.2.html). 

Важной особенностью шаблонов конфигурации является возможность не экранировать дефис (`-`). Т.е. при написании шаблона для `/some-resource` пришлось бы писать `/some%-resource`, но именно в этом конфигурационном файле этого можно не делать. Это сделано для удобства, потому что дефис (`-`) часто встречается в `URL`.

### Порядок проверки доступа

Общий алгоритм:

* если найдено совпадение в `black_list` - **отказ**;
* если найдено совпадение в `dont_apply_for` - **разрешение**;
* если объявлен список `only_apply_for` и в нём нет совпадений - **отказ**;
* если нет заголовка `Authorization`, то используются правила `anon`:
  * если найдено совпадение в `anon` - **разрешение**;
  * иначе - **отказ**;
* если авторизация `Basic`, то используются правила `basic`:
  * если нет совпадений по пользователю - **отказ**;
  * если есть совпадение по имени пользователя, то используются настройки этого пользователя;
    * если найдено совпадение в списке `urls` - **разрешение**;
    * иначе - **отказ**;
* если авторизация `Bearer`, то используются правила `rbac`:
  * если нет совпадений в `roles` по полю `url` - **отказ**;
  * если есть совпадения, то происходит вычисление разрешения по всем совпадениям по следующим правилам:
    * в каждом совпадении проверяется `allow_for_all`. Если хотя бы в одном этот параметр установлен в `true`, то есть разрешающий фактор;
    * в каждом совпадении ищутся роли пользователя в `allow` и `allow_xxx` (в соответствии с `http`-методом). Если найдены, то есть разрешающий фактор;
    * в каждом совпадении ищутся роли пользователя в `deny` и `deny_xxx` (в соответствии с `http`-методом). Если найдены, то есть запрещающий фактор;
    * если есть хотя бы один запрещающий фактор, то - **отказ**;
    * если есть хотя бы один разрешающий фактор, то - **разрешение**;
    * в противном случае (нет ни разрешений, ни запрещений) - **отказ**;
* в противном случае (другой тип авторизационных данных в заголовке `Authorization`) - **отказ**.

## Отладка 

Режим отладки позволяет получить более подробную информацию о процессе авторизации внутри прокси из текущего запроса. В режиме отладки:

* содержательная часть отрицательных ответов содержат раскрытую причину отказа;
* ответ содержит отладочные заголовки с дополнительной информацией.

Для включения режима отладки укажите в конфигурации 

```lua
debug_mode=true
```

### Отладочные заголовки

#### X-Debug-Authorization-Header

Содержит значения заголовка `Authorization` запроса, перенаправленного к запрашиваемому ресурсу.

#### X-Debug-Rbac

Передаёт `json`-объект, содержащий данные о проверке авторизации на базе ролей.

Пример содержания заголовка:

```json
{"total_factor":true,"allow_for_all":true,"pattern":"/bearer-access-[%d]+"}
```

#### X-Debug-Claim-...

Добавляются только при исходящей аутентификации `MyAuth2`. Они содержат значения соответствующих заголовков `X-Claim-...` в перенаправленном запросе.

## Особенности реализации схем аутентификации

### MyAuth v1

#### Basic

При `Basic` аутентификации, в результирующий заголовок авторизации попадает только идентификатор пользователя под именем `sub`(subject, по аналогии с тем, как этот параметр называется в `jwt`). Пример заголовка авторизации для пользователя `foo-user`:

```
Authorization: MyAuth1 sub="foo-user"
```

#### Bearer

При `Bearer` аутентификации с `jwt` токеном, все утверждения из токена попадают в заголовок авторизации со своими именами.

### MyAuth v2

#### Basic

При `Basic` аутентификации, кроме заголовка авторизации, добавляется только заголовок с идентификатором пользователя `X-Claim-User-Id`. Пример заголовка авторизации для пользователя `foo-user`:

```
Authorization: MyAuth2
X-Claim-User-Id: foo-user
```

#### Bearer

##### Имена заголовков

При `Bearer` аутентификации с `jwt` токеном, кроме заголовка авторизации, все утверждения из токена добавляются в перенаправленный запрос в виде заголовков с префиксом `X-Claim-<нормализованный_тип_утверждения>`. 

Нормализация типа утверждения:

* заменяет разделитель `:` на `-`;
* первые буквы разделённых слов переводи в верхний регистр.

Пример утверждений:

```
MyClaim1 = val1
myClaim2 = val2
my-claim-3 = val3 
my:claim:4 = val4
```

Заголовки перенаправленного запроса по этим утверждениям:

```
Authorization: MyAuth2
X-Claim-MyClaim1: val1
X-Claim-MyClaim2: val2
X-Claim-My-Claim-3: val3
X-Claim-My-Claim-4: val4
```

##### Зарезервированные заголовки

Для следующих утверждений используются фиксированные имена заголовков:

* `sub` -> `X-Claim-User-Id`;
* `roles` -> `X-Claim-Roles`;
* `role` -> `X-Claim-Role`;

##### Утверждения-массивы

Утверждения-массивы из `jwt` токена сериализуются в строку, где значения массива разделены запятыми. 

Пример: 

```
Authorization: MyAuth2
X-Claim-Roles: admin,user
```

