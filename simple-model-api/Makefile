# reporpuse: this is a simple model API that can be used to serve a model- in this case training a model on a dataset and serving it with a simple API

.PHONY: help install test lint format clean build run docker-build docker-run train-model

# Default target
help:
	@echo "Available commands:"
	@echo "  install      - Install Python dependencies"
	@echo "  test         - Run tests with pytest"
	@echo "  lint         - Run linting with flake8"
	@echo "  format       - Format code with black"
	@echo "  clean        - Clean up cache files"
	@echo "  build        - Build Docker image"
	@echo "  run          - Run the FastAPI service locally"
	@echo "  docker-run   - Run the service in Docker"
	@echo "  train-model  - Train and save the sample model"

# Install dependencies
install:
	pip install -r requirements.txt

# Run tests
test:
	pytest tests/ -v --cov=app --cov-report=term-missing

# Run linting
lint:
	flake8 app/ tests/ --max-line-length=88 --ignore=E203,W503

# Format code
format:
	black app/ tests/ --line-length=88

# Clean up
clean:
	find . -type f -name "*.pyc" -delete
	find . -type d -name "__pycache__" -delete
	find . -type d -name "*.egg-info" -exec rm -rf {} +
	rm -rf .pytest_cache/
	rm -rf .coverage

# Build Docker image
build:
	docker build -t simple-model-api .

# Run locally
run:
	uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Run in Docker
docker-run: build
	docker run -p 8000:8000 simple-model-api

# Train the sample model
train-model:
	python scripts/train_model.py

# Development setup
dev-setup: install train-model
	@echo "Development environment setup complete!"

# Full CI check
ci: lint format test
	@echo "CI checks passed!" 