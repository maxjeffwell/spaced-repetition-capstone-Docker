# AGENTS.md - Project Documentation for AI Agents

# Project Overview

**IntervalAI (Spaced Repetition Capstone)** is a neural-enhanced spaced repetition learning system that combines the traditional SM-2 algorithm with machine learning to optimize review intervals for better memory retention.

## Purpose and Goals

The primary goal of this project is to improve learning efficiency by predicting optimal review intervals based on individual learner patterns using TensorFlow.js neural networks. The system moves beyond deterministic spaced repetition algorithms to a data-driven approach that adapts to each user's unique learning characteristics.

## Target Audience

- **Students and Learners**: Individuals seeking to optimize their study schedules and improve retention
- **Educators**: Professionals interested in adaptive learning technologies and educational innovation
- **Researchers**: Those studying spaced repetition algorithms and machine learning applications in education

## Business Domain

- **Domain**: Educational Technology (EdTech)
- **Subdomain**: Adaptive Learning and Spaced Repetition
- **Focus**: Machine Learning-Enhanced Memory Retention

## Key Features

1. **Dual Algorithm Support**: SM-2 baseline and ML-enhanced interval predictions
2. **A/B Testing Capability**: Compare algorithm performance and collect data for continuous improvement
3. **Client-Side ML Inference**: WebGPU-accelerated predictions without server round-trips
4. **Real-Time Statistics**: Progress visualization using Chart.js and D3.js
5. **Comprehensive Review History**: Track all learning data for continuous model improvement
6. **Production-Ready Infrastructure**: Kubernetes with horizontal pod autoscaling
7. **Multi-Platform Docker Images**: Support for amd64 and arm64 architectures
8. **Automated CI/CD**: GitHub Actions with Docker Hub integration

---

# Technology Stack

## Frontend Technologies

### Core Framework
- **React** (v18.3.1) - UI library for building interactive components
- **React Router DOM** (v6.30.1) - Client-side routing and navigation

### State Management
- **Redux** (v4.0.1) - Centralized state management with Flux architecture
- **Redux Thunk** (v2.3.0) - Async action creators for API calls
- **Redux Form** (v8.3.10) - Form state management with validation

### Machine Learning
- **TensorFlow.js** (v4.22.0) - JavaScript machine learning library
- **TensorFlow.js WebGPU Backend** (v4.22.0) - GPU-accelerated inference in browser

### Visualization
- **Chart.js** (v4.5.1) - Statistical charts and progress tracking
- **D3.js** (v7.9.0) - Advanced data visualization

### Build Tools
- **React Scripts** (v5.0.1) - Build and development tooling

### Utilities
- **lodash-es** (v4.17.21) - Utility functions
- **jwt-decode** (v2.2.0) - Client-side JWT token decoding

## Backend Technologies

### Runtime and Framework
- **Node.js** - JavaScript runtime environment
- **Express.js** (v5.1.0) - Web application framework

### Database
- **MongoDB Atlas** - Managed cloud database service
- **Mongoose** (v8.19.3) - MongoDB ODM with schema validation

### Authentication
- **Passport.js** (v0.7.0) - Authentication middleware
- **Passport Local** (v1.0.0) - Local username/password strategy
- **Passport JWT** (v4.0.0) - JWT token authentication strategy
- **bcryptjs** (v2.4.3) - Password hashing
- **jsonwebtoken** (v9.0.2) - JWT creation and verification

### Machine Learning
- **TensorFlow.js Node** (v4.22.0) - Server-side ML model training and inference

### Middleware
- **body-parser** (v1.18.3) - Request body parsing
- **cors** (v2.8.4) - Cross-origin resource sharing
- **morgan** (v1.9.1) - HTTP request logging

### Utilities
- **axios** (v1.13.2) - HTTP client
- **dotenv** (latest) - Environment variable management

## Infrastructure

### Containerization and Orchestration
- **Docker** - Container platform
- **Docker Compose** - Multi-container orchestration
- **Kubernetes** - Container orchestration system
- **Nginx** - Reverse proxy and static file server
- **Docker Hub** - Container image registry

### CI/CD
- **GitHub Actions** - Automated workflows
  - Continuous Integration pipeline
  - Docker Build & Push workflow

## Testing

- **Mocha** (v5.2.0) - Testing framework
- **Chai** (v4.2.0) - Assertion library
- **Chai HTTP** (v4.2.0) - HTTP integration testing
- **NYC** (v13.1.0) - Code coverage reporting

