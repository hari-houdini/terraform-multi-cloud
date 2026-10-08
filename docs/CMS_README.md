# Payload CMS Learning POC: Documentation Index

This folder contains comprehensive documentation for a Payload CMS blogging platform POC integrated with the terraform-multi-cloud infrastructure.

## Documents

### 1. **CMS_JOURNEY.md** (Start Here)
**Purpose**: Understand the user journey and how data flows through AWS services.

**What's Inside**:
- **3 User Journeys**:
  - Admin publishes a blog post (step-by-step, data paths, AWS services touched)
  - Public user reads a blog post (how data is fetched and cached)
  - Admin forgot password (authentication reset flow)
  
- **Key Insights**: Explains *why* each AWS service is needed (Cognito, API Gateway, Lambda, RDS, S3, CloudFront, VPC, IAM)

- **Data Relationships**: Prisma ORM schema (Posts → Authors → Categories → Tags)

- **Caching Strategy**: What's cached where and for how long

- **Error Cases**: Common failure scenarios

**Read this first if**: You want to understand how the CMS works end-to-end and why each service exists.

---

### 2. **CMS_ARCHITECTURE_DIAGRAM.md** (Visual Reference)
**Purpose**: Visual diagrams showing service interactions, data flows, and architecture patterns.

**What's Inside**:
- **9 Mermaid Diagrams**:
  1. **Service Architecture Diagram**: All AWS services and connections
  2. **Admin Publishing Sequence**: Step-by-step when admin publishes a post
  3. **Public Read Sequence**: Step-by-step when user reads a post
  4. **Database Schema (ERD)**: Tables, relationships, Prisma structure
  5. **Data Flow Diagram**: From upload to display
  6. **Authentication & Authorization**: Cognito + JWT + IAM
  7. **Caching Strategy**: CloudFront, browser, RDS caching layers
  8. **VPC Network**: Private subnets, security groups, isolation
  9. **Prisma ORM Benefits**: Why ORM matters vs. manual SQL

**Read this if**: You prefer visual representations or want to see architecture at a glance.

---

## Technology Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| **CMS** | Payload CMS v3.90.2 | Headless, Node.js, REST API |
| **ORM** | Prisma | Type-safe, auto-joins, schema-first |
| **Database** | RDS PostgreSQL | Relational, portable across clouds |
| **Backend** | Lambda or EC2 | Serverless or traditional compute |
| **Frontend** | Static site (S3 + CloudFront) | Fast, scalable, low cost |
| **Media** | S3 + CloudFront | Scalable storage, CDN caching |
| **Auth** | Cognito | AWS-managed authentication |
| **API** | API Gateway | REST endpoints, rate limiting, CORS |
| **Networking** | VPC | Private network, security groups |

---

## Key Learnings

### 1. User Journey = Service Cascade
When an admin publishes a post, it triggers:
```
Cognito (auth) → API Gateway → Lambda → Prisma → RDS → S3 → CloudFront
```

### 2. ORM Simplifies Database Access
**Without Prisma**: Write complex SQL joins manually
**With Prisma**: Define schema, ORM generates SQL automatically

Example:
```typescript
const post = await prisma.post.findUnique({
  where: { slug: 'learning-aws' },
  include: { author: true, categories: true, tags: true },
});
```

### 3. Caching Layers Improve Performance
- **CloudFront**: Cache API responses (5 min) and images (1 day)
- **Browser**: Cache JWT and form state
- **RDS**: Source of truth

### 4. Security Through Layers
- **Cognito**: Authenticate users (who are you?)
- **JWT**: Prove you're authenticated (bearer token)
- **IAM**: Authorize actions (what can you do?)
- **Security Groups**: Network firewall (which traffic is allowed?)

### 5. Data Relationships Matter
Posts have:
- **One author** (1:N relationship)
- **Many categories** (M:N relationship)
- **Many tags** (M:N relationship)

Prisma automatically handles all joins.

---

## Architecture Overview

```
[Internet] 
    ↓
[CloudFront] ← Caches everything
    ├→ [S3] (admin UI, images)
    ├→ [API Gateway]
    │   └→ [Lambda/EC2] (Payload CMS)
    │       ├→ [Prisma ORM]
    │       └→ [RDS PostgreSQL]
    ├→ [Cognito] (JWT auth)
    └→ [IAM] (permissions)

[VPC] wraps Lambda/EC2 and RDS (private)
```

---

## Data Model

```
Admin User
  ├─ creates Posts
  │   ├─ has Authors
  │   ├─ has Categories (many-to-many)
  │   ├─ has Tags (many-to-many)
  │   └─ has Featured Image (S3 key)
```

