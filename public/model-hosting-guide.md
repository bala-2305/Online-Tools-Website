# Model Hosting Guide

## Prerequisites
1. Server Requirements:
   - Minimum 16GB RAM (32GB recommended)
   - GPU with at least 8GB VRAM for faster inference
   - 100GB+ storage space
   - Ubuntu 20.04 or later

## Step 1: Set Up the Environment

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and dependencies
sudo apt install python3-pip python3-venv
python3 -m venv venv
source venv/bin/activate

# Install required packages
pip install torch torchvision torchaudio
pip install transformers accelerate
pip install fastapi uvicorn
```

## Step 2: Create Server Application

Create a new file `app.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch
import json

app = FastAPI()

# Load models
models = {
    "deepseek-7b": {
        "name": "deepseek-ai/deepseek-7b-base",
        "model": None,
        "tokenizer": None
    },
    "mistral-7b": {
        "name": "mistralai/Mistral-7B-v0.1",
        "model": None,
        "tokenizer": None
    }
}

# Load models on startup
@app.on_event("startup")
async def load_models():
    for model_id, config in models.items():
        config["tokenizer"] = AutoTokenizer.from_pretrained(config["name"])
        config["model"] = AutoModelForCausalLM.from_pretrained(
            config["name"],
            torch_dtype=torch.float16,
            device_map="auto"
        )

class GenerationRequest(BaseModel):
    model_id: str
    prompt: str
    max_length: int = 1000
    temperature: float = 0.7
    top_p: float = 0.9

@app.post("/generate")
async def generate_text(request: GenerationRequest):
    if request.model_id not in models:
        raise HTTPException(status_code=400, detail="Model not found")
    
    model_config = models[request.model_id]
    
    inputs = model_config["tokenizer"](
        request.prompt, 
        return_tensors="pt"
    ).to("cuda")
    
    with torch.no_grad():
        outputs = model_config["model"].generate(
            **inputs,
            max_length=request.max_length,
            temperature=request.temperature,
            top_p=request.top_p,
            do_sample=True
        )
    
    response = model_config["tokenizer"].decode(outputs[0], skip_special_tokens=True)
    return {"generated_text": response}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Step 3: Create Docker Configuration

Create `Dockerfile`:

```dockerfile
FROM nvidia/cuda:11.8.0-runtime-ubuntu22.04

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip3 install -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Start the application
CMD ["python3", "app.py"]
```

Create `requirements.txt`:

```text
torch
transformers
accelerate
fastapi
uvicorn
pydantic
```

## Step 4: Deployment Options

### A. Direct Server Deployment

```bash
# Start the server
python3 app.py
```

### B. Docker Deployment

```bash
# Build Docker image
docker build -t genai-model-server .

# Run container
docker run --gpus all -p 8000:8000 genai-model-server
```

### C. Cloud Deployment (AWS)

1. Create EC2 instance with GPU support (e.g., g4dn.xlarge)
2. Install NVIDIA drivers and Docker
3. Deploy using Docker as above

## Step 5: Update Client Configuration

Update the API endpoints in `genai-builder.html`:

```javascript
const API_ENDPOINTS = {
    'deepseek-7b': 'http://your-server:8000/generate',
    'mistral-7b': 'http://your-server:8000/generate',
    // ... other endpoints
};
```

## Security Considerations

1. Add API key authentication:
```python
from fastapi.security import APIKeyHeader
from fastapi import Security, HTTPException

api_key_header = APIKeyHeader(name="X-API-Key")

@app.post("/generate")
async def generate_text(
    request: GenerationRequest,
    api_key: str = Security(api_key_header)
):
    if api_key != YOUR_API_KEY:
        raise HTTPException(status_code=401)
    # ... rest of the code
```

2. Set up HTTPS using Nginx reverse proxy
3. Implement rate limiting
4. Add input validation and sanitization

## Monitoring and Scaling

1. Add Prometheus metrics for monitoring
2. Set up logging with ELK stack
3. Use load balancer for multiple instances
4. Implement caching for frequent requests

## Cost Optimization

1. Use spot instances for non-critical workloads
2. Implement model quantization for reduced memory usage
3. Use caching for frequent requests
4. Scale instances based on demand

## Troubleshooting

Common issues and solutions:
1. OOM errors: Reduce batch size or use model quantization
2. Slow inference: Check GPU utilization and batch requests
3. High latency: Implement request queuing and caching