# API-de-Atletas
Uma API RESTful desenvolvida com FastAPI, SQLAlchemy e fastapi-pagination para gerenciar atletas, com suporte a filtros, paginação e tratamento de exceções de integridade.
Funcionalidades
 Cadastro de atletas com validação de CPF único

 Filtro por nome e CPF via query parameters

 Resposta customizada com nome, centro de treinamento e categoria

 Tratamento de exceção IntegrityError com mensagem personalizada

 Paginação com limit e offset usando fastapi-pagination

 Documentação automática via Swagger UI

 database.py:
 from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

# URL de conexão com SQLite (pode trocar por PostgreSQL, MySQL etc.)
SQLALCHEMY_DATABASE_URL = "sqlite:///./atletas.db"

engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Base para os modelos
Base = declarative_base()

# Dependência para obter sessão do banco
def get_db():
    db = SessionLocal()
    try:
        yield db

models.py:

from sqlalchemy import Column, Integer, String
from database import Base

# Modelo da tabela de atletas
class Atleta(Base):
    __tablename__ = "atletas"

    id = Column(Integer, primary_key=True, index=True)
    nome = Column(String, index=True)
    cpf = Column(String, unique=True, index=True)
    centro_treinamento = Column(String)
    categoria = Column(String)

schemas.py:

from pydantic import BaseModel

# Schema base para criação
class AtletaCreate(BaseModel):
    nome: str
    cpf: str
    centro_treinamento: str
    categoria: str

# Schema para resposta customizada
class AtletaOut(BaseModel):
    nome: str
    centro_treinamento: str
    categoria: str

    class Config:
        orm_mode = True



crud.py:

from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from models import Atleta
from schemas import AtletaCreate

# Função para criar atleta com tratamento de exceção
def criar_atleta(db: Session, atleta: AtletaCreate):
    novo_atleta = Atleta(**atleta.dict())
    try:
        db.add(novo_atleta)
        db.commit()
        db.refresh(novo_atleta)
        return novo_atleta
    except IntegrityError:
        db.rollback()
        raise ValueError(f"Já existe um atleta cadastrado com o cpf: {atleta.cpf}")

# Função para buscar atletas com filtros
def buscar_atletas(db: Session, nome: str = None, cpf: str = None):
    query = db.query(Atleta)
    if nome:
        query = query.filter(Atleta.nome.ilike(f"%{nome}%"))
    if cpf:
        query = query.filter(Atleta.cpf == cpf)
    return query.all()


main.py:

from fastapi import FastAPI, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from fastapi_pagination import add_pagination, paginate
from fastapi_pagination.limit_offset import LimitOffsetPage

from database import Base, engine, get_db
from models import Atleta
from schemas import AtletaCreate, AtletaOut
from crud import criar_atleta, buscar_atletas

# Cria as tabelas no banco
Base.metadata.create_all(bind=engine)

app = FastAPI(title="API de Atletas")

# Endpoint para criar atleta
@app.post("/atletas", status_code=201)
def criar(atleta: AtletaCreate, db: Session = Depends(get_db)):
    try:
        return criar_atleta(db, atleta)
    except ValueError as e:
        raise HTTPException(status_code=303, detail=str(e))

# Endpoint para listar atletas com filtros e paginação
@app.get("/atletas", response_model=LimitOffsetPage[AtletaOut])
def listar(
    nome: str = Query(None),
    cpf: str = Query(None),
    db: Session = Depends(get_db)
):
    atletas = buscar_atletas(db, nome, cpf)
    return paginate(atletas)

# Ativa paginação
add_pagination(app)

    finally:
        db.close()
