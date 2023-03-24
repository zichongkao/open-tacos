# Openbeta Organization Abstraction
Kao, Mar 9th

## Background
Current user data model, stored with third-party provider Auth0.
```
IUserProfile { // src/js/types/User.ts
  email?: string
  avatar?: string
  authProviderId: string

  IUserMetadata {
    IReadOnlyUserMetadata {
      uuid: string
      roles: string[]
      loginsCount: number
    }
    IWritableUserMetadata {
      name: string
      nick: string
      bio: string
      website?: string
      ticksImported?: boolean
      collections?: {
        climbCollections?: { [key: string]: string[] }
        areaCollections?: { [key: string]: string[] }
      }
    }
  }
}
```

## Proposal
### Organizations
We need a new entity for LCOs. To future-proof this construct, I propose we make it a subtype of a more general construct called an "Organization". It'll have the following data model which is similar in structure to `IUserProfile`:
```
IOrganization: {
  type: enum  // one of {"local_climbing_organization", …}
  avatar?: string
  // Organizations have no login, therefore no email/authProviderId as with IUserProfile.
  // It is their administrators that can log in and take actions.

  IOrganizationMetadata {
    IReadOnlyOrganizationMetadata {
      uuid: string
    }
    IWritableOrganizationMetadata {
      name: string
      nick: string
      bio: string
      website?: string
      email?: string   // Email moved here since it is no longer an identifier for the model.
      donationLink?: string
      hardwareReportLink?: string
      instagramLink?: string
      collections?: {
        areaCollections?: { [key: string]: string[] }
      }
    }
  }  
}
```

One cool idea here: We could also enable users to be associated with organizations. This could be self-declared or supplied by the organization. This would create stronger network effects and accelerate user growth. However, to prevent scope creep, we should take this idea for now.

### Organization Administrators
We need to be able to record that some users are organization administrators. We will allow multiple administrators per organization with no hierarchy amongst these admins. A user can also be admins of multiple organizations at once. For now, the site admin will manually assign users to be organization administrators.

These administrators will have the following abilities:
- Update the organization's `IWritableOrganizationMetadata` via the organization profile page. This includes updating donation/Instagram/email links, and associating the `Organization` with various climbing areas by adding those areas to `areaCollections`.
- Post notices on routes in their area (eg land-use warnings, raptor closures, etc)

### Users as Organization Administrators
Question here is how to record that a user is an organization administrator. 

1. Expand `roles`
The first option is to expand `user_metadata.roles` which currently supports only two values:
- `user_admin`, which is a site-wide admin capable of viewing all user metadata (src/pages/api/basecamp/users.ts), and migrating other users (src/pages/api/basecamp/migrate.ts)
- `editor`, which all users currently get by default, (see `AuthProvider/postLoginAction/setDefaultRoles-step0.js`) but doesn't seem to control anything.
The tricky thing is that an `organization_administrator` role requires us to specify which organizations they are administrators of, so just adding another possible value to `user_metadata.roles` is insufficient. Instead perhaps we could update `roles` from `[]string` to `[]object` so that we record things in this way:
```
  roles: [
    {role: user_admin},
    {role: organization_admin,
     organization: <bcc_uuid>},
    {role: organization_admin,
     organization: <action_committee_of_eldorado_uuid>},
  ] 
```

2. New `organization_admin` field
Another option is to create a new `user_metadata` field: `user_metadata.organization_admin: []string` where the strings are uuids of the organizations the user administers. This would create an additional layer of role-like controls, which may be confusing in future.

3. Hybrid
I think this is the worst of both worlds, but listing for completeness. Add 'organization_admin' as a possible value for `role`, and then create another `organization_admin` field to track which organizations they are admins of.

Overall, I'm leaning toward option 2. Option 1 introduces a polymorphic data structure in the objects in the array, which can be the source of many errors. Option 2's risk is creating bloat in the top layer of the data model, and interference between multiple role-like systems. But if it's just two systems, we should be fine. In future, more site-level roles could just slot into the `roles` field. Implementing area admins could be more tricky since they would require a user<>area map analogous to user<>organization map. But I guess then having an `area_admin` field just like `organization_admin` isn't that bad either.

------

## Update 2023-03-22
After discussions with Viet and Colin, we have a revised proposal for how to deal with roles and permissions. For some background, we use JWT token-based authentication. We have sessions, but these are on the client side so it's not considered the old-school server-side session-based authentication.

