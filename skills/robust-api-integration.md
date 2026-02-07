# Robust API Integration Pattern

## Context

When integrating with external systems (like FAST API or Payments), failures are expected (network issues, logic errors, validation updates). Throwing exceptions (e.g., `BadRequestException`) immediately upon failure causes data loss if the calling service expects to log the result.

## The Skill

**Design integration methods to return a standardized Result Object instead of throwing exceptions.**

### 1. The Result Object Pattern

Define a standard return interface for all integration methods:

```typescript
interface ServiceResult<T = any> {
  status: number; // 1: Success, 0: Failure
  message: string; // Readable message
  result?: T; // The actual payload/data
  fastApiResponse?: any; // The raw external response (for auditing)
}
```

### 2. Implementation Rules

- **Do NOT throw errors** for expected operational failures (e.g., API returned "Duplicate Invoice").
- **Catch errors locally** within the integration service (try-catch block).
- **Return the full context**: Always return the raw response from the external API, even if it failed.

### 3. Example (From FastApiInvoiceFlowService)

**BAD (Throwing):**

```typescript
async createOrder(data) {
  const res = await api.post(data);
  if (res.status !== 1) {
    throw new BadRequestException("Failed"); // Response body is LOST here
  }
  return res.data;
}
```

**GOOD (Returning Result):**

```typescript
async createOrder(data): Promise<any> {
  try {
    const res = await api.post(data);
    // Return raw response regardless of status
    return res.data;
  } catch (error) {
    // Wrap unexpected network errors in a standard failure object
    return {
      status: 0,
      message: error.message,
      data: null
    };
  }
}
```

### 4. Auditing

The calling service (`SalesInvoiceService`) can now safe-guard this data:

```typescript
const result = await this.apiService.createOrder(data);
// Save to DB regardless of success/failure
await this.auditRepo.save({
  docCode: data.docCode,
  status: result.status,
  responsePayload: JSON.stringify(result), // Full trace available!
});
```

## Benefits

- **No Data Loss**: You see exactly _why_ it failed in your database audit logs.
- **Resilience**: One failed item in a loop doesn't crash the entire batch process.
- **Control**: The business layer decides whether to retry, ignore, or alert.