## Development Tools

- **ESLint** (v5.7.0) - Code linting and quality
- **nodemon** (v1.18.4) - Development server with auto-reload

## Python Environment (ML Training)

- **TensorFlow/Keras** - Neural network training
- **Jupyter/Google Colab** - Interactive notebook environment for model training

---

# Coding Standards

## File Organization

### Structure Rules
- Separate client and server codebases into distinct subdirectories (`spaced-repetition-capstone-client`, `spaced-repetition-capstone-server`)
- Use feature-based organization for routes, models, and components
- Keep ML-related code in dedicated `/ml` directory
- Store maintenance and testing scripts in `/scripts` directory
- Place Kubernetes manifests in `/k8s` directory

## Naming Conventions

- **File Names**: Use kebab-case (e.g., `algorithm-manager.js`, `advanced-features.js`)
- **React Components**: Use PascalCase
- **JavaScript Functions/Variables**: Use camelCase
- **ML Models**: Use descriptive names with underscores (e.g., `interval_model_advanced.h5`)

## Architecture Guidelines

### Design Principles
- Implement linked-list data structure for question ordering in user documents
- Support multiple spaced repetition algorithms (baseline SM-2 and ML-enhanced)
- Enable A/B testing between baseline and ML algorithms
- Use Redux for state management in client application
- Implement RESTful API endpoints with `/api` prefix

### Design Patterns Used
1. **Linked List for Question Ordering**: Each user document contains a 'head' pointer to the first question, with each question having a 'next' pointer
2. **Strategy Pattern for Algorithms**: Algorithm Manager orchestrates between baseline SM-2 and ML-enhanced algorithms
3. **Repository Pattern**: Mongoose models abstract database operations
4. **Middleware Chain**: Express middleware for CORS, authentication, request parsing, and error handling
5. **Redux Flux Architecture**: Unidirectional data flow with actions, reducers, and centralized store

## Security Standards

- **Never** commit `.env` files or secrets to version control
- Use JWT authentication with Passport.js strategies
- Hash passwords with bcryptjs before storage
- Store Kubernetes secrets in separate files (`secrets.yaml` excluded from git)
- Use environment variables for all sensitive configuration

## Docker Practices

- Use multi-stage builds for optimized Docker images
- Support both development and production Docker configurations
- Use Nginx as reverse proxy and static file server
- Build multi-platform images (linux/amd64, linux/arm64)

## Machine Learning Practices

- Train models with TensorFlow/Keras in Python, export for TensorFlow.js
- Store review history with both baseline and ML interval predictions for comparison
- Use 8-feature input for ML models (memoryStrength, difficultyRating, timeSinceLastReview, successRate, averageResponseTime, totalReviews, consecutiveCorrect, timeOfDay)
- Support client-side ML inference with WebGPU acceleration
- Maintain separate models for local and advanced features

## Testing Standards

- Use Mocha and Chai for backend testing
- Implement health checks in Docker and Kubernetes deployments
- Test ML predictions with realistic review data
- Run CI workflows on push and pull requests

## Documentation Requirements

- Maintain comprehensive markdown documentation (README.md, ARCHITECTURE.md, DOCKER.md, KUBERNETES.md)
- Document API endpoints and data models
- Provide setup guides for local development and production deployment
- Include CI/CD pipeline documentation

## Version Control

- Use submodules for client and server repositories
- Include build status badges in README
- Maintain separate CI workflows for client and server

## Performance Optimization

- Implement horizontal pod autoscaling in Kubernetes (2-10 server pods, 2-8 client pods)
- Set resource limits and requests for container resources
- Use connection pooling for MongoDB Atlas
- Enable WebGPU acceleration for client-side ML inference

---

# Project Structure

