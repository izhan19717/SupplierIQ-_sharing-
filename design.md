# Design Document

## Overview

Supplier IQ is a modern, scalable web application built with a microservices architecture to handle supplier risk prediction at enterprise scale. The system integrates advanced ML models with a React-based frontend, FastAPI backend services, and cloud-native infrastructure for optimal performance and scalability.

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[React + TypeScript UI]
        PWA[Progressive Web App]
    end
    
    subgraph "API Gateway"
        GW[Kong/AWS API Gateway]
        AUTH[Authentication Service]
    end
    
    subgraph "Backend Services"
        API[FastAPI Main Service]
        ML[ML Prediction Service]
        FILE[File Processing Service]
        NOTIFY[Notification Service]
    end
    
    subgraph "Data Layer"
        PG[(PostgreSQL)]
        REDIS[(Redis Cache)]
        S3[(S3 Object Storage)]
    end
    
    subgraph "ML Infrastructure"
        MODELS[Model Registry]
        BATCH[Batch Processing]
        QUEUE[Message Queue]
    end
    
    UI --> GW
    PWA --> GW
    GW --> AUTH
    GW --> API
    API --> ML
    API --> FILE
    API --> NOTIFY
    API --> PG
    API --> REDIS
    FILE --> S3
    ML --> MODELS
    ML --> BATCH
    BATCH --> QUEUE
```

### Technology Stack

**Frontend:**
- React 18 with TypeScript for type safety and modern development
- Vite for fast development and optimized builds
- TanStack Query for efficient data fetching and caching
- Plotly.js for advanced data visualizations matching your dashboard
- Tailwind CSS + Headless UI for modern, responsive design
- React Hook Form for efficient form handling

**Backend:**
- FastAPI (Python) for high-performance API development
- Pydantic for data validation and serialization
- SQLAlchemy with async support for database operations
- Celery with Redis for background task processing
- Uvicorn with Gunicorn for production ASGI serving

**Database & Storage:**
- PostgreSQL 15 for primary data storage with JSONB for flexible schemas
- Redis for caching and session management
- AWS S3 (or compatible) for file storage and ML model artifacts

**ML Infrastructure:**
- MLflow for model versioning and registry
- Docker containers for model serving
- Apache Airflow for ML pipeline orchestration
- ONNX Runtime for optimized model inference

**Infrastructure:**
- Docker + Docker Compose for container orchestration
- Simple cloud VM deployment (AWS EC2, GCP Compute, or DigitalOcean)
- GitHub Actions for CI/CD

## Components and Interfaces

### Frontend Components

#### 1. Authentication Module
```typescript
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

interface User {
  id: string;
  email: string;
  name: string;
  organization: string;
  role: UserRole;
}
```

#### 2. File Upload Component
```typescript
interface FileUploadProps {
  onUploadComplete: (result: UploadResult) => void;
  acceptedFormats: string[];
  maxFileSize: number;
}

interface UploadResult {
  fileId: string;
  recordCount: number;
  validationErrors: ValidationError[];
  processingStatus: ProcessingStatus;
}
```

#### 3. Dashboard Components
```typescript
interface DashboardData {
  alertDistribution: AlertDistribution;
  riskMetrics: RiskMetrics;
  temporalTrends: TemporalData[];
  topRiskSuppliers: SupplierRisk[];
  valueAtRisk: number;
}

interface SupplierRisk {
  supplierId: string;
  name: string;
  riskScore: number;
  alertTier: 'Critical' | 'Watch' | 'Normal';
  explanation: string[];
  valueAtRisk: number;
}
```

### Backend API Interfaces

#### 1. Main API Service
```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from typing import List, Optional

class SupplierData(BaseModel):
    vendor_name: str
    material: str
    delivery_date: datetime
    open_amount: float
    lead_time_days: int

class RiskPrediction(BaseModel):
    supplier_id: str
    risk_score: float
    alert_tier: str
    explanation: List[str]
    confidence_interval: Tuple[float, float]

class PredictionRequest(BaseModel):
    data: List[SupplierData]
    model_version: Optional[str] = "latest"
```

#### 2. ML Service Interface
```python
class MLPredictionService:
    async def predict_batch(
        self, 
        data: List[SupplierData],
        model_version: str = "latest"
    ) -> List[RiskPrediction]:
        """Generate risk predictions for batch of suppliers"""
        
    async def explain_prediction(
        self, 
        supplier_data: SupplierData,
        prediction: RiskPrediction
    ) -> ExplanationResult:
        """Generate SHAP explanations for prediction"""
        
    async def get_model_metrics(
        self, 
        model_version: str
    ) -> ModelMetrics:
        """Get model performance metrics"""
