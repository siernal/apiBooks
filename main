import os
from fastapi import FastAPI, HTTPException, Query, Depends
from pydantic import BaseModel, Field
from typing import List, Optional, Dict
import uuid
from datetime import datetime
from sqlalchemy import create_engine, Column, String, Integer, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session, relationship
import re
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware

# ==================== КОНФИГУРАЦИЯ ДЛЯ RENDER.COM ====================
# Используем переменную окружения для DATABASE_URL или SQLite по умолчанию
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./movie_storage.db")

# Для PostgreSQL на Render.com нужно заменить postgres:// на postgresql://
if DATABASE_URL and DATABASE_URL.startswith("postgres://"):
    DATABASE_URL = DATABASE_URL.replace("postgres://", "postgresql://", 1)

# Настройка engine в зависимости от типа БД
if DATABASE_URL.startswith("sqlite"):
    engine = create_engine(
        DATABASE_URL, 
        connect_args={"check_same_thread": False}
    )
else:
    # Для PostgreSQL
    engine = create_engine(DATABASE_URL)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


# ==================== МОДЕЛИ БД ====================
class DBCabinet(Base):
    __tablename__ = "cabinets"

    id = Column(String, primary_key=True, index=True)
    letter = Column(String, index=True)
    created_at = Column(DateTime, default=datetime.now)

    # Связь с кассетами
    cassettes = relationship("DBCassette", back_populates="cabinet", cascade="all, delete-orphan")


class DBCassette(Base):
    __tablename__ = "cassettes"

    id = Column(String, primary_key=True, index=True)
    cabinet_id = Column(String, ForeignKey("cabinets.id", ondelete="CASCADE"))
    shelf = Column(Integer)
    movie_title_en = Column(String)
    release_year = Column(Integer)
    director = Column(String, nullable=True)
    created_at = Column(DateTime, default=datetime.now)

    # Связь со шкафом
    cabinet = relationship("DBCabinet", back_populates="cassettes")


# Создаем таблицы
Base.metadata.create_all(bind=engine)


# ==================== PYDANTIC МОДЕЛИ ====================
class CabinetCreate(BaseModel):
    letter: str = Field(..., description="Буква шкафа A-Z")


class ShelfInfo(BaseModel):
    cassette_count: int
    cassette_ids: List[str]


class CabinetResponse(BaseModel):
    id: str
    letter: str
    created_at: datetime
    shelves: Dict[int, ShelfInfo]

    class Config:
        from_attributes = True


class Cassette(BaseModel):
    cabinet_id: str
    shelf: int
    movie_title_en: str = Field(..., min_length=1, description="Название фильма на английском")
    release_year: int = Field(..., ge=1888, le=datetime.now().year, description="Год выпуска")
    director: Optional[str] = Field(None, description="Режиссер на английском")


class CassetteResponse(BaseModel):
    id: str
    cabinet_id: str
    shelf: int
    movie_title_en: str
    release_year: int
    director: Optional[str]
    created_at: datetime

    class Config:
        from_attributes = True


class CassetteWithCabinetInfo(CassetteResponse):
    cabinet_letter: str


class CassetteCreate(BaseModel):
    cabinet_id: Optional[str] = Field(None, description="ID шкафа (либо cabinet_id, либо cabinet_letter)")
    cabinet_letter: Optional[str] = Field(None, description="Буква шкафа A-Z (либо cabinet_id, либо cabinet_letter)")
    shelf: int = Field(..., ge=1, le=4, description="Полка 1-4")
    movie_title_en: str = Field(..., min_length=1, description="Название фильма на английском")
    release_year: int = Field(..., ge=1888, le=datetime.now().year, description="Год выпуска")
    director: Optional[str] = Field(None, description="Режиссер на английском")


# ==================== DEPENDENCY ====================
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# ==================== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ====================
def validate_cabinet_letter(letter: str):
    """Проверка, что шкаф - это буква A-Z"""
    if len(letter) != 1 or not letter.isalpha():
        raise HTTPException(status_code=400, detail="Cabinet letter must be a single letter A-Z")
    return letter.upper()


