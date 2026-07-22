# Component 6 – Cross-User Isolation: Proving Data Security End-to-End

## Overview

The previous five components progressively built the architecture for a secure serverless Notes API.

Component 1 established a trusted authentication system using Amazon Cognito. Component 2 introduced API Gateway with a JWT Authorizer, ensuring every request was authenticated before reaching the backend. Component 3 designed a DynamoDB table capable of isolating user data through a composite primary key. Component 4 implemented the CRUD logic inside a single Lambda function, with every operation scoped to the authenticated user's identity. Finally, Component 5 connected every service together into a fully functioning REST API and verified that authenticated users could successfully create, retrieve, update and delete their own notes.

Although each of those components proved an important part of the architecture, they all relied on a single authenticated user. While the code strongly suggested that user isolation had been implemented correctly, there was still one important question left unanswered:

> **How can we prove that one authenticated user cannot access another user's data?**

This final component exists to answer that question.

Rather than reviewing the source code and assuming the design was secure, the application's security model was validated through practical testing using two completely independent authenticated users. A second Cognito account was created, authenticated through the same sign-in process as the original user, and then used to perform genuine attempts to access another user's data through the public API.

The purpose of these tests was not simply to observe failed requests. Instead, they were designed to demonstrate that user isolation is enforced structurally by the application's architecture rather than by a permission check that could potentially be forgotten during future development.

This component therefore serves as the final validation of the project, demonstrating that the combination of Amazon Cognito, API Gateway, Lambda and DynamoDB provides genuine end-to-end isolation between authenticated users.

---

# Architecture

No new AWS services were introduced during this component.

Instead, the existing architecture was exercised using two independent authenticated users interacting with the same deployed application.

```
                    Amazon Cognito
                           │
              Authenticates Independent Users
                           │
                  Issues JWT Access Tokens
                           │
                           ▼
                  Amazon API Gateway
                     JWT Authorizer
                           │
                           ▼
                 AWS Lambda (notesHandler)
                           │
                           ▼
                    Amazon DynamoDB
              Partition Key: userid
                 Sort Key: noteid
```

Both users authenticate through the same Amazon Cognito User Pool and receive their own signed JWT Access Tokens.

Every request passes through the same JWT Authorizer before reaching the Lambda function, where the authenticated user's identity is extracted from the verified JWT claims.

Finally, every database operation is executed against the same DynamoDB table, with the authenticated user's Cognito `sub` claim used as the partition key.

This architecture allows multiple users to share the same infrastructure while ensuring that each user's data remains isolated.

---

# Objectives

The objectives of this component were to:

- Create a second independent Cognito user.
- Authenticate both users using genuine JWT Access Tokens.
- Verify that each user can successfully create and retrieve their own notes.
- Attempt to update another user's note.
- Attempt to delete another user's note.
- Confirm that cross-user requests are rejected.
- Demonstrate that isolation is enforced by the application's data model rather than by application-level permission checks.
- Provide practical evidence that answers the project's central interview question:

> **"How do you prevent User A from accessing User B's data?"**

---

# AWS Services Used

| Service | Purpose |
|---------|---------|
| **Amazon Cognito** | Authenticate two independent users and issue JWT Access Tokens. |
| **Amazon API Gateway** | Validate JWT Access Tokens before forwarding requests to Lambda. |
| **JWT Authorizer** | Reject unauthenticated requests before they reach the application code. |
| **AWS Lambda** | Execute CRUD operations using the authenticated caller's identity extracted from verified JWT claims. |
| **Amazon DynamoDB** | Store user notes using a composite primary key (`userid` + `noteid`) that naturally separates each user's data. |
| **AWS CloudShell** | Authenticate users, generate JWT Access Tokens and execute HTTP requests against the deployed API. |

---

# Why This Component Matters

Throughout the project, one design principle has remained consistent:

**The application must never trust information supplied by the client to determine who owns a piece of data.**

Instead, every CRUD operation derives the authenticated user's identity directly from the verified JWT Access Token provided by Amazon Cognito.

While this approach appears secure when reviewing the source code, secure software should not rely solely on assumptions.

It should be tested.

For this reason, the final component focuses entirely on validation rather than implementation.

Instead of adding new functionality, the existing architecture is challenged using realistic attack scenarios designed to answer the following questions:

