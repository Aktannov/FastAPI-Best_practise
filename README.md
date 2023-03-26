# FastAPI Best Practices

#### 1. Структура проекта. Последовательный и предсказуемый
Существует много способов структурировать проект, но лучшая структура — это последовательная, прямая и не имеющая сюрпризов структура.

* Если просмотр структуры проекта не дает вам представления о том, о чем идет речь, возможно, структура неясна.
* Если вам приходится открывать пакеты, чтобы понять, какие модули в них находятся, значит, ваша структура неясна.
* Если частота и расположение файлов кажутся случайными, то структура вашего проекта плохая.
* Если взгляд на расположение модуля и его название не дает вам представления о том, что у него внутри, то ваша структура очень плохая.
Хотя структура проекта, в которой мы разделяем файлы по их типу (например, API, crud, модели, схемы), представленная [@tiangolo](https://github.com/tiangolo) , хороша для микросервисов или проектов с меньшим количеством областей, мы не смогли вписать ее в наш монолит с большим количеством доменов. и модули. Структура, которую я нашел более масштабируемой и развиваемой, вдохновлен [​​Dispatch](https://github.com/Netflix/dispatch) от Netflix с некоторыми небольшими изменениями.

```
fastapi-project
├── alembic/
├── src
│   ├── auth
│   │   ├── router.py
│   │   ├── schemas.py  # pydantic models
│   │   ├── models.py  # db models
│   │   ├── dependencies.py
│   │   ├── config.py  # local configs
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── aws
│   │   ├── client.py  # client model for external service communication
│   │   ├── schemas.py
│   │   ├── config.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   └── posts
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── dependencies.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── config.py  # global configs
│   ├── models.py  # global models
│   ├── exceptions.py  # global exceptions
│   ├── pagination.py  # global module e.g. pagination
│   ├── database.py  # db connection related stuff
│   └── main.py
├── tests/
│   ├── auth
│   ├── aws
│   └── posts
├── templates/
│   └── index.html
├── requirements
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env
├── .gitignore
├── logging.ini
└── alembic.ini
```


1. Хранить все каталоги домена внутри srcпапки
 i. ```src/``` - самый высокий уровень приложения, содержит общие модели, конфиги, константы и т. д.
 ii. ```src/main.py``` - корень проекта, который запускает приложение FastAPI
2. Каждый пакет имеет свой роутер, схемы, модели и т.д.
 i. ```router.py``` - это ядро ​​каждого модуля со всеми конечными точками
 ii. ```schemas.py``` - для пидантических моделей
 iii. ```models.py``` - для моделей БД
 iv. ```service.py```- специфичная для модуля бизнес-логика
 v. ```dependencies.py``` - зависимости от роутера
 vi. ```constants.py``` - специфичные для модуля константы и коды ошибок
 vii. ```config.py``` - например env варс
 viii. ```utils.py``` - функции, не относящиеся к бизнес-логике, например, нормализация ответа, обогащение данных и т. д.
 ix. ```exceptions``` - специфические для модуля исключения, например PostNotFound,InvalidUserData
3. Когда пакету требуются службы, зависимости или константы из других пакетов — импортируйте их с явным именем модуля.

```python
from src.auth import constants as auth_constants
from src.notifications import service as notification_service
from src.posts.constants import ErrorCode as PostsErrorCode  # in case we have Standard ErrorCode in constants module of each package
```


#### 2. Чрезмерное использование Pydantic для проверки данных
Pydantic имеет богатый набор функций для проверки и преобразования данных.

В дополнение к обычным функциям, таким как обязательные и необязательные поля со значениями по умолчанию, Pydantic имеет встроенные комплексные инструменты обработки данных, такие как регулярные выражения, перечисления для ограниченных разрешенных параметров, проверка длины, проверка электронной почты и т. д.

```python
from enum import Enum
from pydantic import AnyUrl, BaseModel, EmailStr, Field, constr

class MusicBand(str, Enum):
   AEROSMITH = "AEROSMITH"
   QUEEN = "QUEEN"
   ACDC = "AC/DC"


class UserBase(BaseModel):
    first_name: str = Field(min_length=1, max_length=128)
    username: constr(regex="^[A-Za-z0-9-_]+$", to_lower=True, strip_whitespace=True)
    email: EmailStr
    age: int = Field(ge=18, default=None)  # must be greater or equal to 18
    favorite_band: MusicBand = None  # only "AEROSMITH", "QUEEN", "AC/DC" values are allowed to be inputted
    website: AnyUrl = None
```


#### 3. Используйте зависимости для проверки данных по сравнению с БД
Pydantic может только проверять значения из ввода клиента. Используйте зависимости для проверки данных на соответствие ограничениям базы данных, таким как электронная почта уже существует, пользователь не найден и т. д.

```python
# dependencies.py
async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


# router.py
@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post_by_id(post: Mapping = Depends(valid_post_id)):
    return post


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    update_data: PostUpdate,  
    post: Mapping = Depends(valid_post_id), 
):
    updated_post: Mapping = await service.update(id=post["id"], data=update_data)
    return updated_post


@router.get("/posts/{post_id}/reviews", response_model=list[ReviewsResponse])
async def get_post_reviews(post: Mapping = Depends(valid_post_id)):
    post_reviews: list[Mapping] = await reviews_service.get_by_post_id(post["id"])
    return post_reviews
```

Если бы мы не поставили проверку данных в зависимость, нам пришлось бы добавить проверку post_id для каждой конечной точки и написать одинаковые тесты для каждой из них.

#### 4. Документы
1. Если ваш API не является общедоступным, скройте документы по умолчанию. Показать его явно только на выбранных envs.

```python
from fastapi import FastAPI
from starlette.config import Config

config = Config(".env")  # parse .env file for env variables

ENVIRONMENT = config("ENVIRONMENT")  # get current env name
SHOW_DOCS_ENVIRONMENT = ("local", "staging")  # explicit list of allowed envs

app_configs = {"title": "My Cool API"}
if ENVIRONMENT not in SHOW_DOCS_ENVIRONMENT:
   app_configs["openapi_url"] = None  # set url for docs as null

app = FastAPI(**app_configs)
```

2. Помогите FastAPI создать понятную документацию
 i. Установить ```response_model```, ```status_code```, ```description```, и т.д.
 ii. Если модели и статусы различаются, используйте ```responses``` атрибут маршрута, чтобы добавить документы для разных ответов.

```python
from fastapi import APIRouter, status

router = APIRouter()

@router.post(
    "/endpoints",
    response_model=DefaultResponseModel,  # default response pydantic model 
    status_code=status.HTTP_201_CREATED,  # default status code
    description="Description of the well documented endpoint",
    tags=["Endpoint Category"],
    summary="Summary of the Endpoint",
    responses={
        status.HTTP_200_OK: {
            "model": OkResponse, # custom pydantic model for 200 response
            "description": "Ok Response",
        },
        status.HTTP_201_CREATED: {
            "model": CreatedResponse,  # custom pydantic model for 201 response
            "description": "Creates something from user request ",
        },
        status.HTTP_202_ACCEPTED: {
            "model": AcceptedResponse,  # custom pydantic model for 202 response
            "description": "Accepts request and handles it later",
        },
    },
)
async def documented_route():
    pass
```

Будет генерировать такие документы:
![respone](custom_responses.png)


#### 5. Печатать важно 
FastAPI, Pydantic и современные IDE поощряют использование подсказок типов.

__Без подсказок типа__
![hit1](type_hintsless.png)

__С подсказками типа__
![hit2](type_hints.png)

#### 6. Сначала SQL, потом Pydantic
* Обычно база данных обрабатывает данные намного быстрее и чище, чем CPython.
* Предпочтительно выполнять все сложные соединения и простые операции с данными с помощью SQL.
* Предпочтительно агрегировать JSON в БД для ответов с вложенными объектами.

```python
# src.posts.service
from typing import Mapping

from pydantic import UUID4
from sqlalchemy import desc, func, select, text
from sqlalchemy.sql.functions import coalesce

from src.database import databse, posts, profiles, post_review, products

async def get_posts(
    creator_id: UUID4, *, limit: int = 10, offset: int = 0
) -> list[Mapping]: 
    select_query = (
        select(
            (
                posts.c.id,
                posts.c.type,
                posts.c.slug,
                posts.c.title,
                func.json_build_object(
                   text("'id', profiles.id"),
                   text("'first_name', profiles.first_name"),
                   text("'last_name', profiles.last_name"),
                   text("'username', profiles.username"),
                ).label("creator"),
            )
        )
        .select_from(posts.join(profiles, posts.c.owner_id == profiles.c.id))
        .where(posts.c.owner_id == creator_id)
        .limit(limit)
        .offset(offset)
        .group_by(
            posts.c.id,
            posts.c.type,
            posts.c.slug,
            posts.c.title,
            profiles.c.id,
            profiles.c.first_name,
            profiles.c.last_name,
            profiles.c.username,
            profiles.c.avatar,
        )
        .order_by(
            desc(coalesce(posts.c.updated_at, posts.c.published_at, posts.c.created_at))
        )
    )
    
    return await database.fetch_all(select_query)

# src.posts.schemas
import orjson
from enum import Enum

from pydantic import BaseModel, UUID4, validator


class PostType(str, Enum):
    ARTICLE = "ARTICLE"
    COURSE = "COURSE"

   
class Creator(BaseModel):
    id: UUID4
    first_name: str
    last_name: str
    username: str


class Post(BaseModel):
    id: UUID4
    type: PostType
    slug: str
    title: str
    creator: Creator

    @validator("creator", pre=True)  # before default validation
    def parse_json(cls, creator: str | dict | Creator) -> dict | Creator:
       if isinstance(creator, str):  # i.e. json
          return orjson.loads(creator)

       return creator
    
# src.posts.router
from fastapi import APIRouter, Depends

router = APIRouter()


@router.get("/creators/{creator_id}/posts", response_model=list[Post])
async def get_creator_posts(creator: Mapping = Depends(valid_creator_id)):
   posts = await service.get_posts(creator["id"])

   return posts
```

Если БД формы агрегированных данных представляет собой простой JSON, взгляните на Jsonтип поля Pydantic, который сначала загрузит необработанный JSON.

```python
from pydantic import BaseModel, Json

class A(BaseModel):
    numbers: Json[list[int]]
    dicts: Json[dict[str, int]]

valid_a = A(numbers="[1, 2, 3]", dicts='{"key": 1000}')  # becomes A(numbers=[1,2,3], dicts={"key": 1000})
invalid_a = A(numbers='["a", "b", "c"]', dicts='{"key": "str instead of int"}')  # raises ValueError
```