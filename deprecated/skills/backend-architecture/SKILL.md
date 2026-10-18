---
name: backend-architecture
description: Backend architecture enforcement for Node.js, Python, and Ruby on Rails. Enforces Controller/Service/Lib/Guard layering.
metadata:
  priority: 10
  pathPatterns:
    - 'app/controllers/**'
    - 'app/models/**'
    - 'app/services/**'
    - 'app/lib/**'
    - 'app/guards/**'
    - 'app/policies/**'
    - 'controllers/**'
    - 'services/**'
    - 'lib/**'
    - 'guards/**'
    - 'routes/**'
    - 'src/controllers/**'
    - 'src/services/**'
    - 'src/lib/**'
    - 'src/guards/**'
    - 'src/routes/**'
    - 'src/middleware/**'
    - '*_controller.rb'
    - '*_controller.ts'
    - '*_controller.js'
    - '*_controller.py'
    - '*_service.rb'
    - '*_service.ts'
    - '*_service.js'
    - '*_service.py'
    - 'app.rb'
    - 'app.ts'
    - 'app.js'
    - 'server.rb'
    - 'server.ts'
    - 'server.js'
    - 'main.py'
    - 'Gemfile'
    - 'requirements.txt'
    - 'package.json'
  bashPatterns:
    - 'rails\s+(new|generate|g)\b'
    - 'rails\s+g\s+controller\b'
    - 'rails\s+g\s+scaffold\b'
    - 'express\b'
    - 'fastapi\b'
    - 'flask\b'
    - 'django\b'
    - 'nest\s+new\b'
---

# Backend Architecture Standard

Enforce this 4-layer architecture in every backend project (Node.js, Python, Ruby on Rails).

---

## Layer Rules

### 1. Controller — HTTP boundary only

**Responsibility**: Parse request, call guard, call service, return response.
**Must NOT**: contain business logic, DB queries, external HTTP calls, or unrelated conditionals.

```ts
// Node.js / TypeScript
export async function createUserController(req: Request, res: Response) {
  await isAuthenticatedGuard(req);
  const user = await createUserService(req.body);
  res.status(201).json(user);
}
```

```python
# Python (FastAPI)
@router.post("/users", status_code=201)
async def create_user(body: CreateUserDto, request: Request):
    await is_authenticated_guard(request)
    user = await create_user_service(body)
    return user
```

```ruby
# Ruby on Rails
class UsersController < ApplicationController
  def create
    authorize!
    user = UserService.create(user_params)
    render json: user, status: :created
  end

  private

  def user_params
    params.require(:user).permit(:name, :email)
  end
end
```

---

### 2. Service (or Use Case) — Business logic

**Responsibility**: Perform the operation. Pure business logic. Calls lib for external comms.
**Must NOT**: access `req`, `res`, HTTP headers, or status codes.

```ts
// Node.js
export async function createUserService(dto: CreateUserDto): Promise<User> {
  const existing = await db.user.findUnique({ where: { email: dto.email } });
  if (existing) throw new ConflictError('Email already in use');
  const hashed = await hashPassword(dto.password);
  const user = await db.user.create({ data: { ...dto, password: hashed } });
  await sendWelcomeEmailLib(user.email); // calls lib
  return user;
}
```

```python
# Python
async def create_user_service(dto: CreateUserDto) -> UserSchema:
    existing = await db.users.find_one({"email": dto.email})
    if existing:
        raise ConflictError("Email already in use")
    hashed = hash_password(dto.password)
    user = await db.users.insert_one({**dto.dict(), "password": hashed})
    await send_welcome_email_lib(dto.email)
    return UserSchema(**user)
```

```ruby
# Ruby on Rails
class UserService
  def self.create(params)
    raise ConflictError, 'Email already in use' if User.exists?(email: params[:email])
    user = User.create!(params.merge(password: hash_password(params[:password])))
    WelcomeEmailLib.send(user.email)
    user
  end
end
```

---

### 3. Lib — External service / tool communication

**Responsibility**: Wrap external HTTP calls, SDKs, or third-party tools. Handle connection errors.
**Must NOT**: contain business logic or know about HTTP request/response objects.

All outbound HTTP from lib files MUST use a traced HTTP client (see `http-outbound-tracing` skill).

```ts
// Node.js — lib/stripe_client.ts
import { createTracedClient } from './http_client';

const client = createTracedClient('https://api.stripe.com');

export async function createStripeChargeLib(amount: number, token: string) {
  const res = await client.post('/v1/charges', { amount, source: token });
  return res.data;
}
```

```python
# Python — lib/stripe_client.py
from lib.http_client import create_traced_client

async def create_stripe_charge_lib(amount: int, token: str):
    async with create_traced_client(base_url="https://api.stripe.com") as client:
        r = await client.post("/v1/charges", data={"amount": amount, "source": token})
        r.raise_for_status()
        return r.json()
```

```ruby
# Ruby on Rails — lib/stripe_client.rb
class StripeClient
  def self.create_charge(amount:, token:)
    conn = HttpClient.build('https://api.stripe.com')
    conn.post('/v1/charges', amount: amount, source: token)
  end
end
```

---

### 4. Guards — Authorization and pre-condition checks

**Responsibility**: Verify permissions, roles, or conditions before calling a service.
**Placement**: Called at the start of the controller, before any service call.
**Must NOT**: return business data. Only allow or raise/throw.

```ts
// Node.js — guards/is_authenticated.ts
export async function isAuthenticatedGuard(req: Request): Promise<void> {
  if (!req.headers.authorization) throw new UnauthorizedError('Missing token');
  const payload = verifyJwt(req.headers.authorization);
  if (!payload) throw new UnauthorizedError('Invalid token');
}
```

```python
# Python — guards/is_authenticated.py
async def is_authenticated_guard(request: Request) -> None:
    token = request.headers.get("Authorization")
    if not token:
        raise HTTPException(status_code=401, detail="Missing token")
    if not verify_jwt(token):
        raise HTTPException(status_code=401, detail="Invalid token")
```

```ruby
# Ruby on Rails — ApplicationController
class ApplicationController < ActionController::API
  def authorize!
    raise UnauthorizedError unless current_user
  end
end
```

---

## File Naming Conventions

| Layer | Node.js | Python | Rails |
|-------|---------|--------|-------|
| Controller | `src/controllers/users_controller.ts` | `app/controllers/users.py` | `app/controllers/users_controller.rb` |
| Service | `src/services/create_user_service.ts` | `app/services/create_user_service.py` | `app/services/user_service.rb` |
| Lib | `src/lib/stripe_client.ts` | `app/lib/stripe_client.py` | `lib/stripe_client.rb` |
| Guard | `src/guards/is_authenticated.ts` | `app/guards/is_authenticated.py` | `app/controllers/concerns/authorization.rb` |

---

## Violations — Never Do This

- Business logic inside a controller
- DB queries inside a controller
- `req` / `res` objects passed into a service
- Auth checks inside a service instead of a guard
- External HTTP calls without going through a lib
- Raw HTTP calls in lib without tracing (see `http-outbound-tracing` skill)
