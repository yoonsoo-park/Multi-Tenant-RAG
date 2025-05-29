# **Amazon Bedrock Knowledge Bases for Multi-Tenant RAG Implementation**

## **1. Executive Summary**

Amazon Bedrock Knowledge Bases emerges as the primary recommendation for implementing Retrieval Augmented Generation (RAG) capabilities in a multi-tenant environment with n8n workflows. This fully managed service significantly simplifies RAG implementation by automating data ingestion, document chunking, vector embedding generation, and storage in a chosen vector database. The service provides RetrieveAndGenerate and Retrieve APIs that can be seamlessly integrated with n8n workflows, offering an excellent balance of strong data isolation, operational efficiency, and accelerated RAG development.

## **2. Bedrock Knowledge Bases Core Capabilities**

### **Managed RAG Pipeline Features**

Amazon Bedrock Knowledge Bases provides a comprehensive, fully managed RAG pipeline that abstracts away much of the complexity traditionally associated with implementing RAG systems:

- **Automated Data Ingestion:** Built-in connectors for data sources like Amazon S3, with support for various document formats
- **Document Processing:** Automatic document chunking and preprocessing
- **Vector Embedding Generation:** Uses selectable Amazon Bedrock foundation models (e.g., Amazon Titan Embeddings) for embedding generation
- **Vector Storage Management:** Supports multiple vector database backends including:
  - Amazon OpenSearch Serverless
  - Amazon Aurora PostgreSQL-Compatible Edition (with pgvector)
  - Pinecone
  - Redis Enterprise Cloud
  - MongoDB Atlas
  - Amazon OpenSearch Managed Cluster

### **API Endpoints**

Bedrock Knowledge Bases exposes two primary APIs for RAG operations:

- **RetrieveAndGenerate API:** Returns LLM-augmented answers by combining retrieval and generation
- **Retrieve API:** Returns relevant text chunks/documents without LLM generation

## **3. Multi-Tenant Architecture Strategies**

### **Primary Strategy: Per-Customer AWS Account Deployment**

The most robust approach leverages the existing N8n-Nabi multi-account architecture:

- **Account-Level Isolation:** Deploy a dedicated Bedrock Knowledge Base instance within each tenant's isolated AWS account
- **Strongest Data Segregation:** All data processing, storage, and RAG operations occur entirely within the tenant's account boundary
- **Alignment with Existing Security Model:** Consistent with the established N8n-Nabi operational model
- **Simplified Compliance:** Makes audit trails and compliance verification straightforward

**Implementation Benefits:**

- Blast radius of any potential issue is confined to a single tenant
- No risk of cross-tenant data access
- Independent scaling and configuration per tenant
- Simplified security posture

### **Secondary Strategy: Shared Knowledge Base with Metadata Filtering**

For scenarios where per-account deployment is not viable (e.g., cost considerations for smaller tenants):

**Metadata-Based Tenant Isolation:**

- Documents must be ingested with tenant-specific metadata using `.metadata.json` files
- Each document is tagged with a unique `tenant_id` during ingestion
- API calls include tenant filtering to ensure data segregation

**Key Implementation Requirements:**

- Meticulous metadata management during data ingestion
- Secure tenant_id handling in n8n workflows
- Proper filtering configuration in all API calls

## **4. n8n Integration Implementation**

### **Integration Method**

Primary integration is achieved through n8n's **HTTP Request node** calling the Bedrock Agent Runtime service APIs.

### **API Call Configuration**

**Target Endpoint:**

```
bedrock-agent-runtime.<region>.amazonaws.com
```

**Retrieve API Example:**

```json
{
  "knowledgeBaseId": "YOUR_KNOWLEDGE_BASE_ID",
  "retrievalQuery": {
    "text": "{{ $json.user_natural_language_query }}"
  },
  "retrievalConfiguration": {
    "vectorSearchConfiguration": {
      "numberOfResults": 5,
      "filter": {
        "equals": {
          "key": "your_tenant_id_metadata_key",
          "value": "{{ $json.current_tenant_id }}"
        }
      }
    }
  }
}
```

**RetrieveAndGenerate API Example:**