```
spaced-repetition-capstone/
├── Root Documentation
│   ├── README.md (Project overview, architecture, quick start)
│   ├── DOCKER.md (Docker Compose deployment guide)
│   ├── KUBERNETES.md (Kubernetes/LKE deployment guide)
│   └── CICD.md (GitHub Actions CI/CD configuration)
│
├── Infrastructure Configuration
│   ├── docker-compose.yml (Main compose configuration)
│   ├── docker-compose.override.yml (Local development overrides)
│   ├── docker-compose.prod.yml (Production configuration)
│   └── nginx/
│       └── nginx.conf (Reverse proxy configuration)
│
├── Kubernetes Manifests (k8s/)
│   ├── namespace.yaml (Namespace definition)
│   ├── configmap.yaml (Environment configuration)
│   ├── secrets.yaml.example (Secret template)
│   ├── server-deployment.yaml (Server deployment spec)
│   ├── server-service.yaml (Server service spec)
│   ├── client-deployment.yaml (Client deployment spec)
│   ├── client-service.yaml (Client service spec)
│   ├── ingress.yaml (Nginx ingress with TLS)
│   └── hpa.yaml (Horizontal Pod Autoscaler)
│
├── Screenshots (screenshots/)
│   └── Application UI images for documentation
│
├── Client Application (spaced-repetition-capstone-client/)
│   ├── Configuration
│   │   ├── package.json (Dependencies and scripts)
│   │   ├── Dockerfile (Multi-stage build for production)
│   │   ├── nginx.conf (Client-specific nginx config)
│   │   ├── static.json (Static hosting configuration)
│   │   └── render.yaml (Render deployment config)
│   │
│   ├── Public Assets (public/)
│   │   ├── index.html (HTML entry point)
│   │   ├── models/ (TensorFlow.js model files)
│   │   ├── fonts/ (Custom fonts)
│   │   └── _redirects (SPA routing configuration)
│   │
│   ├── Source Code (src/)
│   │   ├── index.js (Application entry point)
│   │   ├── index.css (Global styles)
│   │   ├── store.js (Redux store configuration)
│   │   ├── config.js (API configuration)
│   │   ├── local-storage.js (Client storage utilities)
│   │   ├── validators.js (Form validation)
│   │   ├── suppress-legacy-warnings.js (Compatibility fixes)
│   │   │
│   │   ├── actions/ (Redux action creators)
│   │   ├── reducers/ (Redux reducers)
│   │   ├── components/ (React UI components)
│   │   └── services/ (API service layer, ML inference)
│   │
│   └── Documentation
│       ├── README.md (Client-specific documentation)
│       ├── WEBGPU_INTEGRATION.md (WebGPU setup guide)
│       ├── WEBGPU_TEST_RESULTS.md (Performance benchmarks)
│       ├── UPGRADE_NOTES.md (Dependency upgrade logs)
│       └── FINAL_UPGRADE_SUMMARY.md (Migration summary)
│
└── Server Application (spaced-repetition-capstone-server/)
    ├── Configuration
    │   ├── package.json (Dependencies and scripts)
    │   ├── Dockerfile (Standard and GPU variants)
    │   ├── Dockerfile.gpu (GPU-enabled image)
    │   ├── config.js (Server configuration)
    │   ├── render.yaml (Render deployment config)
    │   └── index.js (Express server entry point)
    │
    ├── Database (db/)
    │   ├── db-mongoose.js (MongoDB connection setup)
    │   └── seed/ (Database seeding scripts)
    │
    ├── Models (models/)
    │   ├── user.js (User schema with embedded questions)
    │   └── question.js (Question schema definition)
    │
    ├── Authentication (auth/)
    │   ├── passport.js (Passport strategy configuration)
    │   └── authentication.js (Auth middleware)
    │
    ├── API Routes (routes/)
    │   ├── auth.js (Login, registration, token refresh)
    │   ├── questions.js (Question CRUD, answer submission)
    │   └── users.js (User management, settings)
    │
    ├── Spaced Repetition Algorithms (algorithms/)
    │   ├── sm2.js (Baseline SM-2 implementation)
    │   └── algorithm-manager.js (Algorithm orchestration)
    │
    ├── Machine Learning (ml/)
    │   ├── model.js (TensorFlow.js model loader)
    │   ├── ml-service.js (ML prediction service)
    │   ├── advanced-features.js (Feature engineering)
    │   ├── interval_model_local.h5 (Trained Keras model)
    │   ├── interval_model_advanced.h5 (Advanced model)
    │   ├── saved-model/ (TensorFlow.js format)
    │   ├── saved-model-old-8features/ (Legacy model)
    │   └── README.md (ML documentation)
    │
    ├── Training Scripts (scripts/)
    │   ├── Python Training
    │   │   ├── train-model.js (Node.js training wrapper)
    │   │   ├── train-model-local.py (Local model training)
    │   │   ├── train-model-advanced.py (Advanced feature training)
    │   │   ├── simulate-ml-predictions.py (Prediction simulator)
    │   │   └── test-prediction.py (Model testing)
    │   │
    │   ├── Data Management
    │   │   ├── extract-training-data.js (Data extraction)
    │   │   ├── clean-training-data.js (Data preprocessing)
    │   │   ├── simulate-reviews.js (Review simulation)
    │   │   ├── simulate-realistic-reviews.js (Realistic data generation)
    │   │   └── simulate-reviews-production.js (Production simulation)
    │   │
    │   ├── Model Management
    │   │   ├── export-model-for-browser.js (Browser format export)
    │   │   ├── convert-keras3-to-keras2.js (Model conversion)
    │   │   ├── fix-model-format.js (Format fixes)
    │   │   └── continuous-training-pipeline.js (Automated training)
    │   │
    │   ├── Testing and Benchmarking
    │   │   ├── test-ml-predictions.js (Prediction accuracy tests)
    │   │   ├── test-with-real-data.js (Real data validation)
    │   │   ├── test-advanced-features.js (Feature testing)
    │   │   ├── benchmark-all-algorithms.js (Algorithm comparison)
    │   │   ├── compare-client-server-predictions.js (Cross-platform validation)
    │   │   └── webgpu-performance-test.js (WebGPU benchmarking)
    │   │
    │   └── Database Utilities
    │       ├── check-intervals.js (Interval validation)
    │       ├── fix-corrupted-questions.js (Data repair)
    │       ├── fix-demo-user.js (Demo account setup)
    │       ├── convert-baseline-to-ml.js (Algorithm migration)
    │       └── debug-ml-predictions.js (Debugging tools)
    │
    ├── Utilities (utils/)
    │   ├── hash.js (Password hashing)
    │   ├── question-helpers.js (Question manipulation)
    │   └── seed-database.js (Database seeding)
    │
    ├── Testing (test/)
    │   ├── setup.test.js (Test configuration)
    │   └── advanced-features.test.js (ML feature tests)
    │
    ├── Training Data
    │   ├── training-data.json (Raw training data)
    │   ├── training-data-clean.json (Preprocessed data)
    │   ├── benchmark-results.json (Performance metrics)
    │   ├── webgpu-comparison-results.json (WebGPU benchmarks)
    │   ├── training-output.log (Training logs)
    │   └── colab_training_notebook.ipynb (Google Colab notebook)
    │
    ├── Python Environment (venv-training/)
    │   └── Virtual environment for ML training scripts
    │
    └── Documentation (docs/)
        ├── README.md (Server overview)
        ├── ARCHITECTURE.md (System architecture)
        ├── API_DOCUMENTATION.md (API reference)
        ├── DATA_MODEL.md (Database schema)
        ├── ML_TRAINING_GUIDE.md (Training instructions)
        ├── TRAINING_DATA.md (Data collection guide)
        ├── TRAINING_DATA_COLLECTION.md (Collection strategy)
        ├── TRAINING_GOOGLE_COLAB.md (Colab training guide)
        ├── CONTINUOUS_TRAINING.md (Pipeline documentation)
        ├── TESTING_SUMMARY.md (Test results)
        ├── TESTING_RESULTS.md (Detailed test data)
        ├── WEBGPU_TEST_RESULTS.md (WebGPU performance)
        ├── PROJECT_STATUS.md (Current status)
        ├── ADVANCED_FEATURES_SUMMARY.md (ML features overview)
        └── ADVANCED_FEATURES.md (Detailed ML documentation)
```