```

### Database Schema - Flexible Design

#### Core Tables with Adaptive Schema
```sql
-- Users and Organizations
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    organization_id UUID REFERENCES organizations(id),
    role VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Data Schema Registry (tracks what columns each organization has)
CREATE TABLE data_schemas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    schema_name VARCHAR(255) NOT NULL,
    column_mappings JSONB NOT NULL, -- Maps customer columns to standard fields
    required_fields JSONB NOT NULL, -- List of fields needed for ML
    optional_fields JSONB NOT NULL, -- Additional fields available
    created_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

-- Flexible Supplier Data Storage
CREATE TABLE suppliers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    external_id VARCHAR(255), -- Customer's supplier ID
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE supplier_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    supplier_id UUID REFERENCES suppliers(id),
    schema_id UUID REFERENCES data_schemas(id),
    
    -- Core standardized fields (always extracted)
    purchase_document VARCHAR(255),
    delivery_date DATE,
    purchase_date DATE,
    supplier_name VARCHAR(255),
    material VARCHAR(255),
    
    -- Flexible numeric fields (extracted when available)
    open_amount DECIMAL(15,2),
    open_quantity DECIMAL(15,2),
    delayed_days INTEGER,
    reschedule_count INTEGER,
    lead_time_days INTEGER,
    
    -- Complete raw data storage
    raw_data JSONB NOT NULL, -- Original CSV row
    standardized_data JSONB, -- Processed/cleaned data
    extracted_features JSONB, -- ML features computed from available data
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- Predictions and Risk Scores
CREATE TABLE risk_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id UUID REFERENCES suppliers(id),
    order_id UUID REFERENCES supplier_orders(id),
    model_version VARCHAR(50),
    risk_score DECIMAL(5,4),
    alert_tier VARCHAR(20),
    explanation JSONB,
    confidence_lower DECIMAL(5,4),
    confidence_upper DECIMAL(5,4),
    feature_importance JSONB, -- Which features drove this prediction
    data_quality_score DECIMAL(3,2), -- How complete was the input data (0-1)
    created_at TIMESTAMP DEFAULT NOW()
);

-- Column Mapping Templates (pre-built mappings for common ERP systems)
CREATE TABLE column_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_name VARCHAR(255) NOT NULL, -- "SAP_Standard", "Oracle_ERP", etc.
    description TEXT,
    column_mappings JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_supplier_orders_org_date ON supplier_orders(organization_id, delivery_date);