- Can one authenticated user retrieve another user's notes?
- Can one authenticated user modify another user's data?
- Can one authenticated user delete another user's records?
- Does the API leak the existence of another user's resources?

If any of these scenarios succeeded, the application's security model would be fundamentally flawed.

Passing these tests therefore provides significantly stronger evidence than simply reading the implementation.

---

# Preparing the Test Environment

Before beginning the cross-user isolation tests, the original authenticated user (User A) required a fresh note that could be used as the target for later attack scenarios.

The note created during Component 5 had already been deleted during the CRUD verification tests, meaning a new record first had to be created.

Because Cognito Access Tokens expire after approximately one hour, a fresh Access Token was generated before any requests were made.

As in previous components, authentication was performed using the AWS CLI's `initiate-auth` command, with the response captured programmatically rather than copied manually from the terminal. The returned JSON was then parsed using `jq` to extract the Access Token into a shell variable.

This approach avoids the token truncation issues encountered earlier in the project when manually copying long JWT strings from the terminal, ensuring that subsequent requests always use a complete, valid token. :contentReference[oaicite:0]{index=0}

Once authentication was complete, User A successfully submitted a new note through the public API.

The API returned an HTTP **201 Created** response together with a newly generated note identifier, the authenticated user's Cognito identifier and the submitted content.

This note would become the target for every cross-user access attempt performed later in the component.

With the target resource prepared, the environment was ready for the introduction of a second independent user.

# Creating a Second User

With User A's test note successfully created, the next stage was to introduce a completely independent authenticated user into the application.

The purpose of this second account was to simulate a real-world scenario where multiple users interact with the same API while expecting their data to remain completely private.

Rather than creating a mock identity or manually inserting records into DynamoDB, a second user was created through the existing Amazon Cognito User Pool. This ensured the new account followed exactly the same authentication process as every legitimate user of the application.

Creating a genuine Cognito user was particularly important because it meant every subsequent request would originate from a completely independent authenticated identity rather than from manually modified request data.

This approach allowed the isolation tests to accurately reflect how the application would behave in a production environment.

---

# Authenticating User B

Like the original account created during Component 1, the newly created user initially received a temporary password.

When attempting to authenticate for the first time, Cognito did not immediately issue JWT tokens. Instead, it returned a `NEW_PASSWORD_REQUIRED` challenge, requiring the temporary password to be replaced before authentication could be completed. This behaviour is the default security mechanism used by Amazon Cognito whenever a user is created with a temporary password. :contentReference[oaicite:0]{index=0}

Authentication was completed by responding to the challenge and providing a permanent password.

Once the challenge had been satisfied, Cognito returned a genuine authentication response containing:

- An Access Token
- An ID Token
- A Refresh Token
- A one-hour Access Token lifetime

At this point, User B had become a fully authenticated user of the application in exactly the same way as User A.

Most importantly, Cognito generated a completely different `sub` claim for the new account.

Although both users authenticate against the same User Pool, every account receives its own globally unique identifier.

This identifier is the foundation of the application's security model because it becomes the value used as the DynamoDB partition key throughout every CRUD operation.

The existence of two different `sub` values meant the application could now perform realistic cross-user testing.

---

# Verifying the New Identity

Before interacting with the API, the newly issued Access Token was verified using the same process adopted throughout the project.

Rather than manually copying the JWT from the terminal, the authentication response was parsed using `jq`, allowing the Access Token to be extracted directly into a shell variable.

A simple sanity check was then performed by confirming:

- the token began with the expected JWT prefix,
- the token length matched that of a complete Cognito Access Token,
- and the authentication request had completed successfully.

Using this automated approach eliminated the copy-and-paste issues encountered earlier in the project, where incomplete JWTs resulted in API Gateway rejecting requests before they ever reached the backend. :contentReference[oaicite:1]{index=1}

With a verified Access Token now available, User B was ready to interact with the deployed API.

---

# Creating User B's First Note

To prove that both users could independently use the application, User B first created their own private note through the public API.

An authenticated HTTP POST request was sent to the existing `/notes` endpoint using User B's JWT Access Token.

Unlike the previous requests made by User A, this request represented a completely different authenticated identity while interacting with exactly the same backend infrastructure.

The API responded with **HTTP 201 Created**, returning:

