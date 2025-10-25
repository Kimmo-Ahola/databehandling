# Project Setup - Lathund

## Länk till SQLAlchemys dokumentation

Försök att hitta information i denna innan ni använder ChatGPT. Dokumentationen är oftast den säkraste källan till korrekt och uppdaterad information.
<a href="https://docs.sqlalchemy.org/" target=_blank>Länk till dokumentationen</a>

## Installera nödvändiga paket

> [!WARNING]
> Se till att ni har en venv installerad innan ni sätter igång.

- Kör följande kommandon i terminalen
```bash
pip install sqlalchemy
pip install pymysql
pip install alembic
```

## Skapa en python-fil för databasen

Vi kallar exempelvis filen för db.py och lägger den i en mapp som också heter db/database.

1. Lägg till vår ***Database Connection String***

```python
# username och password är vad ni har valt vid installationen av MySQL på datorn.
# database_name är det namn ni väljer när ni kör CREATE DATABASE i MySQL
DATABASE_URL = "mysql+pymysql://username:password@localhost:3306/database_name"
```

2. Vi använder ***Create engine*** för att skapa konfiguration om hur vi ska koppla upp oss mot database.

```python
engine = create_engine(DATABASE_URL, echo=False) # True when debugging

SessionLocal = sessionmaker(autocommit=False, autoflush = True, bind = endgine)
Session = SessionLocal # Vi skriver över SQLAlchemys Session-klass för att använda vår egna.
```
### Övningar
1. Vad gör echo-parametern?
1. Vad gör autocommit-parametern? (den ska vara False när ni jobbar)
1. Vad gör autoflush-parametern?


## Skapa en basklass för modeller

- Krav för att SQLAlchemy ska kunna modellera våra tabeller. Våra tabeller ***ärver*** från denna klass. Vi kan låta den vara tom tills vidare eftersom den innehåller allt vi behöver.
```python
# base.py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
	pass
	
import models # add models to __init.py__ in models folder
```

## Skapa en första databasmodell

- Vi skapar en python-klass för Driver och ärver från base.py. Vi gör den så minimal som möjligt med endast kolumn: vår primärnyckel (PK).
```python
# driver.py

class Driver(Base):
	id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
```

- Lägg till detta i __init__.py

```python
# __init__.py inside models folder
from .driver import Driver
```

### Övningar
Googla och ta reda på detta:

1. Behöver vi verkligen ha autoincrement=True?
1. Måste det vara en Integer på primary key?
1. Varför behöver vi inte ha unique=True på vår primary key?


## Initiera alembic

- I terminalen skriver vi detta kommando. Se till att vi står i projektets rotmapp. 

```bash
alembic init alembic
```
Detta kommando skapar en ny mapp, alembic, som innehåller nödvändiga filer för att vi ska kunna köra databasmigrationer.

Vi måste ändra inställningar i env.py inuti alembic-mappen. Lägg till detta:

```python

config = context.config # Denna finns redan. Låt stå oförändrad.
config.set_main_option("sqlalchemy.url", DATABASE_URL) # lägg till denna

target_metadata = Base.metadata # ändra denna
```

## Skapa en migration

```
alembic revision --autogenerate -m "created first table"
```

kör en upgrade och kolla i din databas för att se vad som händer

```
alembic upgrade head
```

- Vi lägger nu till en ny kolumn, fullname, i driver.py.

```python
from sqlalchemy import Integer, String
from sqlalchemy.orm import mapped_column, Mapped
from models.base import Base

class Driver(Base):
    __tablename__ = "drivers"


    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
	# Lägg till fullname efter migration
    fullname: Mapped[str] = mapped_column(String(255), nullable=False)
```

- Vi kör nu en ny migration och uppgraderar. Kolla i databasen vad som händer.

```bash
alembic upgrade head
```

### Övning
- Testa dessa kommandon och kolla i databasen. Vad händer?
```
alembic downgrade -1
alembic upgrade 1
```

