# Comprehensive System Documentation & Diagrams: AgriCNN

AgriCNN is an advanced serverless, in-browser agricultural deep-learning computer vision platform. This document serves as the complete technical specification, visual system blueprints, database designs, and functional diagrams for scaling the platform from a client-side workbench into an enterprise-grade production architecture.

---

## 1. System Overview & Vision

AgriCNN bridges precision agriculture and client-side machine learning. By utilizing WebGL-accelerated model compilation in the browser, it allows farmers, agronomists, and researchers to instantly diagnose crop leaf diseases offline, maintaining absolute data privacy and eliminating network latencies.

### Key Visual Specs (Supabase Design Guidelines)
- **Primary Color (Emerald)**: `#3ecf8e` - Used exclusively for primary calls-to-action and indicators to establish high visual intent.
- **Surface Canvas**: `#ffffff` - Pure white backgrounds to convey clarity and simplicity.
- **Contrast Canvas (Night)**: `#1c1c1c` - Deep near-black cells reserved for mathematical widgets, code listings, and WebGL shader displays.
- **Typography Scale**: Clean geometric sans-serif (Inter/Circular) running at weight 500 for display layers with tight tracking (`letter-spacing: -0.03em`).

---

## 2. Use Case Diagram

The use case diagram below details the operational boundaries of the AgriCNN platform and how three distinct user personas (Farmers, AI Students, and Guest Users) interact with the functional subsystems.

```mermaid
rect -- "Use Case Boundaries"
  graph LR
    subgraph "AgriCNN System Core"
      UC1["Upload Leaf Photo"]
      UC2["Scan Leaf Preset"]
      UC3["Inspect Visual Feature Maps"]
      UC4["Read Diagnosis & Guidelines"]
      UC5["Adjust Hyperparameters"]
      UC6["Train Custom Model Live"]
      UC7["Browse Pathogen Catalog"]
      UC8["Export Diagnostics Report"]
    end

    Farmer["Farmer / Agronomist"] --> UC1
    Farmer --> UC4
    Farmer --> UC8
    Farmer --> UC7

    Student["AI Research Student"] --> UC2
    Student --> UC3
    Student --> UC5
    Student --> UC6
    Student --> UC7

    Guest["Guest User"] --> UC2
    Guest --> UC4
    Guest --> UC7
```

---

## 3. Network Architecture Diagram

The Convolutional Neural Network (CNN) is structured sequentially in 7 layers. The diagram below tracks the tensor dimensions, kernel operations, and parameter pools as pixel values transition from a raw leaf photo into a disease probability distribution.

```mermaid
graph TD
  subgraph "Preprocessing"
    L0["Input Canvas (128x128x3 RGB)"] -->|Resize & Normalization| L0b["Tensor: [1, 64, 64, 3]"]
  end

  subgraph "Feature Extraction Pipeline (WebGL)"
    L0b -->|3x3 Kernels, Relu| L1["Conv2D Layer 1<br/>[8 Filters, Shape: 62x62x8]"]
    L1 -->|2x2 Max Pooling| L2["MaxPooling Layer 1<br/>[Pool: 2x2, Shape: 31x31x8]"]
    L2 -->|3x3 Kernels, Relu| L3["Conv2D Layer 2<br/>[16 Filters, Shape: 29x29x16]"]
    L3 -->|2x2 Max Pooling| L4["MaxPooling Layer 2<br/>[Pool: 2x2, Shape: 14x14x16]"]
  end

  subgraph "Classification Block"
    L4 -->|Vector Flattening| L5["Flatten Layer<br/>[Vector Size: 3136]"]
    L5 -->|Fully Connected| L6["Dense Layer 1<br/>[32 Neurons, ReLU]"]
    L6 -->|Dropout 20%| L6b["Regularization<br/>[Keep Rate: 0.8]"]
    L6b -->|Softmax Weights| L7["Dense Output Layer<br/>[5 Neurons, Softmax]"]
  end

  subgraph "Inference Resolution"
    L7 -->|ArgMax Vector| L8["Diagnostic Output<br/>P(Disease) > Threshold"]
  end

  style L0 fill:#fafafa,stroke:#3ecf8e,stroke-width:2px
  style L1 fill:#1c1c1c,stroke:#3ecf8e,stroke-width:1px,color:#fff
  style L3 fill:#1c1c1c,stroke:#3ecf8e,stroke-width:1px,color:#fff
  style L7 fill:#3ecf8e,stroke:#24b47e,stroke-width:2px,color:#171717
  style L8 fill:#fafafa,stroke:#171717,stroke-width:1px
```

---

## 4. Production Database Schema (SQL DDL)

To scale AgriCNN into a cloud-integrated production ecosystem, we require a highly normalized relational database to manage user profiles, farms, sensor reports, image files, and model versioning histories. Below is the optimized PostgreSQL DDL schema:

