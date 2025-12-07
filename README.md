# Ai
Ai full
import openai
import deepl
import stripe
import jwt
import datetime
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# API Keys (Add your actual keys)
openai.api_key = 'YOUR_OPENAI_API_KEY'
translator = deepl.Translator('YOUR_DEEPL_API_KEY')
stripe.api_key = 'YOUR_STRIPE_API_KEY'

SECRET_KEY = "your_secret_key"  # For JWT Token Authentication
DATABASE_URL = "postgresql://username:password@localhost/freelance_platform"

# Set up the database connection
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Define database models
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
    password = Column(String)

class Project(Base):
    __tablename__ = "projects"
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    description = Column(String)
    service_type = Column(String)
    budget = Column(Float)
    status = Column(String, default="pending")  # Track the status of the project

Base.metadata.create_all(bind=engine)

# Define the Project Request data model
class ProjectRequest(BaseModel):
    title: str
    description: str
    service_type: str  # e.g., 'Content Writing', 'Coding', 'Translation'
    budget: float

# Function to handle content writing using OpenAI GPT-4
def handle_content_writing(project: ProjectRequest):
    response = openai.Completion.create(
        engine="text-davinci-003",
        prompt=project.description,
        max_tokens=500
    )
    return response.choices[0].text.strip()

# Function to handle coding projects using OpenAI Codex
def handle_coding(project: ProjectRequest):
    response = openai.Completion.create(
        engine="code-davinci-002",
        prompt=project.description,
        max_tokens=500
    )
    return response.choices[0].text.strip()

# Function to handle translation projects using DeepL
def handle_translation(project: ProjectRequest):
    response = translator.translate_text(project.description, target_lang="EN")
    return response.text

# Function to process payment using Stripe
def process_payment(project: ProjectRequest):
    charge = stripe.Charge.create(
        amount=int(project.budget * 100),  # Convert to cents
        currency="usd",
        description=f"Payment for project: {project.title}",
        source="tok_visa",  # Placeholder for real source
    )
    return charge

# JWT Token Handling
def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.datetime.utcnow() + datetime.timedelta(hours=1)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm="HS256")

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        return None

# Main function to handle project lifecycle
def handle_project(project_data):
    # Convert input project data to ProjectRequest class
    project = ProjectRequest(**project_data)
    
    # Process based on service type
    if project.service_type == "Content Writing":
        project_content = handle_content_writing(project)
    elif project.service_type == "Coding":
        project_content = handle_coding(project)
    elif project.service_type == "Translation":
        project_content = handle_translation(project)
    else:
        return {"error": "Invalid service type"}

    # Save project to the database (SQLAlchemy)
    db = SessionLocal()
    db_project = Project(title=project.title, description=project.description, 
                         service_type=project.service_type, budget=project.budget, status="completed")
    db.add(db_project)
    db.commit()
    db.refresh(db_project)

    # Process payment
    charge = process_payment(project)

    # Return final response with project details and payment confirmation
    return {
        "message": "Project completed and payment processed successfully",
        "project_content": project_content,
        "charge_details": charge,
        "project_id": db_project.id  # Return project ID for tracking
    }

# Example project input (you can modify or accept this via user input)
project_data = {
    "title": "AI Content Generation",
    "description": "Write a 500-word article about the impact of AI in healthcare.",
    "service_type": "Content Writing",
    "budget": 100.0  # Example budget
}

# Run the project processing
result = handle_project(project_data)

# Print the result (or send it to the user)
import json
print(json.dumps(result, indent=2))