---

# External Resources

## Framework Documentation

### React Ecosystem
- **React Documentation**: https://react.dev/
  - Official React framework documentation for building user interfaces
- **Redux**: https://redux.js.org/
  - State management library documentation
- **Redux Thunk**: https://github.com/reduxjs/redux-thunk
  - Middleware for async actions
- **Redux Form**: https://redux-form.com/
  - Form state management with validation
- **React Router**: https://reactrouter.com/
  - Client-side routing and navigation

### Backend Framework
- **Express.js Guide**: https://expressjs.com/
  - Node.js web framework documentation
- **Passport.js Documentation**: https://www.passportjs.org/
  - Authentication middleware for Node.js

## Machine Learning

- **TensorFlow.js Documentation**: https://www.tensorflow.org/js
  - JavaScript machine learning library for training and deploying models
- **TensorFlow.js WebGPU Backend**: https://www.tensorflow.org/js/guide/platform_environment
  - GPU-accelerated inference documentation
- **Google Colab**: https://colab.research.google.com/
  - Cloud-based Jupyter notebook environment for ML training

## Database

- **MongoDB Atlas Documentation**: https://docs.atlas.mongodb.com/
  - Managed MongoDB service guide with backup, monitoring, and scaling features
- **Mongoose Documentation**: https://mongoosejs.com/
  - MongoDB ODM for Node.js with schema validation