```json
{
  "knowledgeBaseId": "YOUR_KNOWLEDGE_BASE_ID",
  "input": {
    "text": "{{ $json.user_query }}"
  },
  "retrieveAndGenerateConfiguration": {
    "type": "KNOWLEDGE_BASE",
    "knowledgeBaseConfiguration": {
      "knowledgeBaseId": "YOUR_KNOWLEDGE_BASE_ID",
      "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-v2",
      "retrievalConfiguration": {
        "vectorSearchConfiguration": {
          "numberOfResults": 5,
          "filter": {
            "equals": {
              "key": "tenantId",
              "value": "{{ $json.current_tenant_id }}"
            }
          }
        }
      }
    }
  }
}
```

### **Authentication and Security**

- **IAM Roles:** Configure appropriate IAM roles for n8n to access Bedrock services
- **API Keys:** Alternative authentication using API Gateway if an intermediate layer is implemented
- **Cross-Account Access:** For per-account deployments, configure cross-account IAM roles if n8n runs in a central account

## **5. Multi-Tenancy Implementation Details**

### **Tenant Context Management in n8n**

Critical aspect: secure handling of `tenant_id` within n8n workflows

**Per-Tenant n8n Deployment (Recommended):**

- n8n instances deployed within each tenant's AWS account
- Tenant context is inherent to the deployment
- Simplifies security and eliminates cross-tenant risks

**Shared n8n Instance:**

- Tenant_id must be securely obtained from authenticated triggers
- Robust input validation required
- Careful workflow design to prevent tenant_id misapplication

### **Metadata Filtering Configuration**

**Document Ingestion with Tenant Metadata:**
When documents are ingested into a shared Knowledge Base, they must include tenant-specific metadata:

```json
{
  "tenantId": "customer-a-12345",
  "documentType": "policy",
  "department": "legal",
  "classification": "internal"
}
```

**API Call Filtering:**
Every Retrieve or RetrieveAndGenerate API call must include appropriate filtering:

```json
"filter": {
  "equals": {
    "key": "tenantId",
    "value": "{{ $json.current_tenant_id }}"
  }
}
```

**Complex Filtering Examples:**

```json
"filter": {
  "and": [
    {
      "equals": {
        "key": "tenantId",
        "value": "{{ $json.current_tenant_id }}"
      }
    },
    {
      "equals": {
        "key": "documentType",
        "value": "policy"
      }
    }
  ]
}
```

## **6. Workflow Integration Patterns**

### **Basic RAG Workflow with n8n**

1. **Trigger:** Webhook or other trigger receives user query with tenant context
2. **Validation:** Validate and extract tenant_id securely
3. **Query Processing:** Format the user query for Bedrock KB API
4. **API Call:** HTTP Request node calls Bedrock Knowledge Base
5. **Response Processing:** Handle and format the RAG response
6. **Output:** Return results to the user or downstream systems

### **Advanced Workflow Patterns**

**Multi-Step RAG with Validation:**

```
Trigger → Tenant Validation → Query Enhancement → Bedrock KB Call → Response Validation → Output
```

**Hybrid Search Approach:**

```
Trigger → Parallel Processing:
  ├── Bedrock KB Vector Search
  ├── Traditional Database Query
  └── Merge Results → LLM Synthesis → Output
```

### **Error Handling and Fallbacks**

- Implement proper error handling for API failures
- Configure fallback mechanisms for high availability
- Log tenant-specific activities for audit purposes

## **7. Operational Considerations**

### **Data Ingestion Strategies**

**Automated Ingestion Pipeline:**

- S3 event triggers for new document uploads
- Lambda functions for metadata assignment
- Automated chunking and embedding generation

**Batch Processing:**

- Scheduled bulk uploads for large document sets
- Progress tracking and error handling
- Tenant-specific ingestion queues

### **Performance Optimization**

**Caching Strategies:**

- Implement caching layers for frequently accessed content
- Tenant-specific cache partitioning
- Cache invalidation strategies for updated documents

**Query Optimization:**

- Optimize embedding models for domain-specific content
- Fine-tune chunk sizes and overlap parameters
- Implement query preprocessing and enhancement

### **Monitoring and Observability**

**Key Metrics to Track:**

- Query latency per tenant
- Retrieval accuracy and relevance scores
- API usage and cost per tenant
- Error rates and failure patterns

**Logging Requirements:**

- Tenant-specific access logs
- Query performance metrics
- Security and compliance audit trails

## **8. Cost Management**

### **Cost Components**

- Bedrock Knowledge Base usage charges
- Underlying vector store costs (OpenSearch, Aurora, etc.)
- Foundation model inference costs
- Data transfer and storage costs

