# Database Schema Design

## Overview
This document outlines the complete database schema for the Policy Management System. The schema is designed to support policy management, versioning, acknowledgments, and audit requirements for SOC 2 compliance.

## Product Requirements

1. Policy Types and Templates
   - Companies require multiple policy types for SOC 2 compliance (Infosec, Acceptable Use, Cryptographic, etc.)
   - Sprinto provides default policy templates for each type
   - Templates are internal tools to generate customer policies
   - Customers can use their own policies instead of template-generated ones

2. Policy Approval Process
   - New policies require approval from designated person (e.g. CTO)
   - Approved policies need employee acknowledgment via checkbox confirmation

3. Employee Acknowledgment Requirements
   - New employees must acknowledge all policies within 30 days of joining
   - All employees must periodically acknowledge policies (minimum annually)
   - Historical acknowledgment records must be maintained for audit purposes

4. Acknowledgment Tracking and Alerts
   - System must support acknowledgment requests for new and existing employees
   - Track if acknowledgments completed within 30 day window
   - Escalate overdue acknowledgments to company CXO level

5. Policy Template Updates
   - When Sprinto updates templates, customers receive upgrade prompts
   - Current active policies remain functional until upgrade
   - Policy configurations (e.g. vulnerability management SLAs) are customer-defined
   - Configuration changes require new approval while previous version remains active

6. Policy Modification Handling
   - No strict requirement for re-acknowledgment after policy changes
   - Data model should support selective re-acknowledgment if needed
   - Customers can request acknowledgment for only modified policies

7. Acknowledgment Types and Tracking
   - System distinguishes between periodic and new hire acknowledgments
   - Supports manual triggering of acknowledgments for selected employees
   - Data model tracks acknowledgment trigger type

8. Role-Based Policy Requirements
   - Different roles require different policy acknowledgments
   - Example: HR may need 3 policies while Engineering needs 15
   - Policy requirements per role should be configurable

## Entity Relationship Diagram 

![ER Diagram](assets/er-diagram.png)



```mermaid
erDiagram
    Company ||--o{ Employee : has
    Company ||--o{ Policy : owns
    Company ||--o{ Role : defines
    Company ||--o{ PolicyApprovalWorkflow : configures
    Company ||--o{ SLAConfiguration : configures

    Role ||--o{ Employee : assigned_to
    Role ||--o{ RolePolicyMapping : requires
    Role ||--o{ PolicyApprovalWorkflow : approves_as

    PolicyType ||--o{ PolicyTemplate : categorizes
    PolicyType ||--o{ Policy : typed_as
    PolicyType ||--o{ PolicyApprovalWorkflow : requires_approval

    PolicyTemplate ||--o{ PolicyTemplateVersion : versions
    PolicyTemplate ||--o{ Policy : generates
    PolicyTemplateVersion ||--o{ Policy : used_by

    Policy ||--o{ PolicyVersion : has
    PolicyVersion ||--|| PolicyConfiguration : configures
    PolicyVersion ||--o{ PolicyApproval : requires
    PolicyVersion ||--o{ AcknowledgmentRequest : requires

    PolicyConfiguration ||--o{ PolicyConfigurationHistory : tracks
    
    Employee ||--o{ PolicyAcknowledgment : makes
    Employee ||--o{ PolicyApproval : approves
    Employee ||--o{ PolicyAuditLog : creates

    AcknowledgmentCampaign ||--o{ AcknowledgmentRequest : contains
    AcknowledgmentRequest ||--o{ PolicyAcknowledgment : generates

    SLAConfiguration ||--o{ SLAEscalationMatrix : defines

    Company {
        UUID id PK
        string name
        string industry
        string compliance_level
        string status
    }

    Policy {
        UUID id PK
        UUID company_id FK
        UUID policy_type_id FK
        UUID template_id FK
        UUID current_version_id FK
        string title
        boolean is_custom
        string status
    }

    PolicyVersion {
        UUID id PK
        UUID policy_id FK
        text content
        integer version_number
        string status
        boolean is_active
    }

    PolicyApprovalWorkflow {
        UUID id PK
        UUID company_id FK
        UUID policy_type_id FK
        integer approval_sequence
        UUID approver_role FK
    }
```

## System Flow 

![System Flow](assets/flows.png)

