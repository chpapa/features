# Web-hook Events

All web-hook events have the same format:
```json
{
    "id": "0E1E9537-DF4F-4AF6-8B48-3DB4574D4F24",
    "seq": 435,
    "type": "after_user_create",
    "payload": {
        // ...
    },
    "context": {
        // ...
    }
}
```

- `id`: string, ID of the web-hook event.
- `seq`: signed 64-bit integer, monotonic increasing event sequence number to
         establish event order.
- `type`: string, the type of the web-hook event.
- `payload`: object, the payload of the web-hook; it is specific to each event type.
- `context`: object, the context of the web-hook.

## Versioning
All fields are guaranteed that only backward-compatible changes would be made:
- Existing fields would not be removed or changed in meaning
- New fields may be added


## Context
Each web-hook event would have a context, it contains the environmental
information about the event:

- `timestamp`:
  signed 64-bit unix timestamp of when this event is generated; retried
  deliveries do not affect this field.
- `request_id`:
  ID of the HTTP request that generated this event; if the event is not
  generated by HTTP request, this field would be `null`.
- `user_id`:
  the ID of user performing the request that generated this event; if request
  is not authenticated or event is not generated by user action, this field
  would be `null`.
- `identity_id`:
  the ID of identity performing the request that generated this event; if
  request is not authenticated or event not generated by user action, this field
  would be `null`.

**NOTE**
- When user is performing authentication (e.g. signing up/logging in),
  the request will not be treated as authenticated with the user to
  sign up/log in, in both BEFORE/AFTER events.


## Mutations

For BEFORE events about a user, following fields can be mutated:
- `metadata`: user metadata
- `verify_info`: verify info of user
- `is_verified`: verified status of user
- `is_disabled`: disbled status of user

**NOTE**
- The mutations would not affect authorization of current operation. e.g.
  disabling user through mutations would not fail an operation requiring
  non-disabled users.
- `is_verified` is normally derived from `verify_info`. If it is mutated to be
   different than the derived value, it would be reset to the derived value
   when `verify_info` is updated.


## Event Types

### before_user_create, after_user_create

When a user is being created, e.g. sign up with password/SSO

```json
{
    "user": { /* a User object */ },
    "identities": [
        { /* an Identity object */ },
        { /* an Identity object */ }
    ]
}
```

### before_identity_create, after_identity_create

When an identity is being created for a existing user, e.g. adding login ID/
linking SSO accounts.

```json
{
    "user": { /* a User object */ },
    "identity": { /* an Identity object */ }
}
```

**NOTE**
- This event would not be generated for creation of user.
  To handle initial values of user, use `before/after_user_create` instead.

### before_identity_delete, after_identity_delete

When an identity is being deleted for a existing user, e.g. removing login ID/
unlinking SSO accounts.

```json
{
    "user": { /* a User object */ },
    "identity": { /* an Identity object */ }
}
```

### before_session_create, after_session_create

When a session is being created for a existing user, e.g. logging in with
password/SSO.

```json
{
    "reason": "signup",
    "user": { /* a User object */ },
    "identity": { /* an Identity object */ }
}
```

- `reason`: The reason for the creation of session, can be `signup` or `login`

**NOTE**
- The `last_login_at` field of user object would be the time last session is
  created, not the time this session is created.

### before_session_delete, after_session_delete

When a session is being deleted for a existing user, e.g. logging out

```json
{
    "reason": "logout",
    "user": { /* a User object */ },
    "identity": { /* an Identity object */ }
}
```

- `reason`: The reason for the deletion of session, can be `logout`

### before_user_update, after_user_update

When user attributes (metadata, disable status, verified status) is being
updated due to API operations.

```json
{
    "reason": "update_metadata",
    "is_disabled": true,
    "is_verified": true,
    "verify_info": { /* ... */ },
    "metadata": {},
    "user": { /* a User object */ }
}
```

- `reason`: The reason for the update of user, can be `update_metadata`,
            `update_identity`, `verification`, or `administrative`.
- `is_disabled`: The new disabled status;
                 if it is not changed, the field would be absent.
- `is_verified`: The new verified status;
                 if it is not changed, the field would be absent.
- `verify_info`: The new verify info;
                 if it is not changed, the field would be absent.
- `metadata`: The new metadata;
                 if it is not changed, the field would be absent.
- `user`: a snapshot of the user object before the operation

**NOTE**
- This event would not be generated for creation of user.
  To handle initial values of user, use `before/after_user_create` instead.

### before_password_update, after_password_update

When the password of a user is being updated.

```json
{
    "reason": "change_password",
    "user": { /* a User object */ }
}
```