### **Cost Optimization Strategies**

**Per-Tenant Cost Allocation:**

- Implement cost tracking by tenant
- Usage-based billing models
- Resource optimization based on tenant usage patterns

**Shared Resource Optimization:**

- Optimize embedding model selection
- Implement intelligent caching
- Right-size vector store resources

## **9. Security and Compliance**

### **Data Protection**

- Encryption in transit and at rest
- Fine-grained access controls
- Data residency compliance
- Secure key management

### **Audit and Compliance**

- Comprehensive logging of all RAG operations
- Tenant-specific audit trails
- Compliance with data protection regulations
- Regular security assessments

## **10. Deployment and Infrastructure**

### **AWS CDK Implementation**

Leverage existing N8n-Nabi CDK infrastructure to deploy Bedrock Knowledge Bases:

```typescript
// Example CDK stack for per-tenant Bedrock KB
export class BedrockKnowledgeBaseStack extends Stack {
  constructor(scope: Construct, id: string, props: BedrockKBStackProps) {
    super(scope, id, props);

    // Create tenant-specific Knowledge Base
    const knowledgeBase = new BedrockKnowledgeBase(this, "TenantKB", {
      name: `${props.tenantId}-knowledge-base`,
      roleArn: props.serviceRole.roleArn,
      dataSource: {
        s3Configuration: {
          bucketArn: props.dataBucket.bucketArn,
        },
      },
      vectorIndex: {
        openSearchServerlessConfiguration: {
          collectionArn: props.vectorCollection.attrArn,
          vectorIndexName: `${props.tenantId}-vector-index`,
        },
      },
    });
  }
}
```

### **Cross-Account Access Configuration**