```mermaid
flowchart TD
    subgraph Template Management Flow
        direction LR
        PT[PolicyType] --> |1| TEM[PolicyTemplate]
        TEM --> |2| TV[PolicyTemplateVersion]
        TV --> |3| P[Policy]
        P --> |4| PV[PolicyVersion]
        
        subgraph Associated Tables
            PT_audit[PolicyAuditLog]
            TV_usage[PolicyTemplateVersionUsage]
        end
        
        TV --> TV_usage
        P --> PT_audit
    end

    subgraph Policy Creation and Approval Flow
        direction LR
        POL[Policy] --> |1| PV2[PolicyVersion]
        PV2 --> |2| PC[PolicyConfiguration]
        PV2 --> |3| PAW[PolicyApprovalWorkflow]
        PAW --> |4| PA[PolicyApproval]
        
        subgraph Audit Tracking
            P_audit[PolicyAuditLog]
            C_audit[ConfigurationAuditLog]
        end
        
        PV2 --> P_audit
        PC --> C_audit
        
        subgraph Lookup Tables
            EMP[Employee]
            ROLE[Role]
        end
        
        ROLE --> PAW
        EMP --> PA
    end

    subgraph Acknowledgment Flow
        direction LR
        AP[Active Policy] --> |1| AC[AcknowledgmentCampaign]
        AC --> |2| AR[AcknowledgmentRequest]
        AR --> |3| PA2[PolicyAcknowledgment]
        AR --> |4| SLA[SLAConfiguration]
        SLA --> |5| EM[SLAEscalationMatrix]
        
        subgraph Related Tables
            EMP2[Employee]
            ROLE2[Role]
            RPM[RolePolicyMapping]
        end
        
        ROLE2 --> RPM
        RPM --> AR
        EMP2 --> PA2
    end

    subgraph Configuration Management Flow
        direction LR
        POL2[Policy] --> |1| PV3[PolicyVersion]
        PV3 --> |2| PC2[PolicyConfiguration]
        PC2 --> |3| PCH[PolicyConfigurationHistory]
        
        subgraph Audit Tables
            C_audit2[ConfigurationAuditLog]
        end
        
        PC2 --> C_audit2
        PCH --> C_audit2
    end
```

## Table Definitions

### Core Tables