## Visualization

- **Chart.js Documentation**: https://www.chartjs.org/
  - Charting library for statistics visualization
- **D3.js Documentation**: https://d3js.org/
  - Advanced data visualization library

## Infrastructure & DevOps

- **Docker Documentation**: https://docs.docker.com/
  - Containerization platform documentation
- **Kubernetes Documentation**: https://kubernetes.io/docs/
  - Container orchestration system
- **Linode Kubernetes Engine**: https://www.linode.com/docs/guides/deploy-and-manage-lke-cluster/
  - Managed Kubernetes service documentation
- **cert-manager Documentation**: https://cert-manager.io/docs/
  - Kubernetes certificate management for SSL/TLS
- **GitHub Actions Documentation**: https://docs.github.com/en/actions
  - CI/CD workflow automation

## Security & Authentication

- **bcryptjs**: https://github.com/dcodeIO/bcrypt.js
  - Password hashing and verification library
- **jsonwebtoken**: https://github.com/auth0/node-jsonwebtoken
  - JWT creation and verification
- **jwt-decode**: https://github.com/auth0/jwt-decode
  - Client-side JWT token decoding

## Testing Tools

- **Mocha**: https://mochajs.org/
  - JavaScript testing framework
- **Chai**: https://www.chaijs.com/
  - Assertion library for testing
- **NYC**: https://github.com/istanbuljs/nyc
  - Code coverage tool

## Development Tools

- **ESLint**: https://eslint.org/
  - JavaScript linting and code quality
- **nodemon**: https://nodemon.io/
  - Development server with auto-reload

## Cloud Services

- **MongoDB Atlas**: https://www.mongodb.com/cloud/atlas
  - Managed MongoDB database hosting with automatic backups, monitoring, scaling, and high availability
- **Let's Encrypt**: https://letsencrypt.org/
  - Free SSL/TLS certificates via cert-manager
- **Render**: https://render.com/
  - Alternative cloud deployment platform
- **Docker Hub**: https://hub.docker.com/
  - Container image registry hosting project images:
    - `maxjeffwell/spaced-repetition-capstone-server`
    - `maxjeffwell/spaced-repetition-capstone-client`

## Research & Algorithms

- **SM-2 Spaced Repetition Algorithm**: https://www.supermemo.com/en/archives1990-2015/english/ol/sm2
  - SuperMemo 2 algorithm for calculating review intervals
- **Spaced Repetition Research**: Scientific research on spaced repetition and memory retention
- **Neural Networks for Adaptive Learning**: Machine learning approaches to personalized learning systems

## Project Repositories

- **Main Repository**: https://github.com/maxjeffwell/spaced-repetition-capstone
- **Server Submodule**: https://github.com/maxjeffwell/spaced-repetition-capstone-server
- **Client Submodule**: https://github.com/maxjeffwell/spaced-repetition-capstone-client

---

# Additional Context

## Architectural Decisions

### 1. Monorepo with Git Submodules
**Rationale**: Separate client and server repositories as submodules for independent CI/CD pipelines while maintaining unified deployment configuration. This allows each component to evolve independently with its own versioning and release cycle.

### 2. External MongoDB Atlas
**Rationale**: Use managed MongoDB Atlas instead of in-cluster database for automatic backups, high availability, and reduced operational overhead. This eliminates the need to manage database infrastructure and provides enterprise-grade reliability.

### 3. TensorFlow.js for Cross-Platform ML
**Rationale**: Train models in Python (TensorFlow/Keras) leveraging powerful training tools and free GPU access via Google Colab, then export to TensorFlow.js format for both Node.js server and browser client inference with WebGPU acceleration. This provides model consistency across platforms.

### 4. Embedded Questions in User Documents
**Rationale**: Store questions as embedded documents within user collection using linked-list structure for efficient question ordering and retrieval. This reduces database queries and keeps related data together.

