# Overview and terminology

UserWidgets manages Users, Organizations, Applications, access roles, and
permissions. It also provides widgets that applications can use to expose
administrative interfaces backed by the same API and authorization rules.

## Core terminology

### User

A **User** is a global identity in UserWidgets. A User can belong to more than
one Organization.

### Organization

An **Organization** is an independent, flat tenant boundary. Organizations are
not nested.

### Application

An **Application** is a global definition registered with UserWidgets. It owns
application-specific roles, permissions, and API keys.

### Application Instance

An **Application Instance** is an Application installed in a particular
Organization. It has organization-specific settings but no independent
identifier.

### Membership

A **Membership** relates one User to one Organization. It records whether the
User is `invited`, `active`, or `suspended` in that Organization. Membership is
a relationship, not an independently identified resource.

### Role

A **Role** belongs to one global Application and groups permissions for
operations in that Application. An Organization can override a Role for its
Application Instance.

### Role Override

A **Role Override** is an Organization-specific definition for one Role in one
Application Instance. It completely replaces the global Role's name,
description, and permission set for that Instance. If no override exists, the
global Role applies unchanged.

Role Overrides have no independent identifier. They are addressed by the
Organization, Application, and Role.

### Permission

A **Permission** is an immutable action key. UserWidgets owns platform
permissions, while an Application owns the domain permissions it registers
with UserWidgets.

### Role Assignment

A **Role Assignment** associates a User with a Role in a specific Organization
and Application Instance context.

### Effective Permission

An **Effective Permission** is a permission available to a User after all
assigned Roles, inherited Roles, and supported wildcard matches have been
evaluated for a specific context.