CREATE INDEX idx_supplier_orders_raw_data ON supplier_orders USING GIN (raw_data);
CREATE INDEX idx_risk_predictions_supplier ON risk_predictions(supplier_id, created_at);
CREATE INDEX idx_risk_predictions_alert_tier ON risk_predictions(alert_tier, risk_score);
CREATE INDEX idx_data_schemas_org ON data_schemas(organization_id, is_active);
```

#### Adaptive Data Processing Architecture

```python
class AdaptiveDataProcessor:
    """Handles varying data schemas from different customers"""
    
    def __init__(self):
        self.core_fields = {
            'supplier_name': ['vendor', 'vendor_name', 'supplier', 'vendor name'],
            'delivery_date': ['delivery_date', '1st delivery date', 'planned_delivery', 'due_date'],
            'purchase_date': ['purchase_date', 'pur_doc_date', 'po_date', 'order_date'],
            'material': ['material', 'part_number', 'item', 'product'],
            'purchase_document': ['po_number', 'purchase_document', 'purchasing_document'],
            'open_amount': ['open_amount', 'po_value', 'order_value', 'amount'],
            'delayed_days': ['delayed_days', 'delay', 'days_late']
        }
        
        self.optional_fields = {
            'reschedule_count': ['reschedule', 'no_of_rescheduling', 'changes'],
            'lead_time': ['lead_time', 'lead_time_days'],
            'plant': ['plant', 'plant_code', 'location'],
            'product_line': ['product_line', 'category', 'division']
        }
    
    async def analyze_schema(self, df: pd.DataFrame, organization_id: str) -> Dict:
        """Analyze uploaded CSV and create column mapping"""
        detected_mappings = {}
        confidence_scores = {}
        
        # Smart column detection using fuzzy matching
        for standard_field, possible_names in self.core_fields.items():
            best_match = self._find_best_column_match(df.columns, possible_names)
            if best_match:
                detected_mappings[standard_field] = best_match['column']
                confidence_scores[standard_field] = best_match['confidence']
        
        # Detect optional fields
        for optional_field, possible_names in self.optional_fields.items():
            best_match = self._find_best_column_match(df.columns, possible_names)
            if best_match and best_match['confidence'] > 0.7:
                detected_mappings[optional_field] = best_match['column']
                confidence_scores[optional_field] = best_match['confidence']
        
        return {
            'mappings': detected_mappings,
            'confidence': confidence_scores,
            'unmapped_columns': [col for col in df.columns if col not in detected_mappings.values()],
            'data_completeness': self._assess_data_completeness(df, detected_mappings)
        }
    
    def _find_best_column_match(self, columns: List[str], possible_names: List[str]) -> Optional[Dict]:
        """Find best matching column using fuzzy string matching"""
        from difflib import SequenceMatcher
        
        best_match = None
        best_score = 0
        
        for col in columns:
            for possible_name in possible_names:
                # Exact match (case insensitive)
                if col.lower().strip() == possible_name.lower().strip():
                    return {'column': col, 'confidence': 1.0}
                
                # Fuzzy match
                score = SequenceMatcher(None, col.lower(), possible_name.lower()).ratio()
                if score > best_score and score > 0.6:
                    best_score = score
                    best_match = {'column': col, 'confidence': score}
        
        return best_match
    
    async def create_adaptive_features(self, df: pd.DataFrame, mappings: Dict) -> pd.DataFrame:
        """Create ML features from whatever data is available"""
        features_df = df.copy()
        
        # Always try to create these core features if data exists
        if 'supplier_name' in mappings:
            features_df = self._add_supplier_features(features_df, mappings['supplier_name'])
        
        if 'delivery_date' in mappings and 'purchase_date' in mappings:
            features_df = self._add_temporal_features(features_df, mappings)
        
        if 'open_amount' in mappings:
            features_df = self._add_value_features(features_df, mappings['open_amount'])
        
        # Add features based on available optional fields
        available_features = []
        if 'reschedule_count' in mappings:
            features_df['has_reschedule'] = features_df[mappings['reschedule_count']] > 0
            available_features.append('reschedule_tracking')
        
        if 'lead_time' in mappings:
            features_df['is_rush_order'] = features_df[mappings['lead_time']] < 30
            available_features.append('lead_time_analysis')
        
        # Store metadata about what features were created
        features_df['_available_features'] = [available_features] * len(features_df)
        features_df['_data_completeness'] = self._calculate_row_completeness(features_df, mappings)
        
        return features_df
```

## Data Models

### ML Pipeline Integration

#### 1. Feature Engineering Service
```python
class FeatureEngineer:
    def __init__(self):
        self.feature_pipeline = self._build_pipeline()
    
    def _build_pipeline(self):
        """Build feature engineering pipeline matching your existing code"""
        return Pipeline([
            ('temporal_features', TemporalFeatureTransformer()),
            ('historical_performance', HistoricalPerformanceTransformer()),
            ('interaction_features', InteractionFeatureTransformer()),
            ('volatility_features', VolatilityFeatureTransformer())
        ])
    
    async def transform(self, raw_data: pd.DataFrame) -> pd.DataFrame:
        """Transform raw supplier data into ML features"""
        return self.feature_pipeline.transform(raw_data)
