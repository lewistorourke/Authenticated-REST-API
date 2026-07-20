# Authenticated-REST-API
# Serverless Authenticated Notes API

A fully serverless CRUD Notes API built on AWS that demonstrates authentication, authorization, least-privilege IAM, API design, and secure multi-user data isolation.

Rather than simply allowing authenticated users to create notes, this project demonstrates **how AWS services work together to enforce security**. Authentication is handled by Amazon Cognito, authorization is enforced by API Gateway using JWT validation, and every database operation is scoped to the authenticated user's identity, preventing one user from accessing another user's data.

---

## Architecture

```text
                ┌──────────────────────┐
                │    Browser / Client  │
                └──────────┬───────────┘
                           │
                    Login / JWT
                           │
                           ▼
                 Amazon Cognito User Pool
                           │
                    Access Token (JWT)
                           │
                           ▼
              API Gateway HTTP API
             (JWT Authorizer validates token)
                           │
                           ▼
                  Lambda (notesHandler)
                           │
             Reads user ID from JWT claims
                           │
                           ▼
                 Amazon DynamoDB
        Partition Key: userid
        Sort Key: noteid
```

---

# AWS Services Used

- Amazon Cognito
- Amazon API Gateway (HTTP API)
- AWS Lambda
- Amazon DynamoDB
- AWS IAM
- AWS CloudWatch Logs

---

# Project Objectives

The goals of this project were to demonstrate:

- Secure user authentication
- JWT-based authorization
- Least-privilege IAM
- REST API design
- CRUD operations
- Serverless architecture
- User-level data isolation
- Secure handling of client identity
- Real-world debugging and troubleshooting

---

# Features

- User registration and authentication using Amazon Cognito
- JWT Access Tokens
- API Gateway JWT Authorizer
- Create Notes
- View Notes
- Update Notes
- Delete Notes
- Per-user data isolation
- Fully serverless architecture
- CloudWatch logging
- Least privilege IAM policies

---

# Architecture Decisions

Several deliberate design decisions were made throughout the build.

## Authentication handled by Cognito

Rather than implementing password storage, hashing, JWT signing and token rotation manually, Amazon Cognito provides a fully managed authentication service.

API Gateway is able to validate Cognito-issued JWTs before requests ever reach Lambda.

---

## Authorization at API Gateway

The Lambda function never validates JWTs itself.

Instead, API Gateway performs:

- Signature validation
- Issuer validation
- Audience validation
- Token expiry validation

Only requests with valid JWTs are forwarded to Lambda.

This removes duplicated authentication code from every Lambda function and centralises security.

---

## Single Router Lambda

Rather than creating one Lambda function per endpoint, the application uses a single router-style Lambda.

The handler branches internally based on the HTTP method.

Benefits include:

- Less duplicated code
- Easier maintenance
- Shared validation logic
- Shared response formatting
- Simpler deployment

---

## DynamoDB Table Design

The Notes table uses a composite primary key.

| Key | Purpose |
|------|----------|
| userid | Groups every user's notes together |
| noteid | Makes every note unique within that user |

This allows efficient Query operations while naturally isolating each user's data.

---

## Identity Never Comes From The Client

One of the most important security decisions in this project is that the client never tells the API who they are.

Instead:

```
JWT
    ↓
API Gateway validates token
    ↓
Lambda reads:
event.requestContext.authorizer.jwt.claims.sub
```

That verified Cognito user ID becomes the partition key for every DynamoDB operation.

The request body never contains a user ID.

---

## Least Privilege IAM

The Lambda execution role only receives the permissions it actually requires.

Example:

- dynamodb:PutItem
- dynamodb:GetItem
- dynamodb:Query
- dynamodb:UpdateItem
- dynamodb:DeleteItem

scoped only to the Notes table.

No wildcard permissions were used.

---

# Security Considerations

This project intentionally demonstrates several AWS security principles.

- JWT authentication handled by Cognito
- Authorization enforced by API Gateway
- Least privilege IAM
- No client-supplied identity
- No wildcard IAM permissions
- CloudWatch logging
- Partition-key based data isolation
- Public SPA client (no client secret)

---

# Cross-User Isolation

The primary objective of the project was proving that authenticated users cannot access each other's data.

Two completely separate Cognito users were created.

Testing confirmed:

✅ User A can CRUD their own notes.

✅ User B can CRUD their own notes.

✅ User B cannot update User A's notes.

✅ User B cannot delete User A's notes.

✅ User B cannot list User A's notes.

Attempting to access another user's note returns:

```text
404 Not Found
```

rather than

```text
403 Forbidden
```

This is intentional.

The lookup itself never succeeds because every DynamoDB operation uses:

```text
userid = JWT sub
```

as part of the primary key.

No separate ownership check is required.

---

# Lessons Learned

During this project I learned:

- Amazon Cognito authentication
- JWT validation
- API Gateway JWT Authorizers
- HTTP APIs vs REST APIs
- IAM least privilege
- Resource-based vs identity-based policies
- DynamoDB composite key design
- Query vs Scan
- Lambda routing
- CloudWatch debugging
- API debugging with curl
- JWT inspection
- Secure multi-user application design
- Cross-user isolation techniques

---

# Repository Structure

```
.
├── Component-1-Cognito-User-Pool
├── Component-2-JWT-Authorizer
├── Component-3-DynamoDB
├── Component-4-Lambda
├── Component-5-API-Routes
├── Component-6-Cross-User-Isolation
└── README.md
```

Each component contains its own detailed README documenting:

- Objectives
- Configuration
- Design decisions
- Verification
- Security considerations
- Lessons learned

---

# Future Improvements

Possible production improvements include:

- Infrastructure as Code (Terraform / CloudFormation)
- CI/CD pipeline
- Refresh token authentication
- Password reset flow
- Input sanitisation
- Rate limiting
- Custom domain
- HTTPS certificate
- CloudFront frontend
- Request validation
- Unit testing
- API documentation with OpenAPI

---

# Key AWS Skills Demonstrated

- Amazon Cognito
- API Gateway HTTP APIs
- JWT Authorizers
- AWS Lambda
- DynamoDB
- IAM
- CloudWatch
- Secure REST API design
- Serverless architecture
- Authentication & Authorization
- Least Privilege IAM
- CRUD APIs
- Secure Multi-user Applications

---

# Build Log

A full build log documenting every component, design decision, verification step, and troubleshooting process is included within the repository. :contentReference[oaicite:0]{index=0}

Each component can also be explored individually through its own dedicated README.