- a newly generated `noteId`,
- User B's authenticated `userid`,
- the submitted content,
- and the creation timestamp.

Although the request was identical to the one previously made by User A, the returned `userid` differed completely.

This demonstrated that the Lambda function was not relying on any value supplied by the client.

Instead, ownership of every newly created note was derived automatically from the authenticated Cognito identity contained within the verified JWT Access Token.

As a result, User B's note was automatically stored inside a different DynamoDB partition from User A's note, despite both records existing within the same table.

No additional permission logic or ownership checks were required.

The table design itself naturally separated the two users' data.

---

# Establishing the Baseline

Before attempting any cross-user operations, it was important to confirm that both users could successfully access their own data.

Using User A's Access Token, an authenticated request was sent to the `GET /notes` endpoint.

The response returned **HTTP 200 OK**, containing exactly one record—the note that had been created earlier during the preparation stage.

Importantly, the response contained no trace of User B's note.

The API returned only the records associated with User A's authenticated Cognito identity.

This established the expected baseline behaviour for the remainder of the testing.

Each authenticated user could successfully create and retrieve their own notes while interacting with the same deployed API and the same DynamoDB table.

The next stage would deliberately attempt to break this isolation by using User B's credentials to modify and delete User A's note.

If the application's architecture had been implemented incorrectly, these requests could potentially succeed.

Instead, they would provide the strongest evidence yet that the application's security model enforced user isolation at the data layer rather than relying on application-level permission checks.

# Executing the Cross-User Isolation Tests

With two independent Cognito users successfully authenticated and each possessing their own private note, the testing environment was now complete.

The remaining objective was to determine whether one authenticated user could interact with another user's data through the public API.

Rather than inspecting the application's source code, these tests were performed against the deployed production architecture using genuine HTTP requests. This approach validated the behaviour of the complete application stack, including Amazon Cognito, API Gateway, Lambda and DynamoDB, exactly as a real client would experience it.

If the application had been implemented incorrectly—for example by trusting a user identifier supplied by the client or querying DynamoDB using only the note identifier—these requests might have succeeded.

Instead, each scenario was designed to verify that user isolation remained intact regardless of what requests were made.

---

# Cross-User Update Attempt

The first security test involved attempting to modify User A's note while authenticated as User B.

To perform this test, User B sent an authenticated HTTP PUT request to the existing `/notes/{noteId}` endpoint.

The request deliberately used:

- User B's valid JWT Access Token.
- User A's valid `noteId`.
- New note content intended to replace User A's existing note.

From the client's perspective, this represented a genuine attempt to overwrite another user's data.

The request itself was completely valid.

The JWT had been successfully issued by Amazon Cognito.

API Gateway accepted the token and forwarded the request to Lambda.

The supplied note identifier genuinely existed within the DynamoDB table.

Despite all of this, the API responded with:

```http
HTTP/1.1 404 Not Found
```

```json
{
    "error": "Note not found"
}
```

No data was modified.

User A's original note remained unchanged.

This result immediately demonstrated that possession of another user's `noteId` alone was insufficient to gain access to their data.

Ownership was determined exclusively by the authenticated Cognito identity rather than by any information supplied within the request itself.

---

# Cross-User Delete Attempt

The second security test repeated the same scenario using the DELETE operation.

Once again, User B remained fully authenticated with a valid Access Token.

This time, an authenticated HTTP DELETE request was sent to the `/notes/{noteId}` endpoint using User A's known note identifier.

As before, every part of the request was technically valid.

- The endpoint existed.
- The JWT was authentic.
- The request successfully passed API Gateway's JWT Authorizer.
- The note identifier belonged to a real record stored within DynamoDB.

However, the response was identical to the previous test.

```http
HTTP/1.1 404 Not Found
```

```json
{
    "error": "Note not found"
}
```

The delete operation failed.

User A's note remained safely stored within the database.

At no point was another user's data exposed or modified.

The application simply behaved as though the requested resource did not exist.

---

# Confirming User B's Data Remained Unaffected

Following the failed update and delete attempts, a final verification was performed using User B's own account.

An authenticated GET request was sent to the `/notes` endpoint using User B's Access Token.

The API returned:

```http
HTTP/1.1 200 OK
```

along with exactly one note—the note previously created by User B.

No records belonging to User A appeared within the response.

This final verification confirmed two important behaviours.

