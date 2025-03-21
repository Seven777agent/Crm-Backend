from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey, DateTime
from sqlalchemy.orm import sessionmaker, declarative_base, relationship, Session
from datetime import datetime

DATABASE_URL = "sqlite:///./crm.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

app = FastAPI()

# Модель лида
class Lead(Base):
    __tablename__ = "leads"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)
    phone = Column(String, nullable=False, unique=True)
    email = Column(String, nullable=True)
    budget = Column(String, nullable=True)
    source = Column(String, nullable=True)
    status = Column(String, default="new")  # Возможные статусы: new, in_progress, waiting_response, closed, declined
    created_at = Column(DateTime, default=datetime.utcnow)
    notes = Column(String, nullable=True)

Base.metadata.create_all(bind=engine)

# Получение сессии БД
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Добавление лида
@app.post("/leads/")
def create_lead(name: str, phone: str, email: str = None, budget: str = None, source: str = None, notes: str = None, db: Session = Depends(get_db)):
    db_lead = Lead(name=name, phone=phone, email=email, budget=budget, source=source, notes=notes)
    db.add(db_lead)
    db.commit()
    db.refresh(db_lead)
    return db_lead

# Получение всех лидов
@app.get("/leads/")
def get_leads(db: Session = Depends(get_db)):
    return db.query(Lead).all()

# Обновление статуса лида
@app.put("/leads/{lead_id}/")
def update_lead_status(lead_id: int, status: str, db: Session = Depends(get_db)):
    lead = db.query(Lead).filter(Lead.id == lead_id).first()
    if not lead:
        raise HTTPException(status_code=404, detail="Лид не найден")
    if status not in ["new", "in_progress", "waiting_response", "closed", "declined"]:
        raise HTTPException(status_code=400, detail="Некорректный статус")
    lead.status = status
    db.commit()
    return {"message": "Статус обновлен"}

# Удаление лида
@app.delete("/leads/{lead_id}/")
def delete_lead(lead_id: int, db: Session = Depends(get_db)):
    lead = db.query(Lead).filter(Lead.id == lead_id).first()
    if not lead:
        raise HTTPException(status_code=404, detail="Лид не найден")
    db.delete(lead)
    db.commit()
    return {"message": "Лид удален"}
