# Backend Intern Technical Assessment

**Time Limit:** 90 minutes
**Tools Allowed:** AI assistants, documentation, any resources
**Submission:** JSON file following the schema at the end

---

## Section A: Debug Challenge (25 points)

This order processing endpoint has race conditions and bugs causing duplicate orders and incorrect inventory. Find all issues.

```typescript
// orders.ts - Express route handler
import { prisma } from '../lib/prisma';
import { sendEmail } from '../lib/email';

interface OrderItem {
  productId: string;
  quantity: number;
}

interface CreateOrderRequest {
  customerId: string;
  items: OrderItem[];
  paymentMethodId: string;
}

export async function createOrder(req: Request, res: Response) {
  const { customerId, items, paymentMethodId } = req.body as CreateOrderRequest;

  // Validate customer exists
  const customer = await prisma.customer.findUnique({ where: { id: customerId } });
  if (!customer) {
    return res.status(404).json({ error: 'Customer not found' });
  }

  // Check inventory and calculate total
  let total = 0;
  for (const item of items) {
    const product = await prisma.product.findUnique({ where: { id: item.productId } });
    if (!product) {
      return res.status(400).json({ error: `Product ${item.productId} not found` });
    }
    if (product.stock < item.quantity) {
      return res.status(400).json({ error: `Insufficient stock for ${product.name}` });
    }
    total += product.price * item.quantity;
  }

  // Process payment
  const paymentResult = await processPayment(paymentMethodId, total);
  if (!paymentResult.success) {
    return res.status(402).json({ error: 'Payment failed' });
  }

  // Create order
  const order = await prisma.order.create({
    data: {
      customerId,
      total,
      status: 'confirmed',
      paymentId: paymentResult.transactionId,
      items: {
        create: items.map(item => ({
          productId: item.productId,
          quantity: item.quantity,
        })),
      },
    },
  });

  // Update inventory
  for (const item of items) {
    await prisma.product.update({
      where: { id: item.productId },
      data: { stock: { decrement: item.quantity } },
    });
  }

  // Send confirmation email
  sendEmail(customer.email, 'Order Confirmed', `Your order ${order.id} is confirmed!`);

  res.json({ orderId: order.id, total });
}

async function processPayment(methodId: string, amount: number) {
  // External payment API call
  const response = await fetch('https://payments.example.com/charge', {
    method: 'POST',
    body: JSON.stringify({ methodId, amount }),
  });
  return response.json();
}
```

**Identify at least 5 issues. For each:**
1. The bug/vulnerability
2. A scenario where it fails (be specific)
3. The consequence (data corruption, money loss, etc.)
4. How to fix it properly

---

## Section B: API Design (25 points)

Design a REST API for a **file sharing service** with these requirements:

- Users can upload files (up to 100MB)
- Files can be shared via link (public or password-protected)
- Links can expire (configurable)
- Download count limit (optional)
- File owner can revoke access anytime
- Audit log of all downloads
- Rate limiting per user

**Provide:**

1. **Endpoint Design** - List all endpoints with methods, paths, request/response shapes
2. **Error Handling** - What error codes and when? How do you handle partial failures?
3. **Security Considerations** - Authentication, authorization, rate limiting approach
4. **Edge Cases** - What happens if:
   - File is deleted while someone is downloading?
   - User hits rate limit mid-upload?
   - Password-protected link password is changed while someone has the old link?

---

## Section C: Database Schema (25 points)

Design a schema for a **multi-tenant SaaS analytics platform**:

Requirements:
- Multiple organizations (tenants)
- Each org has users with roles (admin, analyst, viewer)
- Events are tracked (page views, clicks, custom events)
- Events have properties (JSON-like arbitrary data)
- Dashboards with widgets (charts, tables, metrics)
- Widgets query event data with filters and aggregations
- Data retention: 90 days for free tier, 1 year for paid

Constraints:
- Expect 10M+ events per day per large tenant
- Query patterns: time-series aggregations, funnel analysis, property breakdowns
- Multi-tenant isolation is critical