### 5. A/B Testing Architecture
**Rationale**: Support three algorithm modes (baseline, ML, A/B test) to compare performance and collect data for continuous model improvement. This enables data-driven optimization of the ML model.

### 6. Docker Multi-Platform Images
**Rationale**: Build and publish multi-platform Docker images (amd64, arm64) for broad deployment compatibility across different hardware architectures, including Apple Silicon and ARM-based cloud instances.

### 7. Kubernetes with HPA
**Rationale**: Deploy to Kubernetes with Horizontal Pod Autoscaling for automatic scaling based on CPU/memory usage to handle variable load efficiently and cost-effectively.

### 8. Client-Side ML Inference
**Rationale**: Load ML model in browser for real-time predictions without server round-trips, improving response time and reducing server load. WebGPU acceleration provides near-native performance.

## Data Model Summary

### Users Collection

The application uses a MongoDB collection with the following schema:

#### Top-Level Fields
- `_id`: ObjectId (unique identifier)
- `username`: String (unique, indexed)
- `password`: String (bcrypt hashed)
- `firstName`: String
- `lastName`: String
- `head`: Number (index of first question in linked list)

#### Embedded Questions Array
Each user document contains an embedded questions array with:
- `_id`: ObjectId
- `question`: String (question text)
- `answer`: String (correct answer)
- `memoryStrength`: Number (0-5, current memory strength)
- `next`: Number (index of next question in linked list)
- `difficultyRating`: Number (0-1 difficulty score)
- `lastReviewed`: Date
- `timesCorrect`: Number (success count)
- `timesIncorrect`: Number (failure count)

#### Review History (Nested in Questions)
Each question maintains a detailed review history:
- `timestamp`: Date
- `recalled`: Boolean (correct/incorrect)
- `responseTime`: Number (milliseconds)
- `intervalUsed`: Number (days until next review)
- `algorithmUsed`: String (enum: "baseline", "ml")
- `baselineInterval`: Number (what baseline predicted)
- `mlInterval`: Number (what ML predicted)

#### User Settings
- `useMLAlgorithm`: Boolean
- `algorithmMode`: String (enum: "baseline", "ml", "ab-test")

## Machine Learning Architecture

### Input Features (8 total)
1. `memoryStrength` - Current memory strength (0-5)
2. `difficultyRating` - Question difficulty (0-1)
3. `timeSinceLastReview` - Days since last review
4. `successRate` - Historical success percentage
5. `averageResponseTime` - Average response time in milliseconds
6. `totalReviews` - Total number of reviews
7. `consecutiveCorrect` - Consecutive correct answers
8. `timeOfDay` - Normalized time of day (0-1)

### Neural Network Architecture
1. **Input Layer**: 8 features
2. **Dense Layer 1**: 32 neurons, ReLU activation
3. **Dropout**: 0.2 rate (regularization)
4. **Dense Layer 2**: 16 neurons, ReLU activation
5. **Dense Layer 3**: 8 neurons, ReLU activation
6. **Output Layer**: 1 neuron (predicted interval in days)

### Training Configuration
- **Loss Function**: Mean Squared Error (MSE)
- **Optimizer**: Adam
- **Training Framework**: TensorFlow/Keras (Python)
- **Export Format**: TensorFlow.js (JSON + binary weights)

### Inference Environments
- **Server**: TensorFlow.js Node (CPU-based)
- **Client**: TensorFlow.js with WebGPU acceleration (GPU-based)

## Use Cases

### 1. Learning Session Flow
User starts session → System retrieves next question due for review → User submits answer → System calculates next interval using selected algorithm → Updates question position in linked list → Returns feedback

### 2. ML Training Pipeline Flow
Extract review history from all users → Engineer features (8 inputs) → Train neural network → Evaluate performance → Export model for client/server → Deploy updated model

### 3. Algorithm Comparison Flow
User enables A/B testing mode → System calculates both baseline and ML intervals → Randomly assigns algorithm per question → Stores both predictions in review history → Generates comparison statistics

### 4. Statistics Dashboard Flow
User views dashboard → System aggregates review history → Visualizes retention curves, success rates, and algorithm performance using Chart.js and D3

## Deployment Environments

### Local Development
- **Method**: Docker Compose
- **Services**:
  - Client (port 5000)
  - Server (port 8080)
  - Nginx reverse proxy (port 80)

