# Payload CMS Learning POC: User Journey

## Overview

This document maps the complete journey of users interacting with a Payload CMS blogging platform deployed on AWS infrastructure. Each journey shows how data flows through AWS services, demonstrating why each service is needed in the architecture.

**Goal**: Understand not just *what* services exist, but *why* and *when* they're used in a real application.

---

## Journey 1: Admin Publishes a Blog Post

### Narrative
An author wants to publish a new blog post about AWS architecture. They log into the Payload CMS admin dashboard, write content, upload a featured image, add metadata (category, tags), and publish. Behind the scenes, this triggers a cascade of service interactions.

### Step-by-Step Flow

| Step | User Action | AWS Service(s) | What Happens | Data Path |
|------|-------------|---|---|---|
| 1 | Visits `/admin` URL | CloudFront, S3 | Browser fetches admin UI assets (HTML, CSS, JS) from CloudFront cache, or S3 if cold | S3 → CloudFront → Browser |
| 2 | Clicks "Log In" | Cognito | Admin enters email + password → sent to API Gateway | Browser → API Gateway |
| 3 | API validates credentials | API Gateway → Lambda/EC2 → Cognito | Lambda calls Cognito to verify credentials, receives JWT token | Cognito → Lambda → Browser |
| 4 | JWT token stored locally | Browser | Token stored in browser localStorage for subsequent requests | JWT in headers for all admin requests |
| 5 | Admin UI loads dashboard | Cognito JWT | Dashboard authenticated with JWT; API verifies JWT on each request | JWT → API Gateway → Lambda → Cognito validator |
| 6 | Admin clicks "New Post" | Payload CMS | Form loads for post creation (title, slug, content, featured image, category, tags) | Browser renders form |
| 7 | Admin enters post details | Payload CMS | Form data held in browser state (not yet saved) | Client-side validation |
| 8 | Admin selects featured image file | S3 (presigned URL) | Admin chooses image file; browser requests presigned URL from backend | Browser → API Gateway → Lambda → S3 |
| 9 | Backend generates presigned URL | Lambda → S3 | Lambda generates time-limited S3 presigned URL for direct upload | S3 SDK → Lambda |
| 10 | Admin uploads image directly to S3 | S3 (presigned) | Browser bypasses backend, uploads directly to S3 bucket using presigned URL | Browser → S3 (direct) |
| 11 | S3 confirms upload | S3 → Lambda (optional webhook) | Webhook triggers Lambda to verify image, generate thumbnails | S3 → Lambda trigger |
| 12 | Admin clicks "Save Draft" | Lambda/EC2 → Prisma | Post data sent to backend; Prisma ORM validates schema, opens RDS transaction | Browser → API Gateway → Lambda → RDS |
| 13 | Post written to RDS | Prisma → RDS PostgreSQL | Insert POST record with title, slug, content, published=false, authorId=admin's ID | `INSERT INTO posts (title, slug, content, authorId, published) VALUES (...)` |
| 14 | Featured image URL stored | Prisma → RDS | Post record updated with S3 image key in `featuredImage` column | `UPDATE posts SET featuredImage='s3://bucket/blog/123/image.jpg'` |
| 15 | Category + Tags assigned | Prisma → RDS | Many-to-many relationship: create rows in `post_categories` and `post_tags` junction tables | `INSERT INTO post_categories (postId, categoryId)...` |
| 16 | Prisma commit transaction | RDS | All inserts/updates committed atomically; if any fails, all rollback | Transaction commits or rolls back |
| 17 | Admin clicks "Publish" | Lambda/EC2 | Post status changes from `published=false` to `published=true` | Browser → API Gateway → Lambda → RDS |
| 18 | RDS updated to published | Prisma → RDS | `UPDATE posts SET published=true WHERE id=123` | RDS row updated |
| 19 | Cache invalidation triggered | Lambda → CloudFront | Lambda sends invalidation request to CloudFront to purge any cached API responses | CloudFront cache flushed |
| 20 | Success notification | Payload CMS | Admin sees "Post published!" notification; dashboard refreshes | UI updates |

### Data Stored Across Services