```

#### 2. Adaptive Model Serving Architecture
```python
class AdaptiveEnsembleModelService:
    def __init__(self):
        # Multiple model variants for different data completeness levels
        self.models = {
            'full_feature': {  # For data with 80%+ completeness
                'xgb': self._load_model('xgboost_full'),
                'lgb': self._load_model('lightgbm_full'), 
                'cat': self._load_model('catboost_full')
            },
            'core_feature': {  # For data with 50-80% completeness
                'xgb': self._load_model('xgboost_core'),
                'lgb': self._load_model('lightgbm_core'),
                'cat': self._load_model('catboost_core')
            },
            'minimal_feature': {  # For data with <50% completeness
                'xgb': self._load_model('xgboost_minimal'),
                'lgb': self._load_model('lightgbm_minimal')
            }
        }
        
        self.calibrators = {
            'full_feature': self._load_calibrator('full'),
            'core_feature': self._load_calibrator('core'),
            'minimal_feature': self._load_calibrator('minimal')
        }
        
        # Feature importance for different model variants
        self.feature_sets = {
            'full_feature': ['vendor_hist_fail_rate', 'material_hist_fail_rate', 'lead_time_days', 
                           'vendor_delay_std', 'reschedule_count', 'order_value_log', 'delivery_month'],
            'core_feature': ['vendor_hist_fail_rate', 'lead_time_days', 'delivery_month', 'order_value_log'],
            'minimal_feature': ['vendor_name_enc', 'delivery_month', 'lead_time_days']
        }
        
    async def predict_adaptive(self, features_df: pd.DataFrame, data_completeness: float) -> Dict:
        """Generate predictions based on available data quality"""
        
        # Select model variant based on data completeness
        if data_completeness >= 0.8:
            model_variant = 'full_feature'
        elif data_completeness >= 0.5:
            model_variant = 'core_feature'
        else:
            model_variant = 'minimal_feature'
        
        # Extract available features for this variant
        available_features = []
        for feature in self.feature_sets[model_variant]:
            if feature in features_df.columns:
                available_features.append(feature)
        
        if len(available_features) < 3:  # Minimum viable features
            return self._fallback_prediction(features_df)
        
        # Prepare feature matrix
        X = features_df[available_features].fillna(-999)
        
        # Generate ensemble predictions
        predictions = []
        model_weights = []
        
        for model_name, model in self.models[model_variant].items():
            try:
                pred = model.predict_proba(X)[:, 1]
                predictions.append(pred)
                model_weights.append(1.0)  # Equal weighting, could be optimized
            except Exception as e:
                print(f"Model {model_name} failed: {e}")
                continue
        
        if not predictions:
            return self._fallback_prediction(features_df)
        
        # Weighted ensemble
        ensemble_pred = np.average(predictions, axis=0, weights=model_weights)
        
        # Apply calibration
        calibrated_pred = self.calibrators[model_variant].predict(ensemble_pred)
        
        # Adjust confidence based on data quality
        confidence_adjustment = min(1.0, data_completeness + 0.2)
        adjusted_confidence = calibrated_pred * confidence_adjustment
        
        return {
            'risk_scores': adjusted_confidence,
            'raw_scores': ensemble_pred,
            'model_variant_used': model_variant,
            'data_completeness': data_completeness,
            'features_used': available_features,
            'confidence_level': 'high' if data_completeness > 0.7 else 'medium' if data_completeness > 0.4 else 'low'
        }
    
    def _fallback_prediction(self, features_df: pd.DataFrame) -> Dict:
        """Simple rule-based prediction when ML models can't be used"""
        # Basic heuristics based on available data
        risk_scores = np.full(len(features_df), 0.3)  # Default medium-low risk
        
        # Adjust based on any available signals
        if 'reschedule_count' in features_df.columns:
            high_reschedule = features_df['reschedule_count'] > 2
            risk_scores[high_reschedule] = 0.7
        
        if 'delayed_days' in features_df.columns:
            previously_delayed = features_df['delayed_days'] > 0
            risk_scores[previously_delayed] = 0.6
        
        return {
            'risk_scores': risk_scores,
            'raw_scores': risk_scores,
            'model_variant_used': 'fallback_rules',
            'data_completeness': 0.0,
            'features_used': [],
            'confidence_level': 'low'
        }