### Prisma Schema (Simplified)
```prisma
model Post {
  id        Int
  title     String
  slug      String @unique
  content   String
  published Boolean
  
  author    Author @relation(fields: [authorId], references: [id])
  authorId  Int
  
  categories Category[]
  tags       Tag[]
  featuredImage String? // S3 key
}

model Author {
  id    Int
  name  String
  posts Post[]
}

model Category {
  id    Int
  name  String @unique
  posts Post[]
}

model Tag {
  id    Int
  name  String @unique
  posts Post[]
}
```

---

## Next Steps

### Phase 2: Build Terraform Modules (TODO)
- [ ] Create `/terraform/aws/modules/rds/` for PostgreSQL setup
- [ ] Create `/terraform/aws/modules/lambda/` for Payload CMS deployment
- [ ] Create `/terraform/aws/modules/s3/` for media storage
- [ ] Create `/terraform/aws/modules/cognito/` for authentication
- [ ] Create `/terraform/aws/modules/cloudfront/` for CDN

### Phase 3: Build Application (TODO)
- [ ] Create Node.js + Payload CMS project
- [ ] Set up Prisma schema
- [ ] Create Payload collections (Post, Author, Category, Tag)
- [ ] Implement S3 upload via presigned URLs
- [ ] Implement Cognito integration

### Phase 4: Deployment (TODO)
- [ ] Deploy to AWS using Terraform modules
- [ ] Test admin publishing flow
- [ ] Test public read flow
- [ ] Verify caching and performance

---

## Questions Answered

### "Why Payload CMS?"
- Headless (API-first), no monolithic frontend
- Node.js native (matches ORM choice)
- Flexible, no built-in ORM (gives us opportunity to learn Prisma)

### "Why Prisma ORM?"
- Type-safe database access (TypeScript)
- Schema-first approach (clear structure)
- Automatic SQL generation (less boilerplate)
- Multi-cloud ready (same schema on AWS, GCP, Azure)
- Learning value (understand ORMs in depth)

### "Why PostgreSQL over DynamoDB?"
- Relational data (posts, authors, categories, tags)
- Portable across clouds (RDS on AWS, Cloud SQL on GCP, etc.)
- Traditional choice, widely supported

### "Why CloudFront + S3 for images?"
- Scalable (S3 handles unlimited objects)
- Cheap (pay per GB)
- Cached globally (CloudFront edge locations)
- Decoupled from database (no storing images in DB)

### "Why VPC for networking?"
- Security (RDS not exposed to internet)
- Learning value (understand network isolation)
- Production-ready pattern

---

## Verification Checklist

Once you implement the CMS, verify:

### Admin Publishing Flow
- [ ] Log in with Cognito
- [ ] Create blog post with title, content, category, tags
- [ ] Upload featured image (verify S3 presigned URL)
- [ ] Save draft (verify RDS has post with published=false)
- [ ] Publish post (verify RDS updated, CloudFront cache invalidated)

### Public Read Flow
- [ ] Call GET /api/posts (public endpoint, no auth)
- [ ] Verify post data returned with author, categories, tags
- [ ] Verify featured image URL is S3 key
- [ ] Verify CloudFront caching (response headers have Cache-Control)

### Caching
- [ ] Request same post twice, second should be cache hit
- [ ] Check CloudFront metrics for cache hits
- [ ] Verify images cached for 1 day

---

## File Locations

```
terraform-multi-cloud/
├── docs/
│   ├── ARCHITECTURE.md              (existing: multi-cloud patterns)
│   ├── LOCAL_SETUP.md               (existing: Floci emulator setup)
│   ├── CI_CD.md                     (existing: GitHub Actions)
│   ├── CMS_README.md                ← YOU ARE HERE
│   ├── CMS_JOURNEY.md               ← User journeys & flows
│   └── CMS_ARCHITECTURE_DIAGRAM.md  ← Mermaid diagrams & visuals
├── terraform/
│   ├── aws/                         (todo: Payload CMS modules)
│   ├── gcp/                         (existing: cloud patterns)
│   └── azure/                       (existing: cloud patterns)
└── README.md                        (existing: project overview)
```

---

## Quick Links

- **Journey Document**: [CMS_JOURNEY.md](./CMS_JOURNEY.md)
- **Architecture Diagrams**: [CMS_ARCHITECTURE_DIAGRAM.md](./CMS_ARCHITECTURE_DIAGRAM.md)
- **Multi-Cloud Architecture**: [ARCHITECTURE.md](./ARCHITECTURE.md)
- **Local Setup**: [LOCAL_SETUP.md](./LOCAL_SETUP.md)

---

## Feedback & Questions

This POC demonstrates:
1. **Cohesive user journey** (instead of scattered service knowledge)
2. **ORM integration** (Prisma with PostgreSQL)
3. **AWS service interactions** (real-world application pattern)

The next phase will turn these diagrams and journeys into actual Terraform code and application code.

Happy learning! 🚀