**RDS PostgreSQL**:
```
posts:
  - id: 123
  - title: "Learning AWS Architecture"
  - slug: "learning-aws-architecture"
  - content: "Long markdown content..."
  - authorId: 1 (admin user)
  - featuredImage: "s3://cms-bucket/blog/123/featured.jpg"
  - published: true
  - createdAt: 2025-10-07 10:00:00
  - updatedAt: 2025-10-07 10:05:00

authors:
  - id: 1
  - name: "John Doe"
  - email: "john@example.com"

post_categories:
  - postId: 123
  - categoryId: 5 (AWS category)

post_tags:
  - postId: 123
  - tagId: 10 (Architecture tag)
  - postId: 123
  - tagId: 12 (Cloud tag)
```

**S3 Bucket**:
```
s3://cms-bucket/
  └── blog/
      └── 123/
          ├── featured.jpg (original)
          ├── featured-thumb.jpg (auto-generated)
          └── featured-medium.jpg (auto-generated)
```

**Cognito**:
- User session/JWT token (temporary, expires in 1 hour)

### AWS Services Involved

- **Cognito**: Authenticate admin user
- **API Gateway**: Route admin requests to backend
- **Lambda or EC2**: Run Payload CMS application logic
- **Prisma ORM**: Schema validation, automatic SQL generation
- **RDS PostgreSQL**: Persist post, author, category, tag data
- **S3**: Store featured image and media assets
- **CloudFront**: Cache admin UI assets and invalidate caches
- **VPC**: Private networking between EC2 and RDS
- **IAM**: Control who can call which APIs, who can write to S3

---

## Journey 2: Public User Reads a Blog Post

### Narrative
A reader discovers the blog via search or link, navigates to the published post, and reads the content with the featured image. This journey demonstrates how the data stored in Journey 1 is retrieved and served to public users.

### Step-by-Step Flow

| Step | User Action | AWS Service(s) | What Happens | Data Path |
|------|-------------|---|---|---|
| 1 | Visits blog homepage | CloudFront, S3 | Browser fetches static website (HTML, CSS, JS) from S3 via CloudFront | S3 → CloudFront → Browser |
| 2 | Clicks blog post link | Browser | Link points to `/posts/learning-aws-architecture` | Client-side navigation |
| 3 | Frontend calls API | API Gateway | Frontend makes `GET /api/posts/learning-aws-architecture` (public endpoint, no auth) | Browser → API Gateway |
| 4 | API Gateway routes request | API Gateway → Lambda/EC2 | No JWT required; routes to public API endpoint | API Gateway routes to Lambda or EC2 endpoint |
| 5 | Lambda/EC2 receives request | Lambda/EC2 → Prisma | Payload CMS backend receives slug parameter | Lambda execution environment |
| 6 | Prisma queries RDS | Prisma → RDS PostgreSQL | `SELECT * FROM posts WHERE slug='learning-aws-architecture' AND published=true;` | RDS query executed |
| 7 | RDS returns post + relations | Prisma (auto-join) | Prisma automatically fetches related author, categories, tags (joins via foreign keys) | RDS returns joined result set |
| 8 | Response built | Lambda/EC2 → JSON response | Backend formats response with post data, author name/email, category names, tag names, S3 image URL | JSON response object |
| 9 | Response cached (optional) | Lambda/EC2 → CloudFront | API response sent through CloudFront with cache headers (Cache-Control: max-age=300) | CloudFront caches response for 5 min |
| 10 | Browser receives response | API response → Browser | Frontend gets JSON with post data, image URL, metadata | JavaScript processes JSON |
| 11 | Frontend renders HTML | Browser | Renders post title, content, author info, categories, tags | DOM updated with post content |
| 12 | Featured image loads | CloudFront, S3 | Frontend requests featured image URL (`s3://cms-bucket/blog/123/featured.jpg`) via CloudFront | Browser → CloudFront → S3 |
| 13 | Image served from cache | CloudFront | If image was accessed recently, served from CloudFront cache with headers `Cache-Control: max-age=86400` (1 day) | CloudFront cache hit / S3 origin |
| 14 | User reads post | Browser | Full blog post displayed with image | Post displayed in browser |
| 15 | User navigates away | Browser | Session ends; no further API calls | Browser idle |

### Data Retrieved from RDS (Single Query)