```sql
-- Company and Organization
CREATE TABLE Company (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    industry VARCHAR(100),
    compliance_level VARCHAR(50),
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE Employee (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    role_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    employee_id VARCHAR(100),
    join_date DATE NOT NULL,
    status VARCHAR(50) DEFAULT 'active',
    is_policy_approver BOOLEAN DEFAULT false,
    manager_id UUID REFERENCES Employee(id),
    department VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

-- Policy Management
CREATE TABLE PolicyType (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    is_mandatory BOOLEAN DEFAULT false,
    compliance_framework VARCHAR(100),
    review_frequency INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE PolicyTemplate (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_type_id UUID REFERENCES PolicyType(id),
    version INTEGER NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    placeholders JSONB,
    is_latest BOOLEAN DEFAULT false,
    status VARCHAR(50) DEFAULT 'active',
    effective_from DATE,
    effective_until DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE Policy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    policy_type_id UUID REFERENCES PolicyType(id),
    template_id UUID REFERENCES PolicyTemplate(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    is_custom BOOLEAN DEFAULT false,
    current_version_id UUID,
    review_frequency INTEGER,
    next_review_date DATE,
    status VARCHAR(50) DEFAULT 'draft',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE PolicyVersion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID REFERENCES Policy(id),
    content TEXT NOT NULL,
    version_number INTEGER NOT NULL,
    changelog TEXT,
    status VARCHAR(50) DEFAULT 'draft',
    is_active BOOLEAN DEFAULT false,
    approved_at TIMESTAMP,
    effective_from DATE,
    effective_until DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by UUID REFERENCES Employee(id)
);

CREATE TABLE PolicyTemplateVersion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID REFERENCES PolicyTemplate(id),
    version_number INTEGER NOT NULL,
    content TEXT NOT NULL,
    placeholders JSONB,
    changelog TEXT,
    status VARCHAR(50) DEFAULT 'draft',
    published_at TIMESTAMP,
    effective_from TIMESTAMP,
    effective_until TIMESTAMP,
    created_by UUID REFERENCES Employee(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_template_version 
        UNIQUE (template_id, version_number)
);

-- Add reference in PolicyTemplate
ALTER TABLE PolicyTemplate 
    ADD COLUMN current_version_id UUID REFERENCES PolicyTemplateVersion(id);

CREATE TABLE PolicyApprovalWorkflow (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    policy_type_id UUID REFERENCES PolicyType(id),
    approval_sequence INTEGER NOT NULL,
    approver_role UUID REFERENCES Role(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_approval_workflow 
        UNIQUE (company_id, policy_type_id, approval_sequence)
);

CREATE TABLE PolicyApproval (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_version_id UUID REFERENCES PolicyVersion(id),
    approver_id UUID REFERENCES Employee(id),
    approval_sequence INTEGER NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    comments TEXT,
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_version_approval 
        UNIQUE (policy_version_id, approval_sequence)
);

CREATE TABLE PolicyTemplateVersion (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID REFERENCES PolicyTemplate(id),
    version_number INTEGER NOT NULL,
    content TEXT NOT NULL,
    placeholders JSONB,
    changelog TEXT,
    status VARCHAR(50) DEFAULT 'draft',
    published_at TIMESTAMP,
    effective_from TIMESTAMP,
    effective_until TIMESTAMP,
    created_by UUID REFERENCES Employee(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_template_version 
        UNIQUE (template_id, version_number)
);

-- Add reference in PolicyTemplate
ALTER TABLE PolicyTemplate 
    ADD COLUMN current_version_id UUID REFERENCES PolicyTemplateVersion(id);

CREATE TABLE PolicyAcknowledgment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id UUID REFERENCES AcknowledgmentRequest(id),
    employee_id UUID REFERENCES Employee(id),
    policy_version_id UUID REFERENCES PolicyVersion(id),
    acknowledged_at TIMESTAMP NOT NULL,
    acknowledgment_type VARCHAR(50) NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Acknowledgment Management
CREATE TABLE AcknowledgmentCampaign (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    description TEXT,
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP NOT NULL,
    grace_period_days INTEGER DEFAULT 30,
    reminder_frequency INTEGER,
    max_reminders INTEGER,
    status VARCHAR(50) DEFAULT 'draft',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by UUID REFERENCES Employee(id)
);

CREATE TABLE AcknowledgmentRequest (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID REFERENCES AcknowledgmentCampaign(id),
    employee_id UUID REFERENCES Employee(id),
    policy_version_id UUID REFERENCES PolicyVersion(id),
    due_date TIMESTAMP NOT NULL,
    completion_date TIMESTAMP,
    status VARCHAR(50) DEFAULT 'pending',
    reminder_count INTEGER DEFAULT 0,
    last_reminder_sent TIMESTAMP,
    sla_breach_date TIMESTAMP,
    is_escalated BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE PolicyConfiguration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_version_id UUID REFERENCES PolicyVersion(id),
    config_schema JSONB NOT NULL,
    config_data JSONB NOT NULL,
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE PolicyConfigurationHistory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    configuration_id UUID REFERENCES PolicyConfiguration(id),
    previous_config JSONB,
    new_config JSONB,
    changed_by UUID REFERENCES Employee(id),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    reason TEXT
);

-- Audit Tables
CREATE TABLE PolicyAuditLog (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID REFERENCES Policy(id),
    version_id UUID REFERENCES PolicyVersion(id),
    action_type VARCHAR(50),
    action_by UUID REFERENCES Employee(id),
    action_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    previous_state JSONB,
    new_state JSONB,
    reason TEXT,
    ip_address VARCHAR(45),
    user_agent TEXT
);

CREATE TABLE ConfigurationAuditLog (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID REFERENCES Policy(id),
    configuration_id UUID REFERENCES PolicyConfiguration(id),
    changed_by UUID REFERENCES Employee(id),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    previous_config JSONB,
    new_config JSONB,
    reason TEXT
);

-- SLA Management
CREATE TABLE SLAConfiguration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    acknowledgment_type VARCHAR(50),
    sla_days INTEGER NOT NULL,
    warning_threshold INTEGER,
    escalation_threshold INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE SLAEscalationMatrix (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sla_config_id UUID REFERENCES SLAConfiguration(id),
    escalation_level INTEGER,
    escalate_to_role VARCHAR(50),
    notification_method VARCHAR(50)[],
    wait_hours INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Indexes and Constraints

```sql
-- Indexes for frequently accessed columns
CREATE INDEX idx_policy_acknowledgment_employee ON PolicyAcknowledgment(employee_id, policy_version_id);
CREATE INDEX idx_acknowledgment_request_status ON AcknowledgmentRequest(campaign_id, status);
CREATE INDEX idx_policy_version_active ON PolicyVersion(policy_id, is_active);
CREATE INDEX idx_employee_company ON Employee(company_id);
CREATE INDEX idx_policy_company ON Policy(company_id);
CREATE INDEX idx_audit_log_policy ON PolicyAuditLog(policy_id);

-- Constraints
ALTER TABLE PolicyVersion 
    ADD CONSTRAINT single_active_version 
    EXCLUDE USING gist (policy_id WITH =, is_active WITH =) 
    WHERE (is_active = true);