First, User B's own data remained completely accessible following the failed attack attempts.

Second, User A's data remained entirely invisible to User B despite both users interacting with the same deployed application.

Each user continued to operate only within their own isolated dataset.

---

# Why These Tests Matter

At first glance, these requests may simply appear to demonstrate failed API calls.

In reality, they provide the strongest validation of the project's overall security architecture.

Every previous component was designed to build towards this exact moment.

Component 1 established a trusted identity using Amazon Cognito.

Component 2 ensured that API Gateway only accepted requests carrying valid JWT Access Tokens.

Component 3 designed the DynamoDB table so that every record belonged to a specific authenticated user.

Component 4 implemented CRUD operations that always derived ownership from verified JWT claims rather than client input.

Component 5 connected every service into a fully functioning authenticated REST API.

This component finally proved that all of those individual design decisions work together exactly as intended.

Rather than relying on theoretical reasoning or code inspection, the application's security model was verified through realistic end-to-end testing using two independently authenticated users.

---

# Initial Observations

One result immediately stood out during testing.

Neither of the cross-user requests returned an **HTTP 403 Forbidden** response.

Instead, both operations returned:

```http
HTTP/1.1 404 Not Found
```

This distinction is extremely important.

A **403 Forbidden** response would suggest that the application first located another user's resource before performing an additional permission check to deny access.

Instead, the **404 Not Found** response indicates something fundamentally different.

The requested record was never found in the first place.

Although the supplied `noteId` genuinely existed within the DynamoDB table, it did not exist within the authenticated user's partition.

From User B's perspective, that resource simply did not exist.

This subtle difference provides the strongest evidence yet that user isolation is enforced structurally through the database design rather than by an application-level permission check.

The final section explores this behaviour in detail and explains why the combination of verified JWT claims and DynamoDB's composite primary key naturally prevents one authenticated user from accessing another user's data. :contentReference[oaicite:0]{index=0}

# Understanding Why the Isolation Works

Successfully preventing one user from modifying another user's data is important.

Understanding *why* those requests failed is even more important.

Without understanding the underlying mechanism, it would be easy to assume that the application simply performed an additional permission check before denying access.

In reality, the application's architecture prevents cross-user access much earlier.

Rather than locating another user's data and then deciding whether access should be allowed, the database lookup itself is scoped to the authenticated user's identity.

This distinction is what makes the application's security model significantly stronger.

---

# Authentication and Authorization

One of the key concepts reinforced throughout this project is the difference between **authentication** and **authorization**.

Although these terms are often used together, they solve two different problems.

## Authentication

Authentication answers the question:

> **Who is making this request?**

In this project, authentication is handled entirely by Amazon Cognito.

When a user successfully signs in, Cognito issues a digitally signed JWT Access Token.

That token contains several claims, including the user's unique `sub` identifier.

Because the JWT has already been cryptographically signed by Cognito, neither the client nor the Lambda function can modify its contents.

Every request therefore arrives with a trusted identity that API Gateway can validate before forwarding the request.

---

## Authorization

Authorization answers a different question:

> **What is this authenticated user allowed to access?**

API Gateway performs the first stage of authorization by validating:

- the JWT signature,
- the token issuer,
- the configured audience,
- and the token expiry.

Only after all of these checks succeed is the request forwarded to the Lambda function.

The Lambda function therefore never needs to verify the JWT itself.

Instead, it simply reads the verified claims that API Gateway has already attached to the request context.

This architecture keeps authentication and authorization responsibilities clearly separated while avoiding duplicated JWT validation logic throughout the application. :contentReference[oaicite:0]{index=0}

---

# Never Trusting Client-Supplied Identity

One of the most important security decisions made throughout the project was ensuring that the application never trusted identity information supplied by the client.

The API never accepts a `userid` within the request body.

Likewise, it never allows the client to specify which user's records should be queried.

Instead, every CRUD operation begins by extracting the authenticated user's identity directly from the verified JWT claims.

Conceptually, the Lambda function always performs the equivalent of:

```python
user_id = event["requestContext"]["authorizer"]["jwt"]["claims"]["sub"]
```

This value originates from Amazon Cognito rather than from the client.

As a result, there is no opportunity for a malicious user to impersonate another account simply by modifying a request body or changing a parameter.

Every database operation is automatically tied to the identity that API Gateway has already authenticated.