## Skapa en python-klass licenses.py


```python
from datetime import date
from sqlalchemy import Date, Integer
from sqlalchemy.orm import mapped_column, Mapped
from models.base import Base

class License(Base):

    __tablename__ = "licenses"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    issued_at: Mapped[date] = mapped_column(Date) # visa här hur man kan hitta insert_at på dokumentationen
```

## Ny migration

Kör en migration och visa att ingenting nytt händer.
Vi kollar i migrationsfilen och ser att den är tom. Varför?
Jo, vi har inte lagt till vår nya modell i init.py

Vi kör en downgrade -1, raderar migrationsfilen och gör om.


## Skapa koppling med foreign key

### One-to-One eller One-to-Many?

Fundera på vilket förhållande vi har här.



```python
class Driver(Base):
    __tablename__ = "drivers"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)

    fullname: Mapped[str] = mapped_column(String(255), nullable=False)

    licenses: Mapped[List["License"]] = relationship(back_populates="driver") # List["License"] skapar ett icke-optional beroende. Listan kan vara tom. Optional används för skalärer.
```




```python
class License(Base):
    __tablename__ = "licenses"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)

    issued_at: Mapped[date] = mapped_column(Date, insert_default=func.now())

    driver_id: Mapped[int] = mapped_column(Integer, ForeignKey("drivers.id"), nullable=False, unique=False) # FK är namnet på tabellen, vilket är drivers. unique=True skapar One-to-One, unique=False skapar One-to-Many

    driver: Mapped["Driver"] = relationship(back_populates="drivers")
```

>[!WARNING]
> - Alembics migration inte ger ett namn på FK som standardbeteende och detta kommer att skapa problem vid downgrade.
> - Vi har två val:
>   1. Vi ändrar manuellt efter migration genom att uppdatera namnet i migrationen med det som står i databasen.
>   1. Vi lägger till följande naming convention i metadata inuti basklassen base.py

```python
class Base(DeclarativeBase):
    metadata = MetaData(naming_convention={
  "ix": "ix_%(column_0_label)s",
  "uq": "uq_%(table_name)s_%(column_0_name)s",
  "ck": "ck_%(table_name)s_%(constraint_name)s",
  "fk": "%(column_0_name)s",
  "pk": "pk_%(table_name)s"
})
```

Att ändra manuellt är dock det absolut enklaste sättet att göra det på.
## Exempel på simpel CRUD

I main.py lägger vi till session.begin():

1. Gör en query först. Visa att det blir en tom lista.
2. Lägg till en ny driver
3. Printa ut resultatet

```python
# kod för main.py
from sqlalchemy import select
from db.db import Session
from models.driver import Driver
  

with Session.begin() as session:
	new_driver = Driver(fullname="Kimmo Ahola")
	session.add(new_driver)

with Session.begin() as session:
    stmt = select(Driver)
    result = session.execute(stmt).scalars().all()
    print(result)
```

1. Försöka att lägga till en License utan driver
2. Visa vad som händer
3. Programmet krascar. Varför?


```python
# kod för main.py
from sqlalchemy import select
from db.db import Session
from models.driver import Driver
  

with Session.begin() as session:
	new_driver = Driver(fullname="Kimmo Ahola")
	new_license = License()
	session.add(new_driver)
	session.add(new_license)

with Session.begin() as session:
    stmt = select(Driver)
    result = session.execute(stmt).scalars().all()
    print(result)
```

1. Ändra koden till följande
2. Inspektera ID på driver. Vad har hänt? Vi har skippat en??

```python
# kod för main.py
from sqlalchemy import select
from db.db import Session
from models.driver import Driver
  

with Session.begin() as session:
	new_driver = Driver(fullname="Kimmo Ahola")
	session.add(License(driver=new_driver))

with Session.begin() as session:
    stmt = select(Driver)
    result = session.execute(stmt).scalars().all()
    print(result)
```

