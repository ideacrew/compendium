---
title: Code Review Guidelines
description: >
  General guildlines for what you should expect to see in code reviews.
---

## Controllers and Controller Actions - Security

New controllers and controller actions should provide authorization and authentication.

When reviewing a new controller or action, expect to see Pundit being invoked on an endpoint:
1. Almost all new controller actions should be executing authorization using pundit - very few actions are allowed to be executed by all users.
2. You should expect to see the pundit authorization check usually in the controller action code itself, or, in some cases, a before filter/action.
3. You can easily spot usage of pundit by looking for calls to the `authorize` method.  As an example: `authorize @family, :show?`
4. If a new access check or set of access checks were introduced, expect to see a new pundit policy class with corresponding spec.