ALTER TABLE PolicyAcknowledgment 
    ADD CONSTRAINT valid_acknowledgment_type
    CHECK (acknowledgment_type IN ('new_hire', 'periodic', 'manual'));

CREATE TABLE Role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES Company(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_role_per_company 
        UNIQUE (company_id, name)
);

CREATE TABLE RolePolicyMapping (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID REFERENCES Role(id),
    policy_id UUID REFERENCES Policy(id),
    is_mandatory BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT unique_role_policy 
        UNIQUE (role_id, policy_id)
);

```

## Constraints

```sql

-- Unique Constraints
ALTER TABLE PolicyVersion 
    ADD CONSTRAINT unique_version_number 
    UNIQUE (policy_id, version_number);

ALTER TABLE PolicyTemplateVersion 
    ADD CONSTRAINT unique_template_version 
    UNIQUE (template_id, version_number);

ALTER TABLE PolicyApprovalWorkflow 
    ADD CONSTRAINT unique_approval_sequence 
    UNIQUE (company_id, policy_type_id, approval_sequence);

ALTER TABLE RolePolicyMapping 
    ADD CONSTRAINT unique_role_policy 
    UNIQUE (role_id, policy_id);

ALTER TABLE PolicyAcknowledgment 
    ADD CONSTRAINT unique_acknowledgment 
    UNIQUE (employee_id, policy_version_id, request_id);

-- Check Constraints
ALTER TABLE PolicyVersion 
    ADD CONSTRAINT valid_version_status 
    CHECK (status IN ('draft', 'pending_approval', 'approved', 'active', 'archived'));

ALTER TABLE PolicyAcknowledgment 
    ADD CONSTRAINT valid_acknowledgment_type 
    CHECK (acknowledgment_type IN ('new_hire', 'periodic', 'manual'));

ALTER TABLE AcknowledgmentCampaign 
    ADD CONSTRAINT valid_campaign_type 
    CHECK (type IN ('new_hire', 'periodic', 'policy_update', 'manual'));

-- Important Indexes
CREATE INDEX idx_policy_version_status ON PolicyVersion(policy_id, status, is_active);
CREATE INDEX idx_acknowledgment_employee ON PolicyAcknowledgment(employee_id, acknowledged_at);
CREATE INDEX idx_policy_company ON Policy(company_id, status);
CREATE INDEX idx_role_policy_mapping ON RolePolicyMapping(role_id, policy_id);
CREATE INDEX idx_employee_role ON Employee(role_id, status);
CREATE INDEX idx_acknowledgment_request_status ON AcknowledgmentRequest(status, due_date);
CREATE INDEX idx_policy_approval_status ON PolicyApproval(policy_version_id, status);
CREATE INDEX idx_template_version ON PolicyTemplateVersion(template_id, version_number);

-- Partial Indexes for Common Queries
CREATE INDEX idx_active_policies 
    ON Policy(company_id) 
    WHERE status = 'active';

CREATE INDEX idx_pending_approvals 
    ON PolicyApproval(approver_id) 
    WHERE status = 'pending';

CREATE INDEX idx_overdue_acknowledgments 
    ON AcknowledgmentRequest(employee_id) 
    WHERE status = 'pending' AND due_date < CURRENT_TIMESTAMP;

-- Full Text Search Indexes
CREATE INDEX idx_policy_content_search 
    ON PolicyVersion USING gin(to_tsvector('english', content));

CREATE INDEX idx_policy_title_search 
    ON Policy USING gin(to_tsvector('english', title || ' ' || COALESCE(description, '')));

-- Compound Indexes for Common Joins
CREATE INDEX idx_employee_company_role 
    ON Employee(company_id, role_id, status);

CREATE INDEX idx_acknowledgment_campaign_date 
    ON AcknowledgmentCampaign(company_id, start_date, end_date);

-- Triggers for Maintenance
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply update_timestamp trigger to all tables with updated_at
CREATE TRIGGER update_timestamp
    BEFORE UPDATE ON Policy
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();

-- Similar triggers for other tables with updated_at column

-- Ensure single active version per policy
CREATE OR REPLACE FUNCTION ensure_single_active_version()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.is_active THEN
        UPDATE PolicyVersion 
        SET is_active = false 
        WHERE policy_id = NEW.policy_id 
        AND id != NEW.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER single_active_version
    BEFORE INSERT OR UPDATE ON PolicyVersion
    FOR EACH ROW
    WHEN (NEW.is_active = true)
    EXECUTE FUNCTION ensure_single_active_version();

```