### Övningar:
#### Övning 1: 
1. Testa att gå in i din db.py och ändra echo=False till echo=True
2. Kör en ny session där du skapar License med Driver och sedan kör en select.
3. Vad ser du i konsolfönstret?

#### Övning 2:
1. Lägg till en ny driver men lägg nu till ett specifikt datum bakåt i tiden.
2. Lägg till en ny driver men lägg till ett datum framåt i tiden.
3. Fundera på om dessa beteenden ska vara möjliga. Hur lägger vi in logik som hindrar till exempel framtida/historiska datum?

#### Övning 3:
1. Uppdatera en förares namn

#### Övning 4:
1. Undersök följande kommando med en join:
```python
stmt = select(License).join(License.driver)
with Session.begin() as session:
    licenses = session.execute(stmt).scalars().all()
    for lic in licenses:
        print(f"License ID: {lic.id}, issued_at: {lic.issued_at}, driver: {lic.driver.fullname}")
   ```
   2. Testa nu att köra följande kommando utan vår join
```python
stmt = select(License)
with Session.begin() as session:
    licenses = session.execute(stmt).scalars().all()
    for lic in licenses:
        print(f"License ID: {lic.id}, issued_at: {lic.issued_at}, driver: {lic.driver.fullname}")
```
   
3. Blir det någon skillnad? Varför/Varför inte?

Svaret är att vi får samma resultat. Vi bör dock inte få det eftersom vi inte gör en join, i sql måste vi ju göra det?
Anledningen till detta beteende är att SqlAlchemy gör två sökningar åt oss i det andra fallet, dvs de lägger till en ny sökning för att hämta informationen vi fick från vår join. Med echo=True kan vi se i konsolen:

```
2025-10-24 12:39:07,347 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-10-24 12:39:07,349 INFO sqlalchemy.engine.Engine SELECT licenses.id, licenses.issued_at, licenses.driver_id
FROM licenses
2025-10-24 12:39:07,349 INFO sqlalchemy.engine.Engine [generated in 0.00016s] {}
2025-10-24 12:39:07,351 INFO sqlalchemy.engine.Engine SELECT drivers.id AS drivers_id, drivers.fullname AS drivers_fullname
FROM drivers
WHERE drivers.id = %(pk_1)s
2025-10-24 12:39:07,351 INFO sqlalchemy.engine.Engine [generated in 0.00022s] {'pk_1': 4}
License ID: 1, issued_at: 2025-10-24, driver: Kenneth Ahola
```

Alltså sker det två sökningar till databasen om vi inte specifikt säger join i början. Detta är potentiellt dåligt eftersom en extra sökning till databasen kan vara kostsam. Detta fenomen kallas för N+1 problem och det är därför alltid värt att ha echo=True när man sitter och utvecklar för att sedan ta bort det när man pushar upp koden till produktion.

#### Övning 5
1. Skapa funktioner för CRUD (Create, Read, Update, Delete) för din driver
2. Dessa ska ligga i en ny fil (i en klass). Kom ihåg try except på rätt ställen!
```python
# skelett för en hur vi kan lägga till CRUD-funktioner inuti en klass

class Demo():

	def create_driver(session: Session, driver: Driver) -> Driver | None:
		new_driver = ....
		session.add(...)
		
		return ...
		
	def create_driver_2(session: Session, **kwargs) -> Driver | None:
		new_driver = ....
		session.add(...)
		
		return ...

# Samma som ovan, men denna gång skickar vi in sessionen till konstruktorn.
class AnotherDemo():
	def __init__(self, session: Session):
		self.session = session


	def create_driver(self, driver: Driver) -> Driver | None:
		new_driver = ....
		self.session.add(...)
		
		return ...
		
	def create_driver_2(self, **kwargs) -> Driver | None:
		new_driver = ....
		self.session.add(...)
		
		return ...

```