class SmartColumnMapper:
    """Intelligent column mapping with learning capabilities"""
    
    def __init__(self):
        self.common_mappings = {
            # SAP mappings
            'sap': {
                'Vendor': 'supplier_name',
                'Purchasing Document': 'purchase_document',
                'Delivery Date': 'delivery_date',
                'PO Date': 'purchase_date',
                'Material': 'material',
                'Net Value': 'open_amount'
            },
            # Oracle mappings
            'oracle': {
                'SUPPLIER_NAME': 'supplier_name',
                'PO_NUMBER': 'purchase_document',
                'PROMISED_DATE': 'delivery_date',
                'CREATION_DATE': 'purchase_date',
                'ITEM_DESCRIPTION': 'material',
                'AMOUNT': 'open_amount'
            },
            # Generic mappings
            'generic': {
                'vendor': 'supplier_name',
                'supplier': 'supplier_name',
                'po': 'purchase_document',
                'purchase_order': 'purchase_document',
                'delivery': 'delivery_date',
                'due_date': 'delivery_date',
                'amount': 'open_amount',
                'value': 'open_amount'
            }
        }
    
    async def suggest_mappings(self, columns: List[str]) -> Dict:
        """Suggest column mappings with confidence scores"""
        suggestions = {}
        
        for col in columns:
            best_match = self._find_best_standard_field(col)
            if best_match:
                suggestions[col] = best_match
        
        return suggestions
    
    def _find_best_standard_field(self, column_name: str) -> Optional[Dict]:
        """Find the best standard field match for a column"""
        col_lower = column_name.lower().strip()
        
        # Check all mapping templates
        for template_name, mappings in self.common_mappings.items():
            for template_col, standard_field in mappings.items():
                if col_lower == template_col.lower():
                    return {
                        'standard_field': standard_field,
                        'confidence': 1.0,
                        'template': template_name
                    }
                
                # Fuzzy matching
                from difflib import SequenceMatcher
                similarity = SequenceMatcher(None, col_lower, template_col.lower()).ratio()
                if similarity > 0.8:
                    return {
                        'standard_field': standard_field,
                        'confidence': similarity,
                        'template': template_name
                    }
        
        return None
```

### Data Processing Pipeline

#### 1. CSV Processing Workflow
```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant FileService
    participant MLService
    participant Database
    
    User->>Frontend: Upload CSV
    Frontend->>API: POST /upload
    API->>FileService: Process CSV
    FileService->>FileService: Validate & Clean Data
    FileService->>Database: Store Raw Data
    FileService->>MLService: Trigger Prediction
    MLService->>MLService: Feature Engineering
    MLService->>MLService: Generate Predictions
    MLService->>Database: Store Predictions
    MLService->>API: Prediction Complete
    API->>Frontend: Upload Success
    Frontend->>User: Show Results
```

#### 2. Real-time Processing
```python
@celery.task
async def process_uploaded_file(file_id: str, organization_id: str):
    """Background task for processing uploaded CSV files"""
    try:
        # Load and validate CSV
        raw_data = await load_csv_from_storage(file_id)
        validated_data = await validate_supplier_data(raw_data)
        
        # Store in database
        await store_supplier_orders(validated_data, organization_id)
        
        # Trigger ML predictions
        await trigger_ml_predictions(validated_data, organization_id)
        
        # Send completion notification
        await send_processing_notification(organization_id, "success")
        
    except Exception as e:
        await send_processing_notification(organization_id, "error", str(e))
        raise
```

## Error Handling

### API Error Responses
```python
class APIError(BaseModel):
    error_code: str
    message: str
    details: Optional[Dict] = None
    timestamp: datetime

class ErrorHandler:
    @staticmethod
    def validation_error(details: Dict) -> HTTPException:
        return HTTPException(
            status_code=422,
            detail=APIError(
                error_code="VALIDATION_ERROR",
                message="Data validation failed",
                details=details,
                timestamp=datetime.utcnow()
            ).dict()
        )
    
    @staticmethod
    def ml_prediction_error(error: str) -> HTTPException:
        return HTTPException(
            status_code=500,
            detail=APIError(
                error_code="ML_PREDICTION_ERROR", 
                message="Model prediction failed",
                details={"error": error},
                timestamp=datetime.utcnow()
            ).dict()
        )