**Provide:**

1. **Schema Design** - Tables, columns, types, relationships (SQL or Prisma schema)
2. **Indexing Strategy** - What indexes and why?
3. **Partitioning Strategy** - How do you handle the volume?
4. **Trade-offs** - What did you optimize for? What did you sacrifice?
5. **Query Example** - Write a query for: "Get daily active users for org X in the last 30 days"

---

## Section D: Code Review (25 points)

Review this authentication middleware. Find issues.

```typescript
// auth.ts
import jwt from 'jsonwebtoken';
import { prisma } from '../lib/prisma';

const JWT_SECRET = process.env.JWT_SECRET || 'development-secret';

export async function authMiddleware(req: Request, res: Response, next: Function) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET) as { userId: string; role: string };
    req.user = decoded;

    // Check if user still exists and is active
    const user = await prisma.user.findUnique({
      where: { id: decoded.userId },
      select: { id: true, role: true, isActive: true },
    });

    if (!user || !user.isActive) {
      return res.status(401).json({ error: 'User not found or inactive' });
    }

    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}

export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: Function) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

export async function login(req: Request, res: Response) {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({ where: { email } });

  if (!user) {
    return res.status(401).json({ error: 'User not found' });
  }

  if (user.password !== password) {
    return res.status(401).json({ error: 'Invalid password' });
  }

  const token = jwt.sign(
    { userId: user.id, role: user.role },
    JWT_SECRET,
    { expiresIn: '7d' }
  );

  res.json({ token, user: { id: user.id, email: user.email, role: user.role } });
}

export async function register(req: Request, res: Response) {
  const { email, password, name } = req.body;

  const existing = await prisma.user.findUnique({ where: { email } });
  if (existing) {
    return res.status(400).json({ error: 'Email already registered' });
  }

  const user = await prisma.user.create({
    data: { email, password, name, role: 'user', isActive: true },
  });

  const token = jwt.sign(
    { userId: user.id, role: user.role },
    JWT_SECRET,
    { expiresIn: '7d' }
  );

  res.json({ token, user: { id: user.id, email: user.email } });
}
```

**Categories to consider:**
- Security vulnerabilities
- Authentication/authorization issues
- Error handling
- Performance concerns
- Best practices violations

**For each issue:**
1. Category
2. Severity (critical/high/medium/low)
3. The problem
4. Exploitation scenario or consequence
5. Fix

---

## Submission Format

Save your answers as `submission.json`:

```json
{
  "candidate": {
    "name": "Your Name",
    "email": "your.email@nsut.ac.in"
  },
  "section_a": {
    "issues": [
      {
        "bug": "description",
        "failure_scenario": "specific example",
        "consequence": "what goes wrong",
        "fix": "solution"
      }
    ]
  },
  "section_b": {
    "endpoints": [
      {
        "method": "POST",
        "path": "/api/...",
        "description": "...",
        "request": {},
        "response": {},
        "errors": []
      }
    ],
    "error_handling": "approach description",
    "security": "security considerations",
    "edge_cases": {
      "case1": "handling approach"
    }
  },
  "section_c": {
    "schema": "SQL or Prisma schema (as string)",
    "indexes": ["index descriptions"],
    "partitioning": "strategy description",
    "tradeoffs": "what you optimized for",
    "sample_query": "SQL query for DAU"
  },
  "section_d": {
    "issues": [
      {
        "category": "security|auth|error|performance|practices",
        "severity": "critical|high|medium|low",
        "problem": "description",
        "exploitation": "how it could be exploited",
        "fix": "solution"
      }
    ]
  }
}
```

---

## Evaluation Criteria

| Criteria | Weight |
|----------|--------|
| Security awareness | 30% |
| System design thinking | 25% |
| Practical debugging skills | 25% |
| Edge case handling | 15% |
| Communication clarity | 5% |

**What distinguishes great answers:**
- Identifies non-obvious issues (race conditions, edge cases)
- Considers real-world scale and failure modes
- Shows understanding of trade-offs
- Provides specific, actionable fixes