---

# How DynamoDB Enforces User Isolation

The most important design decision in the entire project was made back in Component 3, when the DynamoDB table was created.

Rather than using a single primary key, the Notes table was designed with a composite primary key:

| Attribute | Purpose |
|----------|---------|
| **Partition Key (`userid`)** | Identifies the authenticated owner of every note. |
| **Sort Key (`noteid`)** | Uniquely identifies each individual note belonging to that user. |

This design means every note stored within DynamoDB belongs to exactly one authenticated user.

Instead of searching an entire table and filtering the results afterwards, every query begins inside the authenticated user's own partition.

That architectural decision provides two major benefits.

First, retrieving a user's notes becomes significantly more efficient because DynamoDB only searches a single partition.

Second, and far more importantly, another user's partition is never queried in the first place.

This is the structural guarantee that underpins the entire application's security model. :contentReference[oaicite:1]{index=1}

---

# Why the Cross-User Requests Returned 404

The most interesting outcome of the testing was not that the update and delete requests failed.

It was *how* they failed.

Both requests returned:

```http
HTTP/1.1 404 Not Found
```

rather than:

```http
HTTP/1.1 403 Forbidden
```

Although these responses may appear similar, they represent two very different implementation strategies.

A **403 Forbidden** response would suggest the following sequence:

1. The application successfully located User A's note.
2. It recognised that the note existed.
3. It checked whether User B owned the note.
4. It denied the operation.

In other words, the application would already know another user's record existed before refusing access.

That is **not** what happened here.

Instead, the application constructed its DynamoDB lookup using two values:

- User B's authenticated `userid` (derived from the verified JWT).
- User A's `noteId` supplied in the URL.

Conceptually, the lookup became:

```
userid = User B

noteid = User A's note
```

No such record exists within DynamoDB.

Although the `noteId` is valid, it belongs to an entirely different partition.

As a result, DynamoDB simply returns no matching item.

From the application's perspective, the requested resource genuinely does not exist.

The Lambda function therefore returns **404 Not Found** rather than **403 Forbidden**.

This behaviour demonstrates that user isolation is enforced by the table's key structure itself rather than by a separate permission check layered on afterwards. :contentReference[oaicite:2]{index=2}

---

# Why Two Real Users Were Necessary

One of the goals established at the beginning of this project was to avoid relying on assumptions.

It would have been easy to claim that the application was secure simply because the Lambda code always extracted the authenticated user's `sub` claim.

Instead, two completely independent Cognito users were created and authenticated.

Each user received:

- their own Cognito identity,
- their own JWT Access Token,
- their own notes,
- and their own isolated DynamoDB partition.

The subsequent testing proved that neither user could retrieve, modify or delete the other user's records through the deployed API.

This provides considerably stronger evidence than simply explaining how the code was intended to behave.

The security model was validated through realistic end-to-end testing using the complete production architecture.

---

# Skills Demonstrated

Completing this final component demonstrated knowledge across several core AWS and security concepts, including:

- Implementing secure authentication using Amazon Cognito.
- Using API Gateway JWT Authorizers to protect HTTP APIs.
- Separating authentication from authorization.
- Designing DynamoDB tables using composite primary keys.
- Structuring serverless applications around authenticated identities.
- Preventing horizontal privilege escalation through data-layer isolation.
- Performing end-to-end security testing using multiple authenticated users.
- Validating architectural assumptions through practical testing rather than code inspection alone.

---

# Project Outcome

This component completes Project 2 by providing practical evidence that the application's security architecture behaves exactly as intended.

Two independently authenticated users successfully interacted with the same deployed REST API while remaining completely isolated from one another.

Each user could create, retrieve, update and delete their own notes, but every attempt to modify another user's data failed without exposing the existence of that data.

Rather than relying on application-level ownership checks, user isolation is enforced through a combination of verified JWT claims, API Gateway authorization and DynamoDB's composite primary key design.

This directly answers the project's original design objective:

> **One authenticated user cannot access another user's data because every database operation is scoped using the authenticated Cognito identity, not client-supplied information.**

With this final validation complete, Project 2 demonstrates a complete production-style serverless architecture that combines secure authentication, centralized authorization, least-privilege access and structural data isolation using fully managed AWS services. :contentReference[oaicite:3]{index=3}
