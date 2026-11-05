# Scaling FastAPI for High Concurrency (1,000+ Concurrent Requests)

Handling 1,000 concurrent requests reliably requires optimizing the application layer, process management, database connections, and infrastructure safeguards. 

Below is an in-depth breakdown of the four key pillars of performance and reliability, followed by a complete production-grade FastAPI boilerplate.

---

## Key Pillars of High Concurrency

### 1. Asynchronous Programming (`async` / `await`)
FastAPI is built on Starlette and uses **ASGI (Asynchronous Server Gateway Interface)**.
* **Non-blocking I/O:** For I/O-bound tasks (database queries, HTTP API calls, file operations), use asynchronous drivers (`httpx`, `asyncpg`, `redis-py`). When an `await` point is reached, the event loop yields control to process other incoming requests.
* **Avoid Blocking the Event Loop:** Synchronous blocking code (e.g., standard `requests` library, heavy synchronous CPU tasks, standard `time.sleep`) stops the entire event loop, freezing all concurrent requests on that worker.
* **Thread Pools for Legacy Code:** If you must call a synchronous library, define the endpoint with standard `def` (without `async`), and FastAPI will execute it inside an external thread pool automatically, keeping the main event loop responsive.

### 2. Worker Processes (Uvicorn + Gunicorn)
A single Python process runs on a single CPU core. Due to the Python **GIL (Global Interpreter Lock)**, an event loop can only utilize one core.
* **Multi-Process Deployment:** Use **Gunicorn** as a process manager with **Uvicorn** worker classes (`uvicorn.workers.UvicornWorker`).
* **Sizing Workers:** A standard rule of thumb for I/O-bound FastAPI workloads is:
  $$\text{Workers} = (2 \times \text{CPU Cores}) + 1$$
* **Resource Isolation:** Deploying multiple workers ensures that if one worker crashes or encounters an uncaught exception, other workers continue serving incoming requests without total service failure.

### 3. Connection Pooling
Creating and closing database or cache connections per request incurs network and handshake overhead that will crash your system under load.
* **Reuse Connections:** Keep a persistent pool of connections created on application startup and closed on shutdown.
* **Pool Sizing:** Size the connection pool according to database limits and the number of workers. For example, if PostgreSQL allows 200 total connections across 4 workers:
  $$\text{Max Pool Size per Worker} = \frac{\text{PostgreSQL Max Connections}}{\text{Number of Workers}} - \text{Safety Margin}$$
* **Connection Timeout & Recycling:** Set a queue timeout so requests fail gracefully instead of hanging indefinitely when the pool is exhausted.

### 4. Rate Limiting and Resilience
Rate limiting protects downstream systems, prevents Denial of Service (DoS) attacks, and maintains service quality for critical traffic.
* **Token Bucket / Sliding Window:** Use Redis to store request counters across distributed processes/nodes.
* **Graceful Degradation:** Return HTTP `429 Too Many Requests` with a `Retry-After` header when clients exceed their rate limits.

---

## Production-Grade FastAPI Implementation

Here is a full Python application demonstrating non-blocking HTTP requests, asynchronous database pooling using **SQLAlchemy 2.0 (Async)**, Redis rate-limiting, and lifecycle state management.

