# Implementation

## Server

### `auth:disable:set`

This action requires the use of a master key or an admin account.

#### Request

* `auth_id` (string) the user id of the user to disable
* `disabled` (boolean) whether the user is to be disabled
* `message` (string, optional) the message shown to user for disabling the user account
* `expiry` (datetime, optional) the time when the user is automatically enabled

#### Response

The API returns Status OK for successful request.

### UserAuthenticator preprocessor

UserAuthenticator will be modified to reject disabled user with
`UserDisabled` error.

```
{
    "name": "UserDisabled",
    "code": 127,
    "info": {
        "message": "Account disabled...",
        "expiry": "2017-07-23T19:30:24Z"
    }
}
```

A user is disabled if the following is true:

* `disabled` is true, and
* `disabled_expiry` is empty OR disabled_expiry is a date in the past

### Database

The following columns will be added to `_auth` table.

* `disabled` (boolean) true if the user is disabled
* `disabled_message` (string)
* `disabled_expiry` (datetime)

### Audit Events

The server should generate audit events when the user is enabled or disabled.

## SDK

### APIs

```
skygear.auth.disableUser("07A9EDA7-574A-4774-B08C-7EB5D310D420")
skygear.auth.enableUser("07A9EDA7-574A-4774-B08C-7EB5D310D420")
```