For shared n8n instance accessing per-tenant Knowledge Bases:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::CENTRAL-ACCOUNT:role/N8nExecutionRole"
      },
      "Action": ["bedrock:Retrieve", "bedrock:RetrieveAndGenerate"],
      "Resource": "arn:aws:bedrock:region:TENANT-ACCOUNT:knowledge-base/*"
    }
  ]
}
```

## **11. Migration Strategy**

### **From Self-Managed pgvector to Bedrock Knowledge Bases**

**Phase 1: Parallel Deployment**

- Deploy Bedrock KB alongside existing pgvector
- Implement data synchronization
- Test with subset of tenants

**Phase 2: Gradual Migration**

- Migrate tenants in phases
- Monitor performance and accuracy
- Maintain rollback capabilities

**Phase 3: Full Transition**

- Complete tenant migration
- Decommission legacy systems
- Optimize new architecture

### **Data Migration Considerations**

- Document format compatibility
- Embedding model consistency
- Metadata preservation
- Query pattern adaptation

## **12. Best Practices and Recommendations**

### **Development Best Practices**

- Implement comprehensive testing for multi-tenant scenarios
- Use infrastructure as code for reproducible deployments
- Establish clear data governance policies
- Implement proper CI/CD pipelines

### **Operational Best Practices**

- Regular backup and disaster recovery testing
- Performance monitoring and alerting
- Capacity planning and scaling strategies
- Security review and updates

### **Integration Best Practices**

- Standardize API response formats
- Implement proper error handling
- Use idempotent operations where possible
- Design for high availability and fault tolerance

## **13. Conclusion**

Amazon Bedrock Knowledge Bases provides a compelling solution for implementing RAG capabilities in a multi-tenant environment with n8n workflows. The service's fully managed nature significantly reduces operational overhead while providing robust multi-tenancy options through both physical (per-account) and logical (metadata filtering) isolation strategies.

The integration with n8n via HTTP Request nodes is straightforward and allows for sophisticated workflow orchestration around RAG operations. When combined with the existing N8n-Nabi multi-account architecture, Bedrock Knowledge Bases offers a secure, scalable, and efficient path forward for transitioning from self-managed vector database solutions.

The key to successful implementation lies in careful planning of the multi-tenancy strategy, proper security configuration, and comprehensive testing across different tenant scenarios. With these foundations in place, Bedrock Knowledge Bases can provide a future-proof vector data strategy that meets stringent isolation requirements while delivering operational efficiency gains.

## **14. References**

### **Amazon Bedrock Knowledge Bases Documentation**

- [Knowledge bases for Amazon Bedrock - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/retrieval-augmented-generation-options/rag-fully-managed-bedrock.html)
- [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases - AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Vector database options - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html)

### **Multi-Tenant Implementation Guides**

- [Multi-tenant RAG with Amazon Bedrock Knowledge Bases | AWS Machine Learning Blog](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-with-amazon-bedrock-knowledge-bases/)
- [Multi-tenancy in RAG applications in a single Amazon Bedrock knowledge base with metadata filtering | AWS Machine Learning Blog](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/)
- [Implementing tenant isolation using Agents for Amazon Bedrock in a multi-tenant environment | AWS Machine Learning Blog](https://aws.amazon.com/blogs/machine-learning/implementing-tenant-isolation-using-agents-for-amazon-bedrock-in-a-multi-tenant-environment/)

### **Integration with Vector Stores**

- [Multi-tenant vector search with Amazon Aurora PostgreSQL and Amazon Bedrock Knowledge Bases | AWS Database Blog](https://aws.amazon.com/blogs/database/multi-tenant-vector-search-with-amazon-aurora-postgresql-and-amazon-bedrock-knowledge-bases/)
- [Amazon Bedrock Knowledge Bases now supports Amazon OpenSearch Managed Cluster for vector storage](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-opensearch-cluster-vector-storage/)

### **Code Examples and Samples**

- [aws-samples/multi-tenant-full-stack-rag-application-demo - GitHub](https://github.com/aws-samples/multi-tenant-full-stack-rag-application-demo)
- [data-for-saas-patterns/samples/multi-tenant-vector-database/amazon-aurora/aws-managed/2_bedrock_knowledgebase_managed_rag.ipynb - GitHub](https://github.com/aws-samples/data-for-saas-patterns/blob/main/samples/multi-tenant-vector-database/amazon-aurora/aws-managed/2_bedrock_knowledgebase_managed_rag.ipynb)

### **n8n Integration Resources**

- [What do you use to have a RAG-based knowledge base in your n8n AI LLM workflows?](https://community.n8n.io/t/what-do-you-use-to-have-a-rag-based-knowledge-base-in-your-n8n-ai-llm-workflows/117629)
- [LangChain concepts in n8n - n8n Docs](https://docs.n8n.io/advanced-ai/langchain/langchain-n8n/)
- [AWS Bedrock Chat Model node documentation - n8n Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatawsbedrock/)

### **Architecture and Best Practices**

- [Amazon Bedrock & Retrieval Augmented Generation (RAG): Building Smarter AI Systems with Context-Aware Response - Microtica](https://www.microtica.com/blog/amazon-bedrock-retrieval-augmented-generation-rag)
- [AWS re:Invent 2024 - Generative AI meets multi-tenancy: Inside a working solution (SAS407) - YouTube](https://www.youtube.com/watch?v=hq3h5HNIBPE)
- [Building SaaS on AWS - S7E1 - Multi tenant GenAI architecture patterns - YouTube](https://www.youtube.com/watch?v=PGzxSzPPTWI)

### **Cross-Account Access and Security**

- [Cross-account access to Amazon Bedrock Knowledge Base for RAG applications](https://repost.aws/questions/QUusDKSZ05RdWosaRhNlV-tQ/cross-account-access-to-amazon-bedrock-knowledge-base-for-rag-applications)
- [Multi tenancy in Bedrock | AWS re:Post](https://repost.aws/questions/QUwOYioZ5vS0CzbnMr9bpOvw/multi-tenancy-in-bedrock)

### **Cost and Performance Optimization**

- [Cost comparisons and considerations - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/cost.html)
- [Choosing the Right Vector Database on AWS for AI and ML Workloads](https://www.cloudthat.com/resources/blog/choosing-the-right-vector-database-on-aws-for-ai-and-ml-workloads)
- [AWS Prescriptive Guidance - Choosing an AWS vector database for RAG use cases - AWS Documentation](https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/choosing-an-aws-vector-database-for-rag-use-cases.pdf)

### **Community Resources and Discussions**

- [Build a Custom Knowledge RAG Chatbot using n8n](https://blog.n8n.io/rag-chatbot/)
- [How to access different AWS Bedrock Models? - Questions - n8n Community](https://community.n8n.io/t/how-to-access-different-aws-bedrock-models/90993)
- [AWS Bedrock HTTP request and Model listings - Nodes - n8n Community](https://community.n8n.io/t/aws-bedrock-http-request-and-model-listings/117849)
