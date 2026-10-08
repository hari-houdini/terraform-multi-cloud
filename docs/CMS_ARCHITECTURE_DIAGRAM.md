# Payload CMS Learning POC: Architecture Diagrams

This document contains Mermaid diagrams visualizing the Payload CMS architecture and data flows across AWS services.

---

## 1. Service Architecture Diagram

**Diagram**: [`diagrams/01-architecture.mmd`](./diagrams/01-architecture.mmd)

This diagram shows all AWS services and how they connect.

### Service Descriptions

| Service | Role | Responsibilities |
|---------|------|---|
| **CloudFront** | Content Delivery Network (CDN) | Cache and serve admin UI assets, images, and API responses from edge locations near users |
| **S3 (Admin UI)** | Static Asset Storage | Serve Payload CMS admin dashboard (HTML, CSS, JavaScript) |
| **S3 (Media)** | Media Storage | Store blog post images, featured images, uploads |
| **API Gateway** | API Entry Point | Expose REST API endpoints, route requests to Lambda/EC2, handle CORS, rate limiting |
| **Cognito** | Authentication Service | Authenticate users, issue JWT tokens, manage user sessions |
| **Lambda / EC2** | Application Runtime | Run Payload CMS application logic, Prisma ORM queries, file uploads |
| **Prisma ORM** | Database Abstraction | Type-safe database access, automatic SQL generation, relation handling |
| **RDS PostgreSQL** | Relational Database | Persist posts, users, categories, tags, and all relationships |
| **VPC** | Virtual Private Cloud | Secure network isolation for private resources (EC2, RDS) |
| **IAM** | Identity & Access Management | Control permissions (who can read/write what) |

---

## 2. Admin Publishing Flow: Sequence Diagram

**Diagram**: [`diagrams/02-admin-publish-sequence.mmd`](./diagrams/02-admin-publish-sequence.mmd)

This diagram shows the step-by-step flow when an admin publishes a blog post.

### Key Points

1. **Admin UI is cached** (CloudFront) → Subsequent visits faster
2. **JWT token** used for authentication on every admin request
3. **Presigned URL** allows browser to upload directly to S3 without backend relay
4. **Prisma transaction** ensures all-or-nothing insertion (post + categories + tags)
5. **Cache invalidation** ensures public API serves fresh data
6. **Database relations** automatically handled by Prisma

---

## 3. Public Read Flow: Sequence Diagram

**Diagram**: [`diagrams/03-public-read-sequence.mmd`](./diagrams/03-public-read-sequence.mmd)

This diagram shows how public users fetch and read blog posts.

### Key Points

1. **CloudFront cache** saves repeated API calls (5 min for data, 1 day for images)
2. **No JWT required** for public endpoints (anonymous read)
3. **Prisma auto-joins** author, categories, tags in single query
4. **Nested response** includes all related data
5. **S3 image caching** via CloudFront for fast image loads

---

## 4. Database Schema Diagram

**Diagram**: [`diagrams/04-database-schema.mmd`](./diagrams/04-database-schema.mmd)

This diagram shows the relational structure stored in RDS PostgreSQL.

### Schema Details

- **USERS**: Admins who create posts
- **POSTS**: Blog posts with foreign key to author
- **CATEGORIES**: Blog categories (AWS, Cloud, etc.)
- **TAGS**: Blog tags (multiple per post)
- **POST_CATEGORIES**: Many-to-many junction table
- **POST_TAGS**: Many-to-many junction table

**Prisma ORM** handles all joins automatically. When you query a post, Prisma fetches author, categories, and tags in one optimized query.

---

## 5. Data Flow: Upload to Display

**Diagram**: [`diagrams/05-data-flow.mmd`](./diagrams/05-data-flow.mmd)

This diagram shows how data flows from admin upload to public display.

---

## 6. Authentication & Authorization Flow

**Diagram**: [`diagrams/06-auth-flow.mmd`](./diagrams/06-auth-flow.mmd)

This diagram shows how Cognito, JWT, and IAM work together.

### Authentication Layers

1. **Cognito** validates password → issues JWT
2. **JWT token** sent with every admin request
3. **API Gateway** validates JWT with Cognito
4. **IAM policies** control what actions are allowed (read-only vs. write)

---

## 7. Caching Strategy Diagram

**Diagram**: [`diagrams/07-caching-strategy.mmd`](./diagrams/07-caching-strategy.mmd)

This diagram shows what's cached where and for how long.

### Cache Invalidation

- **Manual**: When admin publishes post, Lambda sends CloudFront invalidation request
- **Time-based**: Cache expires after TTL (5 min for API, 1 day for images)
- **Browser**: JWT and form state cleared on logout or page refresh

---

## 8. VPC Network Diagram

**Diagram**: [`diagrams/08-vpc-network.mmd`](./diagrams/08-vpc-network.mmd)

This diagram shows private network segmentation.

### Key Security Points

- **RDS is private**: Only accessible from Lambda/EC2, never from internet
- **Lambda has IAM role**: Allows access to S3, RDS, Cognito without storing credentials
- **API Gateway is managed**: AWS handles DDoS protection, SSL termination
- **Security groups**: Firewall rules control traffic between subnets

---

## 9. Prisma ORM Benefits Diagram

**Diagram**: [`diagrams/09-prisma-benefits.mmd`](./diagrams/09-prisma-benefits.mmd)

This diagram shows why Prisma simplifies development.

### Example: Fetching Post with Relations

**Without Prisma (Manual SQL):**
```sql
SELECT p.*, a.name, a.email, c.name, t.name
FROM posts p
LEFT JOIN authors a ON p.authorId = a.id
LEFT JOIN post_categories pc ON p.id = pc.postId
LEFT JOIN categories c ON pc.categoryId = c.id
LEFT JOIN post_tags pt ON p.id = pt.postId
LEFT JOIN tags t ON pt.tagId = t.id
WHERE p.slug = 'learning-aws'
ORDER BY c.name, t.name;
```

**With Prisma (Type-safe):**
```typescript
const post = await prisma.post.findUnique({
  where: { slug: 'learning-aws' },
  include: {
    author: true,
    categories: true,
    tags: true,
  },
});
// Returns: { id, title, slug, content, author, categories, tags }
// TypeScript knows all fields and their types!
```

---

## Summary

These diagrams show:

1. **Architecture** → All services and their roles
2. **Admin Flow** → How posts get published (Cognito → RDS → S3)
3. **Public Flow** → How users read posts (cached API → RDS joins → S3 images)
4. **Database** → Relational structure with Prisma ORM
5. **Data Flow** → From upload to display
6. **Auth** → JWT + IAM permission layers
7. **Caching** → CloudFront edge + browser storage
8. **Networking** → VPC security and isolation
9. **Prisma Value** → Why ORM matters

Each diagram builds understanding: what services exist, how they connect, where data flows, and why each layer matters.