### Production
- **Platform**: Kubernetes (Linode LKE)
- **Features**:
  - Horizontal Pod Autoscaling (HPA)
  - TLS/SSL with Let's Encrypt
  - MongoDB Atlas external database
  - Multi-platform Docker images
- **CI/CD**: GitHub Actions with Docker Hub registry

### Alternative Deployment
- **Platform**: Render
- **Configuration**: Static site for client, web service for server

## Development Workflow

1. **Local Development**: Use Docker Compose with hot reloading for rapid iteration
2. **Version Control**: Git commit triggers CI workflow
3. **Continuous Integration**: Automated tests and builds run via GitHub Actions
4. **Image Building**: Successful builds push multi-platform images to Docker Hub
5. **Deployment**: Kubernetes pulls latest images and deploys
6. **Auto-Scaling**: HPA scales pods based on CPU/memory load
7. **SSL Management**: cert-manager automatically manages certificate renewal

## Key Innovation Points

### Cross-Platform ML Pipeline
Models are trained in Python using TensorFlow/Keras (leveraging Google Colab for free GPU access), then exported to TensorFlow.js format. This allows the same model to run on:
- **Server**: Node.js with TensorFlow.js Node for batch processing
- **Client**: Browser with TensorFlow.js and WebGPU for real-time inference

### A/B Testing for Continuous Improvement
The system stores predictions from both baseline (SM-2) and ML algorithms for every review, regardless of which was used. This enables:
- Direct performance comparison
- Data collection for model retraining
- Evidence-based algorithm selection

### Linked-List Question Management
Questions are stored as embedded documents in the user collection, organized as a linked list:
- Efficient sequential access during learning sessions
- O(1) head retrieval for next question
- Minimal database queries during active learning

---

# Testing Instructions

## Backend Testing

### Running Tests
```bash
cd spaced-repetition-capstone-server

# Run all tests
npm test

# Run tests with coverage
npm run cover

# Run tests once (CI mode)
npm run mocha
```

### Test Structure
- **Unit Tests**: Located in `test/` directory
- **Integration Tests**: Test API endpoints with Chai HTTP
- **ML Tests**: Validate feature engineering and predictions

### Test Configuration
- **Framework**: Mocha with Chai assertions
- **Setup**: `test/setup.test.js` initializes test environment
- **Coverage**: NYC generates coverage reports

## Frontend Testing

```bash
cd spaced-repetition-capstone-client

# Run tests
npm test

# Run tests in watch mode
npm test -- --watch
```

## ML Model Testing

### Testing ML Predictions
```bash
cd spaced-repetition-capstone-server

# Test with real data
node scripts/test-with-real-data.js

# Test ML predictions
node scripts/test-ml-predictions.js

# Compare client and server predictions
node scripts/compare-client-server-predictions.js

# Benchmark algorithms
node scripts/benchmark-all-algorithms.js

# WebGPU performance test
node scripts/webgpu-performance-test.js
```

## Docker Testing

```bash
# Test local Docker builds
docker-compose up --build

# Test production configuration
docker-compose -f docker-compose.prod.yml up
```

---

# Build Steps

## Local Development Setup

### Prerequisites
- Node.js 14+ and npm
- Docker and Docker Compose
- MongoDB Atlas account
- Git

### Initial Setup

1. **Clone Repository**
```bash
git clone <repository-url>
cd spaced-repetition-capstone
git submodule update --init --recursive
```

2. **Configure Environment**
```bash
cp .env.example .env
# Edit .env with your MongoDB Atlas connection string and JWT secret
```

3. **Start with Docker Compose**
```bash
docker-compose up -d
```

4. **Access Application**
- Application: http://localhost
- API: http://localhost/api
- Health: http://localhost/health

## Manual Build (Development)

### Server
```bash
cd spaced-repetition-capstone-server
npm install
npm run dev
# Server runs on port 8080
```

### Client
```bash
cd spaced-repetition-capstone-client
npm install
npm start
# Client runs on port 3000
```

## Production Build

### Docker Images
```bash
# Build server image
cd spaced-repetition-capstone-server
docker build -t spaced-repetition-server:latest .

# Build client image
cd ../spaced-repetition-capstone-client
docker build --build-arg REACT_APP_API_BASE_URL=/api \
  -t spaced-repetition-client:latest .
```