### Existing Authentication Mechanism
The whole system is organized using the `next-auth` library with `Auth0` as our (only) third-party [authentication provider](https://next-auth.js.org/providers/auth0). `Auth0` stores our user data (separate from the rest of the route/climbing data which is stored in our self-managed MongoDB instance) and on successful login, stuffs that data into two JWTs ("id", to identify the user, and "access", to determine what the user can access) which are saved in the user's browser. When the user is redirected back to our page, `next-auth` in our client application parses the JWTs and uses the data within to populate context variables in the user's session.

In more detail:

1. User clicks on the "Become a contributor" button and is redirected to an `Auth0` hosted page to log in. 
2. `Auth0` servers verify their login credentials and call the `postLoginActions` we've uploaded into the `Auth0` dashboard. (See `AuthProvider/postLoginAction/`, also [Auth0 docs](https://auth0.com/docs/customize/actions/flows-and-triggers/login-flow/event-object)). Our main action gives users the `editor` role if they don't already have it. We do this by inserting it as custom claims into the outgoing "access" and "id" JWT tokens.
3. `Auth0` returns a JWT to the user's browser and redirects the user back to our page.
4. On our client application, the `next-auth` framework calls the JWT callback (See src/pages/api/auth/[...nextauth].ts). Profile information from the JWT is reformatted and saved into a new JWT.
5. It then calls the Session callback (same file). Data from the JWT is loaded into `next-auth` session variables that our application can access freely.
6. At various points in our app, we refer to the session variables to check if the user should be allowed to perform various actions eg `if (session?.user.metadata?.roles?.includes('user_admin') ?? false) { ... }`  (src/pages/api/basecamp/users.ts). In the graphql server, we also parse the JWT in the headers of the API requests, check that it isn't tempered with and deduce the roels of the user. See (https://github.com/OpenBeta/openbeta-graphql/blob/develop/src/auth/middleware.ts)

### Revised Proposal
1. We utilize `Auth0`'s role-based access control framework (RBAC). In some ways, we've already jury-rigged an RBAC because we have `Auth0` assign roles to users (Step 2 above) and we utilize those roles to determine what users can do. Moving to `Auth0`'s RBAC takes this one step further by introducing granular "permissions". We will configure `Auth0` to grant the user a set of permissions based on the roles they have. For example the new `org:admin` role could grant `organization_profile:write` and `climbing_area:admin_write` permissions.

In the JWT, `Auth0` will list not only the roles, but also the permissions the user is granted. In our app and GraphQL server, we will check these permissions (instead of the role, as we currently do) before granting access. `organization_profile:write` would give the user the ability to update the data of the organization they are an admin for.

In addition, we'll have to update the codebase to not check for the existing `user_admin` and `editor` roles directly, but instead use the permissions they grant.

2. We need to map users not just to roles and permissions, but also different roles within different organizations. `Auth0` has [organization functionality](https://auth0.com/docs/manage-users/organizations/using-tokens) but it is inadequate for what we want because while it enables users to log in through different organizations, they can only be in one organization at a time. For us, a user may be an admin of multiple LCOs, and they should be able to access all their LCO's data through a single log-in. `Auth0`'s mechanism is to insert an `org_id` parameter into the JWTs.

I propose to borrow this construct. In Step 4 in the jwt callback, we check if the user has an `org:admin` role specified in the JWTs from `Auth0`. If so, we add an additional parameter `org:admin-org_ids` to the JWTs which is an array of org_ids for which the user is an admin of. In future, we can extend this by hydrating other `<role>-org_ids` parameters. **Open question: where would user <> org_ids mapping be stored? Auth0? Mongo? Can the JWT callback even access Mongo?**

For example, the access token would look something like this:
```
{
  "iss": "https://openbeta.auth0.com/",
  "sub": "auth0|602c0dcab993d10073daf680",
  "aud": [
    "https://openbeta-api/",
    "https://openbeta.auth0.com/userinfo"  
  ],
  "iat": 1616499255,
  "exp": 1616585655,
  "azp": "ENDmmAJsbwI1hOG1KPJddQ8LHjV6kLkV",
  "scope": "openid profile email",
  "https://tacos.openbeta.io/roles": [ // Custom Claim
    "editor",
    "org:admin",
  ]
  "org:admin-org_ids": [ // Custom Claim
    "org_9ybsU1dN2dKfDkBi",
    "org_t89djT8didj350gd",
  ].
  "permissions": [
    "organization_profile:write",
    "climbing_area:admin_write",
    .. other default permissions
  ]
}
```

3. Organization access control. In the app, we'll load these into the session variables as we currently do (Step 5). When the user attempts to edit the organization profile page, we check for two things in both the frontend and the GraphQL server: 1) Does the user have an org:admin role? 2) Does their `org:admin-org_ids` include the org they are trying to access?

4. Climbing area access control to enable admin-level editing, ie. updating of closures/landuse issues/etc. The question here is whether to introduce a dedicated permission for this eg. `climbing-area:admin-write`, which would be distinct from `climbing-area:write` that users would have by default and would allow them to update area descriptions, routes, etc. 

In theory we could rely on the org:admin role without a dedicated permission to determine if a user can admin-edit an area. The logic would be as follows:
```
  user 
  -> has `org:admin` role
  -> for each `org_id` in `org:admin-org_ids`
  -> for each org's `areaCollections`
  -> if area under consideration is a child area, allow admin-editing
```

But this goes against the RBAC patterns, so instead we could do:
```
  user 
  -> has `climbing-area:admin-write` permission (because of `org:admin` role)
  -> for each `org_id` in `org:admin-org_ids`
  -> for each org's areaCollections 
  -> if area under consideration is a child area, allow admin-editing
```

But this isn't very elegant because the `climbing-area:admin-write` permission becomes dependent on the user having `org:admin-org_ids`, but this relationship isn't very clear or explicit. It would not cleanly allow us to allow others who are not org admins to have the permission.

### Phasing
Phase 1: Define roles and permissions.
Phase 2: Create roles and permission on Auth0, update site to utilize permissions instead of roles.
Phase 3: Create `Organization` model and profile page.
Phase 4: Introduce org:admin roles and with suitable permissions to allow the user to update organizations.
