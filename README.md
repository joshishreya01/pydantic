# pydantic
Create a school blog API with FastApI and Mongodb (motor) with data validation with Pydantic

Here's a simple implementation of a school blog API using FastAPI, MongoDB (Motor), and Pydantic for data validation. This API supports CRUD operations for blogs.

 Prerequisites
1. Install the required dependencies:
   ```bash
   pip install fastapi uvicorn motor pydantic[email]
   ```

2. Set up a MongoDB instance (locally or on the cloud, e.g., MongoDB Atlas).

---

 Project Structure
```plaintext
school_blog_api/
├── main.py
└── models.py
```

---

 Code Implementation

`models.py` (Pydantic Models for Validation)
```python
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime


class Blog(BaseModel):
    title: str = Field(..., max_length=100, description="Title of the blog")
    content: str = Field(..., description="Content of the blog")
    author: str = Field(..., max_length=50, description="Author's name")
    tags: Optional[list[str]] = Field(default=[], description="Tags for the blog")
    created_at: Optional[datetime] = Field(default_factory=datetime.utcnow)
    updated_at: Optional[datetime] = None


class BlogUpdate(BaseModel):
    title: Optional[str] = Field(None, max_length=100)
    content: Optional[str] = None
    author: Optional[str] = Field(None, max_length=50)
    tags: Optional[list[str]] = None
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

---

 main.py (FastAPI Implementation)
python
from fastapi import FastAPI, HTTPException, Depends
from motor.motor_asyncio import AsyncIOMotorClient
from bson import ObjectId
from models import Blog, BlogUpdate
from typing import List

app = FastAPI()

# MongoDB Configuration
MONGO_URL = "mongodb://localhost:27017"
DATABASE_NAME = "school_blog"
COLLECTION_NAME = "blogs"

client = AsyncIOMotorClient(MONGO_URL)
db = client[DATABASE_NAME]
collection = db[COLLECTION_NAME]

# Helpers
def blog_helper(blog) -> dict:
    return {
        "id": str(blog["_id"]),
        "title": blog["title"],
        "content": blog["content"],
        "author": blog["author"],
        "tags": blog.get("tags", []),
        "created_at": blog["created_at"],
        "updated_at": blog.get("updated_at"),
    }


# Routes
@app.post("/blogs", response_model=dict, status_code=201)
async def create_blog(blog: Blog):
    blog_dict = blog.dict()
    result = await collection.insert_one(blog_dict)
    new_blog = await collection.find_one({"_id": result.inserted_id})
    return blog_helper(new_blog)


@app.get("/blogs", response_model=List[dict])
async def get_all_blogs():
    blogs = await collection.find().to_list(100)
    return [blog_helper(blog) for blog in blogs]


@app.get("/blogs/{blog_id}", response_model=dict)
async def get_blog(blog_id: str):
    blog = await collection.find_one({"_id": ObjectId(blog_id)})
    if not blog:
        raise HTTPException(status_code=404, detail="Blog not found")
    return blog_helper(blog)


@app.put("/blogs/{blog_id}", response_model=dict)
async def update_blog(blog_id: str, blog_update: BlogUpdate):
    blog_dict = {k: v for k, v in blog_update.dict().items() if v is not None}
    result = await collection.update_one({"_id": ObjectId(blog_id)}, {"$set": blog_dict})
    if result.modified_count == 0:
        raise HTTPException(status_code=404, detail="Blog not found or no changes made")
    updated_blog = await collection.find_one({"_id": ObjectId(blog_id)})
    return blog_helper(updated_blog)


@app.delete("/blogs/{blog_id}", response_model=dict)
async def delete_blog(blog_id: str):
    result = await collection.delete_one({"_id": ObjectId(blog_id)})
    if result.deleted_count == 0:
        raise HTTPException(status_code=404, detail="Blog not found")
    return {"detail": "Blog deleted successfully"}
```

---

### Run the Application
Start the FastAPI server:
```bash
uvicorn main:app --reload
```

---

API Endpoints
1. Create a Blog 
   `POST /blogs`  
   Body:
   ```json
   {
       "title": "My First Blog",
       "content": "This is the content of my first blog.",
       "author": "John Doe",
       "tags": ["education", "school"]
   }
   ```

2. Get All Blogs
   `GET /blogs`

3. Get a Blog by ID
   `GET /blogs/{blog_id}`

4. Update a Blog
   `PUT /blogs/{blog_id}`  
   Body (Partial Update):
   ```json
   {
       "title": "Updated Title"
   }
   ```

5. Delete a Blog 
   `DELETE /blogs/{blog_id}`

---

### Features
- Data Validation: Using Pydantic for input validation.
- MongoDB Integration: With Motor for asynchronous operations.
- CRUD Operations: Fully functional create, read, update, delete endpoints.

This setup provides a foundational API for managing blogs. You can expand it further with features like authentication or more advanced query capabilities.
