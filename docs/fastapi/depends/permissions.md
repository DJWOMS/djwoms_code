# Как реализовать permissions в FastAPI и передать параметры в Depends

## Идея
Реализовать проверку прав для каждого url. Данные о правах хранятся в БД. 

## Задача
Используя Depends FastAPI реализовать permissions, с возможностью передать список прав которые нужно 
проверить. В качестве orm использовать SqlAlchemy.

## Решение
У каждого пользователя может быть несколько прав. К примеру на чтение и редактирование.

!!! note warning
    Код всего проекта не буду приводить, только необходимых частей для решения задачи.

### Модели
Опишем модели `Permission` и `User`, а также таблицу для связи `m2m`.

```py linenums="1" title="myapp/models.py"
from sqlalchemy import Column, String, Integer, Boolean, Table, ForeignKey
from sqlalchemy.orm import relationship, backref

from myapp.db.session import Base


class Permission(Base):
    __tablename__ = "permissions"

    id = Column(Integer, primary_key=True, index=True, unique=True)
    name = Column(String)
    code_name = Column(Integer)

    
users_permissions = Table(
    'users_permissions',
    Base.metadata,
    Column('id', Integer, primary_key=True, unique=True),
    Column('user_id', Integer, ForeignKey('users.id')),
    Column('permission_id', Integer, ForeignKey('permissions.id'))
)


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True, unique=True)
    username = Column(String, unique=True)
    email = Column(String, unique=True)
    password = Column(String)
    is_active = Column(Boolean, default=False)
    is_superuser = Column(Boolean, default=False)
    permissions = relationship(
        "Permission",
        secondary=users_permissions,
        backref=backref('users_permissions', lazy="dynamic")
    )
```
У каждого permission есть имя и код. По коду будем проверять права юзера.

Связь между `User` и `Permission` реализована с помощью таблицы `users_permissions`.

### Pydantic model
```py linenums="1" title="myapp/schemas.py"
from pydantic import BaseModel


class User(BaseModel):
    id: int
    username: str
    email: str
    

class TokenPayload(BaseModel):
    user_id: int
```
Тут ничего сложного. Наши pydantic модели для юзера и токена.


### Depends permission
```py linenums="1" hl_lines="15-26" title="myapp/permissions.py"
import jwt
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, Security
from starlette.status import HTTP_403_FORBIDDEN

from myapp.user import models, crud, schemas


SECRET_KEY = "udfsdu6%$&(*Y9dHG(&ytdf987gFST*Sg897"
ALGORITHM = "HS256"
reusable_oauth2 = OAuth2PasswordBearer(tokenUrl="/api/v1/login/access-token")


def get_current_user(token: str = Security(reusable_oauth2)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        token_data = schemas.TokenPayload(**payload)
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=HTTP_403_FORBIDDEN, detail="Could not validate credentials"
        )

    if user := crud.user.get(id=token_data.user_id):
        raise HTTPException(status_code=404, detail="User not found")
    return user


class PermissionsRouter:
    def init(self, permissions: tuple):
        self.permissions = permissions

    def check_access(self, current_user: models.User):
        for permission in current_user.permissions:
            if permission.code_name in self.permissions:
                return current_user
        
        raise HTTPException(status_code=400, detail="The user doesn't have enough privileges")
    
    def call(self, user: models.User = Depends(get_current_user)):
        return self.check_access(current_user=user)
```
Для начала проверяем токен.

На 24-й строке проверяем есть ли пользователь с таким `id` и если есть, то получаем его.
**Пример кода crud не привожу, так как у вас он будет свой.**

Таким образом проверяем токен пользователя и получаем объект текущего пользователя.

```py linenums="1" hl_lines="29-41" title="myapp/permissions.py"
import jwt
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, Security
from starlette.status import HTTP_403_FORBIDDEN

from myapp.user import models, crud, schemas


SECRET_KEY = "udfsdu6%$&(*Y9dHG(&ytdf987gFST*Sg897"
ALGORITHM = "HS256"
reusable_oauth2 = OAuth2PasswordBearer(tokenUrl="/api/v1/login/access-token")


def get_current_user(token: str = Security(reusable_oauth2)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        token_data = schemas.TokenPayload(**payload)
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=HTTP_403_FORBIDDEN, detail="Could not validate credentials"
        )

    if user := crud.user.get(id=token_data.user_id):
        raise HTTPException(status_code=404, detail="User not found")
    return user


class PermissionsRouter:
    def init(self, permissions: tuple):
        self.permissions = permissions

    def check_access(self, current_user: models.User):
        for permission in current_user.permissions:
            if permission.code_name in self.permissions:
                return current_user
        
        raise HTTPException(status_code=400, detail="The user doesn't have enough privileges")
    
    def call(self, user: models.User = Depends(get_current_user)):
        return self.check_access(current_user=user)
```
Затем создаем класс зависимости который будет вызываться как функция. В него будем передавать список
(кортеж) `permissions` которые разрешены.

В методе `call` с помощью `Depends` получаем текущего пользователя, что позволит проверить его авторизацию.
Затем вызываем метод `check_access` для проверки прав.

В методе `check_access()` проверяем, чтобы хотя бы один `permission` пользователя входил 
в переданный список разрешенных `permissions`. И если есть совпадения, то вернем текущего юзера.

!!! note success
    В методе `check_access()` может (должна) быть ваша логика проверки прав.


## Endpoints

```py linenums="1" hl_lines="12 19" title="myapp/api.py"
from typing import List
from fastapi import APIRouter, Depends

from myapp.user import models, schemas, crud, permissions


user_router = APIRouter()


@user_router.get('/me', response_model=schemas.UserMe)
def user_me(
        current_user: models.User = Depends(permissions.PermissionsRouter((0,)))
):
    return current_user


@user_router.get('/', response_model=List[schemas.UserMe])
def get_all_users(
        current_user: models.User = Depends(permissions.PermissionsRouter((0, 1))), 
        skip: int = 0, 
        limit: int = 100
):
    return crud.user.get_multi(skip=skip, limit=limit)
```
Здесь мы используем `Depends` в который передаем класс `PermissionsRouter` с нужными параметрами.
**Так в зависимости мы можем передать параметры.**

### Альтернативный способ

```py linenums="1" hl_lines="9 10 15 22" title="myapp/api.py"
from typing import List
from fastapi import APIRouter, Depends

from myapp.user import models, schemas, crud, permissions


user_router = APIRouter()

permission_me = permissions.PermissionsRouter((0,))
permission_users = permissions.PermissionsRouter((0, 1))


@user_router.get('/me', response_model=schemas.UserMe)
def user_me(
        current_user: models.User = Depends(permission_me)
):
    return current_user


@user_router.get('/', response_model=List[schemas.UserMe])
def get_all_users(
        current_user: models.User = Depends(permission_users), 
        skip: int = 0, 
        limit: int = 100
):
    return crud.user.get_multi(skip=skip, limit=limit)
```
Здесь мы сначала создаем нужные `permissions`, а затем передаем в `Depends`.

## Итог
- Первый вариант удобен когда для большинства `endpoints` будут разные `permissions`.
- Второй вариант, когда мы будем переиспользовать `permissions`.