```python
import os
import time
import asyncio
from typing import AsyncGenerator
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request, HTTPException, status, Depends
from fastapi.responses import JSONResponse
import httpx
import redis.asyncio as redis
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import text

# =====================================================================
# Configurations
# =====================================================================
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql+asyncpg://user:password@localhost:5432/appdb")
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

# =====================================================================
# Database Setup (SQLAlchemy 2.0 Async with Connection Pooling)
# =====================================================================
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,          # Persistent connections per worker
    max_overflow=10,       # Temporary connections beyond pool_size under high load
    pool_timeout=30,       # Seconds to wait before throwing a pool timeout error
    pool_recycle=1800,     # Recycle connections every 30 minutes to avoid stale sockets
    pool_pre_ping=True,    # Test connectivity before using a pooled connection
)

AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autoflush=False
)

class Base(DeclarativeBase):
    pass

class AuditLog(Base):
    __tablename__ = "audit_logs"
    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    endpoint: Mapped[str]
    response_time_ms: Mapped[float]

# =====================================================================
# Redis Connection Management
# =====================================================================
redis_client: redis.Redis = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Lifespan context manager handles startup and shutdown tasks efficiently.
    Connection pools are initialized once and cleaned up on application exit.
    """
    global redis_client
    # Startup
    redis_client = redis.from_url(REDIS_URL, encoding="utf-8", decode_responses=True)
    
    # Initialize DB Tables (For demo purposes; use Alembic in production)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
        
    yield
    
    # Shutdown
    await redis_client.close()
    await engine.dispose()

# FastAPI Application Declaration
app = FastAPI(
    title="High-Concurrency FastAPI Architecture",
    version="1.0.0",
    lifespan=lifespan
)

# =====================================================================
# Dependencies
# =====================================================================
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Dependency injection for database sessions with automatic context handling."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def rate_limiter(request: Request):
    """
    Sliding window rate-limiter powered by Redis.
    Limits clients to 100 requests per minute based on IP address.
    """
    client_ip = request.client.host if request.client else "127.0.0.1"
    key = f"rate_limit:{client_ip}"
    limit = 100
    window = 60

    current_requests = await redis_client.incr(key)
    if current_requests == 1:
        await redis_client.expire(key, window)

    if current_requests > limit:
        ttl = await redis_client.ttl(key)
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Rate limit exceeded. Try again later.",
            headers={"Retry-After": str(ttl)}
        )

# =====================================================================
# Middleware for Performance Profiling
# =====================================================================
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.perf_counter()
    response = await call_next(request)
    process_time = (time.perf_counter() - start_time) * 1000
    response.headers["X-Process-Time-MS"] = f"{process_time:.2f}"
    return response

# =====================================================================
# Endpoints
# =====================================================================
@app.get("/health", status_code=status.HTTP_200_OK)
async def health_check():
    """Simple ping-pong health check endpoint for load balancers."""
    return {"status": "healthy"}

@app.get("/data", dependencies=[Depends(rate_limiter)])
async def fetch_aggregated_data(
    db: AsyncSession = Depends(get_db_session)
):
    """
    High-concurrency endpoint executing concurrent external API calls 
    and asynchronous database queries.
    """
    # 1. Non-blocking concurrent external HTTP requests using asyncio.gather
    async with httpx.AsyncClient(timeout=5.0) as client:
        task1 = client.get("https://httpbin.org/delay/1")
        task2 = client.get("https://httpbin.org/delay/1")
        
        # Execute both requests concurrently
        res1, res2 = await asyncio.gather(task1, task2, return_exceptions=True)

    # 2. Asynchronous DB Query
    result = await db.execute(text("SELECT 1"))
    db_val = result.scalar()

    return {
        "status": "success",
        "db_result": db_val,
        "external_api_1": res1.status_code if isinstance(res1, httpx.Response) else "failed",
        "external_api_2": res2.status_code if isinstance(res2, httpx.Response) else "failed",
    }
```

---

## Production Deployment Configuration

To run this application under a load of 1,000 concurrent requests, run it using **Gunicorn** combined with **Uvicorn workers**.

### 1. Command Line Execution

```bash
gunicorn main:app \
  --workers 9 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --backlog 2048 \
  --keep-alive 65 \
  --max-requests 10000 \
  --max-requests-jitter 1000
```

### 2. Parameter Explanation

| Parameter | Recommended Value | Purpose |
| :--- | :--- | :--- |
| `--workers` | `(2 x Cores) + 1` | Spawns multiple OS processes to utilize all CPU cores and bypass Python's GIL. |
| `--worker-class` | `uvicorn.workers.UvicornWorker` | Uses Uvicorn's high-performance `uvloop` event loop inside each Gunicorn process. |
| `--backlog` | `2048` | Increases the TCP socket queue depth to accept incoming bursts of connections without dropping them. |
| `--keep-alive` | `65` | Keeps TCP connections open longer to avoid overhead from repeated TCP handshakes (align with upstream load balancers like Nginx/ALB). |
| `--max-requests` | `10000` | Periodically restarts worker processes to prevent memory leaks over time. |
| `--max-requests-jitter`| `1000` | Adds randomness to worker restarts to prevent all workers from restarting simultaneously. |

---

## Architectural Best Practices Checklist

1. **Offload Heavy Computation:** If endpoints require heavy CPU computation (e.g., image processing, data science models), offload them to background task queues like **Celery** or **ARQ** with Redis/RabbitMQ brokers.
2. **Reverse Proxy & TLS Termination:** Always deploy an upstream reverse proxy (Nginx, HAProxy, or AWS ALB) to handle TLS handshakes, static files, and buffering slow HTTP clients.
3. **Caching Layer:** Cache frequent database queries in Redis to avoid hitting database connection limits under traffic spikes.