- `reason`: The reason for the update of password, can be `change_password`,
            `reset_password`, or `administrative`
- `user`: a snapshot of the user object before the operation

**NOTE**
- This event would not be generated for creation of user.
  To handle initial values of user, use `before/after_user_create` instead.

### user_sync

When user state (user/identity/session) is potentially being updated.

```json
{
    "user": { /* a User object */ }
}
```

- `user`: the user object after the operation

**NOTE**
- The event would be generated unconditionally whenever a mutating operation is
  used; for example, disabling an already disabled user would still generate
  this event.
- If this event is generated by a session creation API, the `last_login_at`
  field of user object would be the time this session is created, unlike
  `session_create` events.


## Sample Web-hook Event

### before_user_create
```json
{
    "id": "A2BB162C-15EC-44A4-87D0-BF87354E1208",
    "seq": 50862,
    "type": "before_user_create",
    "payload": {
        "user": {
            "id": "20568C13-C72B-42EC-982C-C7C926830650",
            "created_at": "2019-07-08T11:12:44.317236Z",
            "last_login_at": "2019-07-11T10:47:36.684763Z",
            "is_verified": false,
            "is_disabled": false,
            "verify_info": {},
            "metadata": {}
        },
        "identities": [
            {
                "id": "9747C5CC-565E-4837-8414-0E10458F383A",
                "type": "password",
                "login_id_key": "email",
                "login_id": "test@example.com",
                "realm": "default",
                "claims": { "email": "test@example.com" }
            }
        ]
    },
    "context": {
        "timestamp": 1562922362,
        "request_id": "D6B65ABE-0AD6-4BD1-A85A-AC0DDEE92B17",
        "user_id": null,
        "identity_id": null
    }
}
```

### after_identity_create
```json
{
    "id": "D46EF8B6-4E30-4574-B7B5-D925A989AEFA",
    "seq": 50867,
    "type": "after_identity_create",
    "payload": {
        "is_user_creating": true,
        "identity": {
            "id": "9747C5CC-565E-4837-8414-0E10458F383A",
            "type": "password",
            "login_id_key": "email",
            "login_id": "test@example.com",
            "realm": "default",
            "claims": { "email": "test@example.com" }
        }
    },
    "context": {
        "timestamp": 1562922396,
        "request_id": "D6B65ABE-0AD6-4BD1-A85A-AC0DDEE92B17",
        "user_id": null,
        "identity_id": null
    }
}
```


## Sample Request Event Flows

### Signup with password
1. Transaction begin
2. User object is generated and sent to database
3. Identity objects are generated and sent to database
4. `before_user_create` event is triggered
5. Session token is generated and sent to database
6. `before_session_create` event is triggered
7. Transaction commit
8. `after_user_create`, `after_session_create` and `user_sync` events are triggered

### Signup with password (with metadata mutation)
1. Transaction begin
2. User object is generated and sent to database
3. Identity objects are generated and sent to database
4. `before_user_create` event is triggered
5. Session token is generated and sent to database
6. `before_session_create` event is triggered
7. Mutation is requested: metadata of user is updated
8. Transaction commit
9. `after_user_create`, `after_session_create` and `user_sync` events are triggered

### Signup with password (failed)
1. Transaction begin
2. User object is generated and sent to database
3. `before_user_create` event is triggered
4. Identity objects are generated and sent to database
5. FAILED: login ID is duplicated
6. Transaction rollback

### Signup with password (disallowed by web-hook handler)
1. Transaction begin
2. User object is generated and sent to database
3. `before_user_create` event is triggered
4. DISALLOWED: "metadata does not contain user address"
5. Transaction rollback

### Linking with OAuth accounts/Adding new login ID
1. Transaction begin
2. Identity object is generated and sent to database
3. `before_identity_create` event is triggered
4. Transaction commit
5. `after_identity_create` and `user_sync` event is triggered

### Unlinking OAuth accounts/Removing new login ID
1. Transaction begin
2. Identity object is deleted from database
3. `before_identity_delete` event is triggered
4. Transaction commit
5. `after_identity_delete` and `user_sync` event is triggered

### Disabling user
1. Transaction begin
2. User disabled status is updated in database
3. `before_user_update` event is triggered
4. Transaction commit
5. `after_user_update` and `user_sync` event is triggered

### Updating login ID
1. Transaction begin
2. Old login ID is deleted from database
3. New login ID is generated and sent to database
4. `before_identity_delete` event is triggered
5. `before_identity_create` event is triggered
6. Transaction commit
7. `after_identity_delete`, `after_identity_create`, `user_sync` event is triggered