```sql
-- Prisma generates this (simplified for illustration)
SELECT 
  p.id, p.title, p.slug, p.content, p.featuredImage,
  a.name AS author_name, a.email AS author_email,
  c.name AS category_name,
  t.name AS tag_name
FROM posts p
LEFT JOIN authors a ON p.authorId = a.id
LEFT JOIN post_categories pc ON p.id = pc.postId
LEFT JOIN categories c ON pc.categoryId = c.id
LEFT JOIN post_tags pt ON p.id = pt.postId
LEFT JOIN tags t ON pt.tagId = t.id
WHERE p.slug = 'learning-aws-architecture' AND p.published = true;

-- Prisma returns nested object:
{
  id: 123,
  title: "Learning AWS Architecture",
  slug: "learning-aws-architecture",
  content: "...",
  featuredImage: "s3://cms-bucket/blog/123/featured.jpg",
  author: {
    name: "John Doe",
    email: "john@example.com"
  },
  categories: [
    { name: "AWS" }
  ],
  tags: [
    { name: "Architecture" },
    { name: "Cloud" }
  ]
}
```

### AWS Services Involved

- **API Gateway**: Route public read-only requests
- **Lambda or EC2**: Serve Payload CMS API
- **Prisma ORM**: Automatic joins, relation fetching
- **RDS PostgreSQL**: Query posts, authors, categories, tags
- **S3**: Store and serve featured images
- **CloudFront**: Cache API responses and images (faster for repeat visitors)
- **VPC**: Private networking for Lambda/EC2 to RDS

---

## Journey 3: Admin Forgot Password (Alternative Flow)

### Narrative
An admin forgets their password and needs to reset it. This journey shows how authentication state changes via RDS, and how email/SMS notifications work.

### Step-by-Step Flow

| Step | User Action | AWS Service(s) | What Happens | Data Path |
|------|-------------|---|---|---|
| 1 | Clicks "Forgot Password?" on login | Browser | Form asks for email address | Client-side form |
| 2 | Enters email and submits | API Gateway → Lambda/EC2 | Email sent to API endpoint | Browser → API Gateway → Lambda |
| 3 | Lambda looks up user in RDS | Prisma → RDS | `SELECT id, email FROM users WHERE email='john@example.com'` | RDS query |
| 4 | Generate reset token | Lambda | Lambda generates random reset token (UUID), calculates expiry (15 min from now) | In-memory token generation |
| 5 | Store reset token in RDS | Prisma → RDS | `UPDATE users SET resetToken='uuid...', resetTokenExpiry='2025-10-07 10:20:00' WHERE id=1` | RDS update |
| 6 | Send email with reset link | Lambda → SES | Lambda calls AWS SES to send email with link: `/reset-password?token=uuid...` | SES sends transactional email |
| 7 | Admin receives email | Email provider → Admin | Email contains reset link | Admin checks email |
| 8 | Admin clicks reset link | Browser → API Gateway | Browser navigates to `/reset-password?token=uuid...` | URL with token parameter |
| 9 | Validate token | Lambda/EC2 → Prisma → RDS | `SELECT * FROM users WHERE resetToken='uuid...' AND resetTokenExpiry > NOW()` | RDS query validates token |
| 10 | Token is valid, show reset form | Browser | Frontend displays "Enter new password" form | Form rendered |
| 11 | Admin enters new password | API Gateway → Lambda/EC2 | New password sent to backend | Browser → API Gateway → Lambda |
| 12 | Hash password | Lambda | Lambda hashes new password using bcrypt | One-way hash |
| 13 | Update password in RDS | Prisma → RDS | `UPDATE users SET passwordHash='bcrypt...$2b$...', resetToken=NULL, resetTokenExpiry=NULL WHERE id=1` | RDS update, token cleared |
| 14 | Success notification | Browser | Admin sees "Password reset successful" | UI feedback |
| 15 | Admin logs in with new password | Journey 1, Step 2-3 | Admin can now log in with new password | Cognito validates new password |

### AWS Services Involved

- **API Gateway**: Route password reset endpoints
- **Lambda or EC2**: Orchestrate reset flow
- **RDS PostgreSQL**: Store reset tokens, hashed passwords
- **SES (Simple Email Service)**: Send password reset email

---

## Key Insights: Why Each Service?

### Cognito (Authentication)
- **Why**: AWS-managed authentication service
- **When**: Every admin login
- **Alternative**: Payload CMS native auth (simpler, less to learn)
- **Learning value**: Understand AWS identity services, JWTs, token validation

### API Gateway (Entry Point)
- **Why**: Expose REST APIs to internet, rate limiting, CORS headers
- **When**: Every user request (admin + public)
- **Data flow**: Routes requests to Lambda or EC2 backend
- **Learning value**: How internet traffic routes to private AWS services