def validate_shelf(shelf: int):
    """Проверка, что полка от 1 до 4"""
    if shelf < 1 or shelf > 4:
        raise HTTPException(status_code=400, detail="Shelf must be between 1 and 4")
    return shelf


def validate_year(year: int):
    """Проверка года выпуска"""
    current_year = datetime.now().year
    if year < 1888 or year > current_year:
        raise HTTPException(status_code=400, detail=f"Year must be between 1888 and {current_year}")
    return year


def validate_english_text(text: str, field_name: str):
    """Проверка, что текст содержит только английские буквы и знаки препинания"""
    if not text:
        return text

    if not re.match(r'^[A-Za-z0-9\s\.,!?\-\'":;()]+$', text):
        raise HTTPException(
            status_code=400,
            detail=f"{field_name} must contain only English characters, numbers, and punctuation"
        )
    return text


def cabinet_exists(cabinet_id: str, db: Session):
    """Проверка существования шкафа"""
    cabinet = db.query(DBCabinet).filter(DBCabinet.id == cabinet_id).first()
    if not cabinet:
        raise HTTPException(status_code=400, detail=f"Cabinet with id '{cabinet_id}' not found")
    return cabinet


# ==================== FastAPI APP ====================
app = FastAPI(
    title="Movie Storage API",
    description="API для хранения видеокассет по шкафам и полкам",
    version="1.0.0",
    docs_url="/docs",  # Swagger UI
    redoc_url="/redoc"  # ReDoc documentation
)

# Добавляем CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Монтируем статические файлы
app.mount("/", StaticFiles(directory="static", html=True), name="static")

# ==================== Root endpoint ====================
@app.get("/")
def root():
    """Корневой эндпоинт с информацией о сервисе"""
    return {
        "service": "Movie Storage API",
        "version": "1.0.0",
        "status": "operational",
        "documentation": {
            "swagger": "/docs",
            "redoc": "/redoc"
        },
        "environment": os.getenv("RENDER", "development")
    }