### Multi-Platform Build
```bash
# Setup buildx
docker buildx create --name multiplatform --use

# Build and push server
docker buildx build --platform linux/amd64,linux/arm64 \
  -t maxjeffwell/spaced-repetition-capstone-server:latest \
  --push .

# Build and push client
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg REACT_APP_API_BASE_URL=/api \
  -t maxjeffwell/spaced-repetition-capstone-client:latest \
  --push .
```

### Kubernetes Deployment
```bash
# Configure kubectl
export KUBECONFIG=~/.kube/spaced-repetition-kubeconfig.yaml

# Create namespace
kubectl apply -f k8s/namespace.yaml

# Create secrets
kubectl create secret generic spaced-repetition-secrets \
  --from-literal=MONGODB_URI='your-connection-string' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  --namespace=spaced-repetition

# Apply configurations
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/server-deployment.yaml
kubectl apply -f k8s/server-service.yaml
kubectl apply -f k8s/client-deployment.yaml
kubectl apply -f k8s/client-service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml
```

## ML Model Training

### Local Training
```bash
cd spaced-repetition-capstone-server

# Create Python virtual environment
python3 -m venv venv-training
source venv-training/bin/activate  # On Windows: venv-training\Scripts\activate

# Install dependencies
pip install tensorflow numpy pandas scikit-learn

# Extract training data
node scripts/extract-training-data.js

# Train model
python scripts/train-model-local.py

# Export for browser
node scripts/export-model-for-browser.js
```

### Google Colab Training
1. Upload `colab_training_notebook.ipynb` to Google Colab
2. Mount Google Drive for data access
3. Run training cells with GPU acceleration
4. Download trained model
5. Export to TensorFlow.js format

---

# API Endpoints Reference

## Authentication
- `POST /api/auth/login` - User login
- `POST /api/auth/register` - User registration
- `POST /api/auth/refresh` - Token refresh

## Questions
- `GET /api/question/next` - Get next question due for review
- `POST /api/answer` - Submit answer and get feedback
  - Body: `{ questionId, answer, responseTime }`
  - Response: `{ correct, correctAnswer, nextInterval, algorithmUsed, baselineInterval, mlInterval, feedback }`

## Statistics
- `GET /api/stats` - Overall learning statistics
- `GET /api/stats/comparison` - Baseline vs ML comparison
- `GET /api/stats/retention-curve` - Retention over time

## ML Model
- `POST /api/ml/train` - Trigger model training
- `GET /api/ml/model` - Download latest model
- `GET /api/ml/metrics` - Model performance metrics

## User Settings
- `PATCH /api/user/settings` - Update algorithm mode
  - Body: `{ algorithmMode: 'baseline' | 'ml' | 'ab-test' }`

---

# Troubleshooting Guide

## Common Issues

### MongoDB Connection Failed
- Verify connection string in `.env`
- Check MongoDB Atlas Network Access whitelist
- Ensure database user has correct permissions

### Docker Port Conflicts
- Check for processes using ports 80, 8080, 5000
- Modify port mappings in `docker-compose.yml`

### ML Model Not Loading
- Verify model files exist in `public/models/`
- Check browser console for CORS errors
- Ensure model format is compatible with TensorFlow.js version

### Kubernetes Pod Failures
- Check pod logs: `kubectl logs -f <pod-name> -n spaced-repetition`
- Verify secrets are correctly configured
- Check resource limits and node capacity

---

# Performance Considerations

## Client-Side
- WebGPU acceleration for ML inference
- Lazy loading of components
- Optimized Redux state updates
- Memoized selectors for derived state

## Server-Side
- Connection pooling for MongoDB
- Caching of frequently accessed data
- Efficient linked-list traversal
- Batched database operations

## Infrastructure
- Horizontal pod autoscaling (2-10 server, 2-8 client pods)
- Resource limits prevent overconsumption
- CDN-ready static assets
- Multi-platform images for optimal performance

---

# Security Considerations

## Authentication & Authorization
- JWT-based stateless authentication
- bcrypt password hashing with salt rounds
- Passport.js strategy pattern for extensibility

## Data Protection
- Environment variables for sensitive config
- Kubernetes secrets for production credentials
- No secrets committed to version control
- HTTPS/TLS via Let's Encrypt

## API Security
- CORS configuration restricts origins
- Request rate limiting (recommended for production)
- Input validation on all endpoints
- Sanitization of user-generated content

---

This documentation is maintained as part of the project and should be updated when significant architectural or implementation changes occur.