### Lambda or EC2 (Application)
- **Why**: Run Payload CMS application code
- **Lambda**: Serverless, scales automatically, pays per invocation
- **EC2**: Traditional server, always warm, better for persistent WebSockets
- **Learning value**: Understand serverless vs. traditional compute

### RDS PostgreSQL (Database)
- **Why**: Relational database for structured data (posts, users, categories)
- **Prisma ORM**: Automatically handles SQL generation, joins, transactions
- **When**: Every read/write operation
- **Learning value**: Understand relational queries, ORMs, why databases exist

### S3 (Media Storage)
- **Why**: Scalable object storage for images/files
- **Presigned URLs**: Allow browser to upload directly without backend relay
- **CloudFront**: Cache images for fast delivery
- **When**: Admin uploads images, public viewers load images
- **Learning value**: Understand how media is decoupled from database

### CloudFront (CDN)
- **Why**: Cache content closer to users, faster load times
- **What's cached**: Admin UI assets, API responses, images
- **When**: First request goes to origin (S3, Lambda), subsequent requests served from edge
- **Learning value**: Understand caching layers, cache invalidation

### VPC (Networking)
- **Why**: Private networking between services
- **What's isolated**: EC2/Lambda and RDS are private, only accessible via security groups
- **When**: EC2 needs to connect to RDS without going through internet
- **Learning value**: Understand network segmentation, security groups

### IAM (Permissions)
- **Why**: Control who can access what
- **Examples**: 
  - Admin can write posts, public cannot
  - Lambda can read/write S3, users cannot
  - Cognito can issue JWTs
- **When**: Every API call checks IAM permissions
- **Learning value**: Understand least-privilege principle

---

## Data Relationships (Prisma Schema)

```
User (Admin)
  └─ has many Posts
  
Post
  ├─ belongs to Author (User)
  ├─ has many Categories (many-to-many)
  └─ has many Tags (many-to-many)
  
Category
  └─ has many Posts (many-to-many)
  
Tag
  └─ has many Posts (many-to-many)
```

**Why relational?**
- Admin creates multiple posts
- Post can have multiple categories/tags
- Categories and tags can have multiple posts
- Prisma handles all joins automatically

---

## Caching Strategy

| What | Where | How Long | Why |
|------|-------|----------|-----|
| Admin UI assets (HTML, CSS, JS) | CloudFront | 1 hour | Change rarely; but if updated, need to eventually propagate |
| Featured images | CloudFront | 1 day | Images don't change once uploaded |
| API responses (posts list) | CloudFront | 5 minutes | Posts published/updated, so cache shorter than images |
| API response (single post) | CloudFront | 5 minutes | Ditto |
| Admin dashboard data | Browser cache (localStorage) | Until logout | Admin-specific, not cached by CDN |

---

## Error Cases (Not Detailed, But Worth Knowing)

1. **Admin tries to upload 1GB image**: S3 returns error; Lambda validates; frontend shows error
2. **Network drops during image upload**: Presigned URL expires (15 min); admin must retry
3. **RDS connection lost**: Prisma reconnects automatically; if timeout exceeds 5s, Lambda returns 503
4. **Admin password reset token expired**: Lambda rejects; user sees "link expired, please retry"
5. **Concurrent edits**: One admin publishes, another is still editing same post; Prisma handles via `updatedAt` timestamps or explicit locking

---

## Next Steps: Multi-Cloud Comparison

This journey is AWS-specific. In a future document, we can show how the same journey runs on **GCP** or **Azure**:

| Service | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Auth | Cognito | Cloud Identity | Azure AD |
| API Gateway | API Gateway | Cloud API Gateway | Application Gateway |
| Compute | Lambda/EC2 | Cloud Run/Compute Engine | Functions/VMs |
| Database | RDS PostgreSQL | Cloud SQL | Azure Database for PostgreSQL |
| Media Storage | S3 | Cloud Storage | Blob Storage |
| CDN | CloudFront | Cloud CDN | Azure CDN |

The logic stays the same; service names change.

---

## Summary

This document maps **real user journeys** to **AWS services**:

1. **Admin publishes**: Cognito → API Gateway → Lambda → RDS (Prisma) → S3 → CloudFront
2. **Public reads**: CloudFront (cached) → RDS (joins) → S3 images
3. **Password reset**: Cognito + SES + RDS

Each service has a purpose. Understanding *why* helps when building and troubleshooting.