# ==================== Health check endpoint ====================
@app.get("/health")
def health_check(db: Session = Depends(get_db)):
    """Эндпоинт для проверки здоровья сервиса (для Render.com)"""
    try:
        # Проверяем подключение к БД
        db.execute("SELECT 1")
        return {
            "status": "healthy",
            "database": "connected",
            "environment": os.getenv("RENDER", "development")
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Database connection failed: {str(e)}")


# ==================== ЭНДПОИНТЫ ДЛЯ ШКАФОВ ====================
@app.post("/cabinets/", status_code=201, response_model=CabinetResponse)
def create_cabinet(cabinet: CabinetCreate, db: Session = Depends(get_db)):
    """Создать новый шкаф с буквой A-Z и уникальным ID"""
    letter = validate_cabinet_letter(cabinet.letter)

    # Генерируем уникальный ID для шкафа
    cabinet_id = str(uuid.uuid4())[:8]

    # Создаем шкаф в БД
    db_cabinet = DBCabinet(
        id=cabinet_id,
        letter=letter,
        created_at=datetime.now()
    )

    db.add(db_cabinet)
    db.commit()
    db.refresh(db_cabinet)

    # Формируем информацию о полках (все пустые)
    shelves_info = {}
    for shelf_num in range(1, 5):
        shelves_info[shelf_num] = ShelfInfo(
            cassette_count=0,
            cassette_ids=[]
        )

    return CabinetResponse(
        id=db_cabinet.id,
        letter=db_cabinet.letter,
        created_at=db_cabinet.created_at,
        shelves=shelves_info
    )


@app.get("/cabinets/", status_code=200, response_model=List[CabinetResponse])
def list_cabinets(db: Session = Depends(get_db)):
    """Получить список всех шкафов"""
    db_cabinets = db.query(DBCabinet).all()
    cabinets_list = []

    for db_cabinet in db_cabinets:
        # Получаем все кассеты для этого шкафа
        cassettes_in_cabinet = db.query(DBCassette).filter(DBCassette.cabinet_id == db_cabinet.id).all()

        # Группируем кассеты по полкам
        shelves_dict = {1: [], 2: [], 3: [], 4: []}
        for cassette in cassettes_in_cabinet:
            shelves_dict[cassette.shelf].append(cassette.id)

        # Формируем информацию о полках
        shelves_info = {}
        for shelf_num in range(1, 5):
            shelves_info[shelf_num] = ShelfInfo(
                cassette_count=len(shelves_dict[shelf_num]),
                cassette_ids=shelves_dict[shelf_num]
            )

        cabinets_list.append(
            CabinetResponse(
                id=db_cabinet.id,
                letter=db_cabinet.letter,
                created_at=db_cabinet.created_at,
                shelves=shelves_info
            )
        )

    return cabinets_list


@app.get("/cabinets/{cabinet_id}", status_code=200, response_model=CabinetResponse)
def get_cabinet(cabinet_id: str, db: Session = Depends(get_db)):
    """Получить информацию о шкафе по ID с информацией о полках"""
    db_cabinet = cabinet_exists(cabinet_id, db)

    # Получаем все кассеты для этого шкафа
    cassettes_in_cabinet = db.query(DBCassette).filter(DBCassette.cabinet_id == cabinet_id).all()

    # Группируем кассеты по полкам
    shelves_dict = {1: [], 2: [], 3: [], 4: []}
    for cassette in cassettes_in_cabinet:
        shelves_dict[cassette.shelf].append(cassette.id)

    # Формируем информацию о полках
    shelves_info = {}
    for shelf_num in range(1, 5):
        shelves_info[shelf_num] = ShelfInfo(
            cassette_count=len(shelves_dict[shelf_num]),
            cassette_ids=shelves_dict[shelf_num]
        )

    return CabinetResponse(
        id=db_cabinet.id,
        letter=db_cabinet.letter,
        created_at=db_cabinet.created_at,
        shelves=shelves_info
    )


@app.delete("/cabinets/{cabinet_id}", status_code=200)
def delete_cabinet(cabinet_id: str, db: Session = Depends(get_db)):
    """Удалить шкаф (только если пустой)"""
    db_cabinet = cabinet_exists(cabinet_id, db)

    # Проверяем, есть ли кассеты в шкафу
    cassettes_count = db.query(DBCassette).filter(DBCassette.cabinet_id == cabinet_id).count()
    if cassettes_count > 0:
        raise HTTPException(status_code=400, detail="Cannot delete non-empty cabinet")

    db.delete(db_cabinet)
    db.commit()

    return {"message": f"Cabinet '{db_cabinet.letter}' (id: {cabinet_id}) deleted successfully"}


@app.get("/cabinets/letter/{letter}", status_code=200, response_model=List[CabinetResponse])
def get_cabinets_by_letter(letter: str, db: Session = Depends(get_db)):
    """Получить все шкафы с указанной буквой"""
    letter = validate_cabinet_letter(letter)

    # Ищем все шкафы с этой буквой
    db_cabinets = db.query(DBCabinet).filter(DBCabinet.letter == letter).all()

    if not db_cabinets:
        raise HTTPException(status_code=400, detail=f"No cabinets found with letter '{letter}'")

    matching_cabinets = []
    for db_cabinet in db_cabinets:
        # Получаем кассеты для шкафа
        cassettes_in_cabinet = db.query(DBCassette).filter(DBCassette.cabinet_id == db_cabinet.id).all()

        # Группируем по полкам
        shelves_dict = {1: [], 2: [], 3: [], 4: []}
        for cassette in cassettes_in_cabinet:
            shelves_dict[cassette.shelf].append(cassette.id)

        # Формируем информацию о полках
        shelves_info = {}
        for shelf_num in range(1, 5):
            shelves_info[shelf_num] = ShelfInfo(
                cassette_count=len(shelves_dict[shelf_num]),
                cassette_ids=shelves_dict[shelf_num]
            )

        matching_cabinets.append(
            CabinetResponse(
                id=db_cabinet.id,
                letter=db_cabinet.letter,
                created_at=db_cabinet.created_at,
                shelves=shelves_info
            )
        )

    return matching_cabinets


# ==================== ЭНДПОИНТЫ ДЛЯ КАССЕТ ====================
@app.post("/cassettes/", status_code=201, response_model=CassetteResponse)
def add_cassette(cassette: CassetteCreate, db: Session = Depends(get_db)):
    """Добавить новую кассету (по ID шкафа или по букве)"""

    # Определяем cabinet_id
    cabinet_id = None

    # Вариант 1: Передан ID шкафа
    if cassette.cabinet_id:
        cabinet_id = cassette.cabinet_id
        # Проверяем существование шкафа
        cabinet_exists(cabinet_id, db)

    # Вариант 2: Передана буква шкафа
    elif cassette.cabinet_letter:
        letter = validate_cabinet_letter(cassette.cabinet_letter)

        # Ищем первый шкаф с этой буквой
        db_cabinet = db.query(DBCabinet).filter(DBCabinet.letter == letter).first()

        if not db_cabinet:
            raise HTTPException(
                status_code=400,
                detail=f"No cabinet found with letter '{letter}'. Please create a cabinet first."
            )
        cabinet_id = db_cabinet.id

    # Вариант 3: Ничего не передано
    else:
        raise HTTPException(
            status_code=400,
            detail="Either cabinet_id or cabinet_letter must be provided"
        )

    # Проверяем полку
    shelf = validate_shelf(cassette.shelf)

    # Проверяем год
    year = validate_year(cassette.release_year)

    # Проверяем, что название на английском
    movie_title_en = validate_english_text(cassette.movie_title_en, "Movie title")

    # Проверяем режиссера на английском, если он указан
    director = None
    if cassette.director:
        director = validate_english_text(cassette.director, "Director")

    # Генерируем уникальный ID для кассеты
    cassette_id = str(uuid.uuid4())[:8]

    # Создаем кассету в БД
    db_cassette = DBCassette(
        id=cassette_id,
        cabinet_id=cabinet_id,
        shelf=shelf,
        movie_title_en=movie_title_en,
        release_year=year,
        director=director,
        created_at=datetime.now()
    )

    db.add(db_cassette)
    db.commit()
    db.refresh(db_cassette)

    return CassetteResponse(
        id=db_cassette.id,
        cabinet_id=db_cassette.cabinet_id,
        shelf=db_cassette.shelf,
        movie_title_en=db_cassette.movie_title_en,
        release_year=db_cassette.release_year,
        director=db_cassette.director,
        created_at=db_cassette.created_at
    )


@app.get("/cassettes/{cassette_id}", status_code=200, response_model=CassetteResponse)
def get_cassette(cassette_id: str, db: Session = Depends(get_db)):
    """Получить информацию о кассете"""
    db_cassette = db.query(DBCassette).filter(DBCassette.id == cassette_id).first()

    if not db_cassette:
        raise HTTPException(status_code=400, detail="Cassette not found")

    return CassetteResponse(
        id=db_cassette.id,
        cabinet_id=db_cassette.cabinet_id,
        shelf=db_cassette.shelf,
        movie_title_en=db_cassette.movie_title_en,
        release_year=db_cassette.release_year,
        director=db_cassette.director,
        created_at=db_cassette.created_at
    )


@app.get("/cassettes/", status_code=200, response_model=List[CassetteResponse])
def list_all_cassettes(db: Session = Depends(get_db)):
    """Получить список всех кассет"""
    db_cassettes = db.query(DBCassette).all()

    return [
        CassetteResponse(
            id=c.id,
            cabinet_id=c.cabinet_id,
            shelf=c.shelf,
            movie_title_en=c.movie_title_en,
            release_year=c.release_year,
            director=c.director,
            created_at=c.created_at
        )
        for c in db_cassettes
    ]


@app.get("/search/", status_code=200, response_model=List[CassetteWithCabinetInfo])
def search_cassettes(
        title: Optional[str] = Query(None, description="Поиск по названию фильма"),
        director: Optional[str] = Query(None, description="Поиск по режиссеру"),
        db: Session = Depends(get_db)
):
    """Поиск кассет по названию и/или режиссеру"""

    if not title and not director:
        raise HTTPException(
            status_code=400,
            detail="Please provide at least one search parameter: title or director"
        )

    query = db.query(DBCassette)

    if title:
        query = query.filter(DBCassette.movie_title_en.ilike(f"%{title}%"))

    if director:
        query = query.filter(DBCassette.director.ilike(f"%{director}%"))

    db_cassettes = query.all()
    results = []

    for db_cassette in db_cassettes:
        # Получаем букву шкафа
        cabinet = db.query(DBCabinet).filter(DBCabinet.id == db_cassette.cabinet_id).first()
        cabinet_letter = cabinet.letter if cabinet else "Unknown"

        results.append({
            "id": db_cassette.id,
            "cabinet_id": db_cassette.cabinet_id,
            "shelf": db_cassette.shelf,
            "movie_title_en": db_cassette.movie_title_en,
            "release_year": db_cassette.release_year,
            "director": db_cassette.director,
            "created_at": db_cassette.created_at,
            "cabinet_letter": cabinet_letter
        })

    return results


@app.put("/cassettes/{cassette_id}", status_code=200, response_model=CassetteResponse)
def update_cassette(cassette_id: str, cassette: Cassette, db: Session = Depends(get_db)):
    """Обновить информацию о кассете"""
    db_cassette = db.query(DBCassette).filter(DBCassette.id == cassette_id).first()

    if not db_cassette:
        raise HTTPException(status_code=400, detail="Cassette not found")

    # Проверяем существование нового шкафа
    cabinet_exists(cassette.cabinet_id, db)

    # Валидация
    shelf = validate_shelf(cassette.shelf)
    year = validate_year(cassette.release_year)
    movie_title_en = validate_english_text(cassette.movie_title_en, "Movie title")

    director = None
    if cassette.director:
        director = validate_english_text(cassette.director, "Director")

    # Обновляем данные
    db_cassette.cabinet_id = cassette.cabinet_id
    db_cassette.shelf = shelf
    db_cassette.movie_title_en = movie_title_en
    db_cassette.release_year = year
    db_cassette.director = director

    db.commit()
    db.refresh(db_cassette)

    return CassetteResponse(
        id=db_cassette.id,
        cabinet_id=db_cassette.cabinet_id,
        shelf=db_cassette.shelf,
        movie_title_en=db_cassette.movie_title_en,
        release_year=db_cassette.release_year,
        director=db_cassette.director,
        created_at=db_cassette.created_at
    )


@app.delete("/cassettes/{cassette_id}", status_code=200)
def delete_cassette(cassette_id: str, db: Session = Depends(get_db)):
    """Удалить кассету"""
    db_cassette = db.query(DBCassette).filter(DBCassette.id == cassette_id).first()

    if not db_cassette:
        raise HTTPException(status_code=400, detail="Cassette not found")

    db.delete(db_cassette)
    db.commit()

    return {"message": "Cassette deleted successfully"}


@app.get("/storage/{cabinet_id}", status_code=200)
def view_cabinet(cabinet_id: str, db: Session = Depends(get_db)):
    """Просмотреть содержимое шкафа по ID"""
    cabinet_exists(cabinet_id, db)

    cabinet = db.query(DBCabinet).filter(DBCabinet.id == cabinet_id).first()
    cassettes_in_cabinet = db.query(DBCassette).filter(DBCassette.cabinet_id == cabinet_id).all()

    result = {
        "cabinet_id": cabinet_id,
        "cabinet_letter": cabinet.letter,
        "shelves": {}
    }

    # Группируем кассеты по полкам
    shelves_dict = {1: [], 2: [], 3: [], 4: []}
    for cassette in cassettes_in_cabinet:
        shelves_dict[cassette.shelf].append(cassette)

    for shelf_num in range(1, 5):
        shelf_cassettes = []
        for cassette in shelves_dict[shelf_num]:
            shelf_cassettes.append({
                "id": cassette.id,
                "movie_title_en": cassette.movie_title_en,
                "release_year": cassette.release_year,
                "director": cassette.director
            })

        result["shelves"][f"shelf_{shelf_num}"] = {
            "shelf_number": shelf_num,
            "cassette_count": len(shelf_cassettes),
            "cassettes": shelf_cassettes
        }

    return result


@app.get("/storage/letter/{letter}", status_code=200)
def view_cabinets_by_letter(letter: str, db: Session = Depends(get_db)):
    """Просмотреть содержимое всех шкафов с указанной буквой"""
    letter = validate_cabinet_letter(letter)

    # Ищем все шкафы с этой буквой
    cabinets_list = db.query(DBCabinet).filter(DBCabinet.letter == letter).all()

    if not cabinets_list:
        raise HTTPException(status_code=400, detail=f"No cabinets found with letter '{letter}'")

    matching_cabinets = []
    for cabinet in cabinets_list:
        cabinet_info = view_cabinet(cabinet.id, db)
        matching_cabinets.append(cabinet_info)

    return {
        "letter": letter,
        "total_cabinets": len(matching_cabinets),
        "cabinets": matching_cabinets
    }


@app.get("/storage/{cabinet_id}/{shelf_number}", status_code=200)
def view_shelf(cabinet_id: str, shelf_number: int, db: Session = Depends(get_db)):
    """Просмотреть содержимое конкретной полки"""
    cabinet_exists(cabinet_id, db)
    shelf_number = validate_shelf(shelf_number)

    cabinet = db.query(DBCabinet).filter(DBCabinet.id == cabinet_id).first()
    cassettes_on_shelf = db.query(DBCassette).filter(
        DBCassette.cabinet_id == cabinet_id,
        DBCassette.shelf == shelf_number
    ).all()

    shelf_cassettes = []
    for cassette in cassettes_on_shelf:
        shelf_cassettes.append({
            "id": cassette.id,
            "movie_title_en": cassette.movie_title_en,
            "release_year": cassette.release_year,
            "director": cassette.director
        })

    return {
        "cabinet_id": cabinet_id,
        "cabinet_letter": cabinet.letter,
        "shelf": shelf_number,
        "cassette_count": len(shelf_cassettes),
        "cassettes": shelf_cassettes
    }


@app.get("/status", status_code=200)
def system_status(db: Session = Depends(get_db)):
    """Получить статус системы"""
    total_cassettes = db.query(DBCassette).count()
    total_cabinets = db.query(DBCabinet).count()

    # Получаем все шкафы
    all_cabinets = db.query(DBCabinet).all()

    # Группируем шкафы по буквам
    cabinets_by_letter = {}
    for cabinet in all_cabinets:
        if cabinet.letter not in cabinets_by_letter:
            cabinets_by_letter[cabinet.letter] = []
        cabinets_by_letter[cabinet.letter].append({
            "id": cabinet.id,
            "created_at": cabinet.created_at.isoformat() if cabinet.created_at else None
        })

    # Считаем заполненность полок
    shelf_stats = {}
    for cabinet in all_cabinets:
        for shelf_num in range(1, 5):
            count = db.query(DBCassette).filter(
                DBCassette.cabinet_id == cabinet.id,
                DBCassette.shelf == shelf_num
            ).count()

            if count > 0:
                key = f"{cabinet.letter}-{shelf_num}"
                if key not in shelf_stats:
                    shelf_stats[key] = []
                shelf_stats[key].append({
                    "cabinet_id": cabinet.id,
                    "count": count
                })

    return {
        "status": "operational",
        "environment": os.getenv("RENDER", "development"),
        "total_cassettes": total_cassettes,
        "total_cabinets": total_cabinets,
        "cabinets_by_letter": cabinets_by_letter,
        "shelves_with_cassettes": shelf_stats,
        "timestamp": datetime.now().isoformat()
    }


# Для локального запуска
if __name__ == "__main__":
    import uvicorn
    
    port = int(os.getenv("PORT", 8000))
    uvicorn.run(
        "main:app",  # Изменено для работы с Render.com
        host="0.0.0.0", 
        port=port,
        reload=False  # Отключаем reload на продакшене
    )
