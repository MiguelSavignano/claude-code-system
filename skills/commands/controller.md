When writing or modifying Django controller/view code in this project, always apply the decorator pattern from docs/refactors/decorators.md. Rules:

**Structure every endpoint as:**
```python
@csrf_exempt
@require_POST          # or @require_GET etc.
@validate_payload(SchemaClass)   # omit if no body
@inject_agent(lambda p: p.reference)  # omit if no agent needed
async def view_name(request, payload, agent):
    ...
```

**`validate_payload(schema)`** — defined in `apps/chats/decorators.py`
- Parses `request.body` with `schema.model_validate_json()`
- Returns 400 with `{'errors': e.errors()}` on `ValidationError`
- Injects validated `payload` as second argument

**`inject_agent(get_reference)`** — defined in `apps/chats/decorators.py`
- Calls `get_reference(payload)` to extract the reference field
- Returns 400 if missing, 404 if agent not found
- Injects resolved `agent` as third argument
- Uses `Agent.objects.filter(...).afirst()` (async ORM)

**Always:**
- Views must be `async def`
- Use async ORM: `acreate`, `aget`, `afirst`, `aupdate`, `adelete`
- Schemas live in `apps/chats/schemas.py` as Pydantic models
- No manual `json.loads(request.body)` or `request.POST` — let `validate_payload` handle it
- No Agent lookup inside the view body — let `inject_agent` handle it

**Example of a complete endpoint:**
```python
@csrf_exempt
@require_POST
@validate_payload(CreateConversationRequest)
@inject_agent(lambda p: p.reference)
async def create_conversation(request, payload, agent):
    conversation = await Conversation.objects.acreate(
        external_id=uuid.uuid4().hex,
        agent=agent,
        llm_identifier=settings.OPENAI_MODEL,
        country_code=request.country_code,
        support_phone=get_support_phone(request.country_code),
    )
    return JsonResponse({'conversation_id': conversation.external_id})
```

Before writing any controller, read the existing decorators in `apps/chats/decorators.py` to confirm their current signatures.