```

### Frontend Error Boundaries
```typescript
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log to monitoring service
    logError(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

## Testing Strategy

### Backend Testing
```python
# Unit Tests
class TestMLPredictionService:
    @pytest.fixture
    def sample_data(self):
        return [
            SupplierData(
                vendor_name="Test Vendor",
                material="Test Material", 
                delivery_date=datetime.now(),
                open_amount=10000.0,
                lead_time_days=30
            )
        ]
    
    async def test_batch_prediction(self, sample_data):
        service = MLPredictionService()
        predictions = await service.predict_batch(sample_data)
        
        assert len(predictions) == len(sample_data)
        assert all(0 <= p.risk_score <= 1 for p in predictions)
        assert all(p.alert_tier in ['Critical', 'Watch', 'Normal'] for p in predictions)

# Integration Tests  
class TestFileUploadFlow:
    async def test_csv_upload_and_prediction(self, test_client, sample_csv):
        # Upload CSV
        response = await test_client.post(
            "/upload",
            files={"file": sample_csv}
        )
        assert response.status_code == 200
        
        # Wait for processing
        file_id = response.json()["file_id"]
        await wait_for_processing(file_id)
        
        # Check predictions were generated
        predictions = await test_client.get(f"/predictions/{file_id}")
        assert predictions.status_code == 200
        assert len(predictions.json()) > 0
```

### Frontend Testing
```typescript
// Component Tests
describe('FileUpload Component', () => {
  test('handles successful file upload', async () => {
    const mockOnComplete = jest.fn();
    render(<FileUpload onUploadComplete={mockOnComplete} />);
    
    const file = new File(['test,data'], 'test.csv', { type: 'text/csv' });
    const input = screen.getByLabelText(/upload file/i);
    
    await userEvent.upload(input, file);
    
    await waitFor(() => {
      expect(mockOnComplete).toHaveBeenCalledWith(
        expect.objectContaining({
          recordCount: expect.any(Number),
          processingStatus: 'completed'
        })
      );
    });
  });
});

// E2E Tests
describe('Supplier Risk Prediction Flow', () => {
  test('complete user journey', async () => {
    await page.goto('/login');
    await page.fill('[data-testid=email]', 'test@example.com');
    await page.fill('[data-testid=password]', 'password');
    await page.click('[data-testid=login-button]');
    
    await page.goto('/upload');
    await page.setInputFiles('[data-testid=file-input]', 'test-data.csv');
    await page.click('[data-testid=upload-button]');
    
    await page.waitForSelector('[data-testid=upload-success]');
    await page.goto('/dashboard');
    
    await expect(page.locator('[data-testid=risk-chart]')).toBeVisible();
    await expect(page.locator('[data-testid=alert-count]')).toContainText(/\d+/);
  });
});
```

## Deployment Architecture

### Container Strategy
```dockerfile
# Multi-stage build for frontend
FROM node:18-alpine AS frontend-build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM nginx:alpine AS frontend
COPY --from=frontend-build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# Backend API container
FROM python:3.11-slim AS backend
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker"]

# ML Service container
FROM python:3.11-slim AS ml-service
WORKDIR /app
COPY ml-requirements.txt .
RUN pip install --no-cache-dir -r ml-requirements.txt
COPY ml_service/ .
CMD ["python", "ml_server.py"]
```

### Docker Compose Deployment
```yaml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - api
    environment:
      - API_URL=http://api:8000
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/supplieriq
      - REDIS_URL=redis://redis:6379
      - ML_SERVICE_URL=http://ml-service:8001
    volumes:
      - ./uploads:/app/uploads

  ml-service:
    build:
      context: ./ml_service
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - MODEL_PATH=/app/models
    volumes:
      - ./models:/app/models:ro

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=supplieriq
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  celery-worker:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A app.celery worker --loglevel=info
    depends_on:
      - redis
      - postgres
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/supplieriq
      - REDIS_URL=redis://redis:6379

volumes:
  postgres_data:
  redis_data:
```

### Infrastructure as Code
```hcl
# Simple Terraform for cloud VM deployment
resource "aws_instance" "supplier_iq" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04
  instance_type = "t3.large"  # 2 vCPU, 8GB RAM
  
  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y docker.io docker-compose
              systemctl start docker
              systemctl enable docker
              EOF
  
  tags = {
    Name = "supplier-iq-app"
  }
}
```

This design provides a robust, scalable foundation for your Supplier IQ platform that can handle your current CSV upload needs while being ready for future ERP integrations. The architecture leverages modern cloud-native patterns and integrates seamlessly with your existing ML pipeline. db-secret
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi" 
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Infrastructure as Code
```hcl
# Terraform configuration for AWS
resource "aws_eks_cluster" "supplier_iq" {
  name     = "supplier-iq-cluster"
  role_arn = aws_iam_role.cluster.arn
  version  = "1.27"

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}

resource "aws_rds_instance" "postgres" {
  identifier = "supplier-iq-db"
  engine     = "postgres"
  engine_version = "15.3"
  instance_class = "db.r6g.large"
  allocated_storage = 100
  storage_encrypted = true
  
  db_name  = "supplier_iq"
  username = var.db_username
  password = var.db_password
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "supplier-iq-cache"
  engine               = "redis"
  node_type            = "cache.r6g.large"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379
}
```

This design provides a robust, scalable foundation for your Supplier IQ platform that can handle your current CSV upload needs while being ready for future ERP integrations. The architecture leverages modern cloud-native patterns and integrates seamlessly with your existing ML pipeline.