```sql
-- 1. Users Table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    role VARCHAR(30) CHECK (role IN ('Farmer', 'Agronomist', 'Researcher', 'Admin')) DEFAULT 'Farmer',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. Farms Table
CREATE TABLE farms (
    farm_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    name VARCHAR(150) NOT NULL,
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    total_area_hectares DECIMAL(10,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. Crops Table
CREATE TABLE crops (
    crop_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id UUID NOT NULL REFERENCES farms(farm_id) ON DELETE CASCADE,
    crop_type VARCHAR(100) NOT NULL CHECK (crop_type IN ('Tomato', 'Corn', 'Apple', 'Grape', 'Other')),
    variety VARCHAR(100),
    planted_at DATE,
    status VARCHAR(50) DEFAULT 'Active'
);

-- 4. Disease Catalog Table
CREATE TABLE disease_catalog (
    disease_id SERIAL PRIMARY KEY,
    common_name VARCHAR(150) UNIQUE NOT NULL,
    pathogen_scientific VARCHAR(150) NOT NULL,
    class_label INT UNIQUE NOT NULL, -- Corresponds to CNN outputs [0-4]
    severity_level VARCHAR(30) CHECK (severity_level IN ('Low', 'Medium', 'High', 'Critical')),
    organic_treatment TEXT NOT NULL,
    chemical_treatment TEXT NOT NULL
);

-- 5. Scan Reports Table
CREATE TABLE scan_reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE SET NULL,
    crop_id UUID REFERENCES crops(crop_id) ON DELETE CASCADE,
    image_url VARCHAR(512) NOT NULL,
    predicted_class INT NOT NULL REFERENCES disease_catalog(class_label),
    confidence_score DECIMAL(5,2) NOT NULL CHECK (confidence_score BETWEEN 0.00 AND 100.00),
    gps_latitude DECIMAL(9,6),
    gps_longitude DECIMAL(9,6),
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 6. Model Registry Table
CREATE TABLE model_registry (
    model_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version VARCHAR(30) UNIQUE NOT NULL,
    accuracy_metric DECIMAL(5,2) NOT NULL,
    loss_metric DECIMAL(6,4) NOT NULL,
    hyperparameters JSONB NOT NULL, -- Stores Epochs, LR, and Batch configs
    weights_binary_url VARCHAR(512) NOT NULL,
    created_by UUID REFERENCES users(user_id),
    deployed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. Entity Relationship Diagram (ERD)

The Entity Relationship Diagram below details how data models relate to each other, mapping cardinality limits and primary/foreign key connections.

```mermaid
erDiagram
    USERS ||--o{ FARMS : "owns"
    USERS ||--o{ SCAN_REPORTS : "submits"
    FARMS ||--o{ CROPS : "cultivates"
    CROPS ||--o{ SCAN_REPORTS : "logged_for"
    DISEASE_CATALOG ||--o{ SCAN_REPORTS : "diagnoses"
    USERS ||--o{ MODEL_REGISTRY : "trains_and_registers"

    USERS {
        uuid user_id PK
        varchar email UK
        varchar password_hash
        varchar full_name
        varchar role
        timestamp created_at
    }

    FARMS {
        uuid farm_id PK
        uuid owner_id FK
        varchar name
        decimal latitude
        decimal longitude
        decimal total_area_hectares
    }

    CROPS {
        uuid crop_id PK
        uuid farm_id FK
        varchar crop_type
        varchar variety
        date planted_at
        varchar status
    }

    DISEASE_CATALOG {
        int disease_id PK
        varchar common_name UK
        varchar pathogen_scientific
        int class_label UK
        varchar severity_level
        text organic_treatment
        text chemical_treatment
    }

    SCAN_REPORTS {
        uuid report_id PK
        uuid user_id FK
        uuid crop_id FK
        varchar image_url
        int predicted_class FK
        decimal confidence_score
        decimal gps_latitude
        decimal gps_longitude
        text notes
        timestamp created_at
    }

    MODEL_REGISTRY {
        uuid model_id PK
        varchar version UK
        decimal accuracy_metric
        decimal loss_metric
        jsonb hyperparameters
        varchar weights_binary_url
        uuid created_by FK
        timestamp deployed_at
    }
```

---

## 6. Verification & System Extension Guides

> [!TIP]
> **Connecting the Frontend to the PostgreSQL Backend:**
> When migrating AgriCNN from a client-only sandbox into this production environment:
> 1. Create a RESTful or GraphQL API endpoint using Node.js (Express) or Python (FastAPI).
> 2. On upload, standard base64 files from `app.js` can be uploaded to an Amazon S3 bucket, returning an `image_url` to be logged into `scan_reports`.
> 3. Save trained weights from in-browser learning dashboards directly using `await STATE.model.save('http://api.agricnn.com/model/upload')` to store the weight matrix in the model registry.
