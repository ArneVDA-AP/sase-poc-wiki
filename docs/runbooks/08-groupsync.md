---
title: "Runbook: GroupSync"
tags: [runbook, identity, entra-id, zitadel, netbird, groupsync]
---

# Runbook: GroupSync

**Node(s):** Entra ID (aplab.be) + Zitadel (mgmt01) + NetBird Dashboard
**Prerequisites:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.md) completed (overlay operational), Entra ID admin access on aplab.be
**Status:** Operational

---

## Prerequisites checklist

- [ ] NetBird overlay is operational ([Runbook 02](02-ztna-overlay.md))
- [ ] Entra ID admin access on aplab.be tenant
- [ ] Zitadel admin access on mgmt01
- [ ] App registration `11803ee8-eb15-462c-a286-5415c17a29c6` exists in Entra ID

---

## Step 1: Configure Token Configuration in Entra ID

In the Azure portal, navigate to the app registration (`11803ee8-eb15-462c-a286-5415c17a29c6`):

1. Go to **Token configuration** in the left menu
2. Open the **Manifest** editor and attach `cloud_displayname` to the groups claim (configured in Step 2): under `optionalClaims` add `"additionalProperties": ["cloud_displayname"]` to the `groups` entry, and set `"groupMembershipClaims": "ApplicationGroup"`
3. Save the manifest

> **Gotcha: `cloud_displayname` is not selectable in the Token Configuration UI** — it must be added by editing the manifest directly. It makes the `groups` claim emit each group's *display name* (e.g. `2ITCSC1A-Studenten`) instead of its ObjectID GUID — the hard precondition for Path B, since Zitadel's allowlist matches on the display-name string and cannot match a GUID. Requires Azure AD Premium (aplab.be has A5/P2) and cloud-only security groups.

---

## Step 2: Configure groups claim

Still in Token configuration:

1. Click **Add groups claim**
2. Select: **Groups assigned to the application** (not "All groups")
3. For each token type (ID, Access, SAML), ensure group claims are enabled
4. Save

> **Gotcha: Do NOT select "All groups".** Selecting all groups includes every group in the tenant, which bloats the token and may hit JWT size limits. "Groups assigned to the application" returns only the 3 security groups explicitly assigned to this Enterprise Application.

---

## Step 3: Assign security groups to the Enterprise Application

Navigate to **Enterprise Applications** → find the corresponding Enterprise Application:

1. Go to **Users and groups** → **Add user/group**
2. Assign these 3 security groups:

| Entra ID security group | NetBird group (Path B) | Purpose |
|-------------------------|------------------------|---------|
| `2ITCSC1A-Studenten` | `Studenten` | Student persona — proxy access via Core-Services |
| `2ITCSC1A-Docenten` | `Docenten` | Teacher persona — proxy access via Core-Services |
| `2ITCSC1A-Admins` | `Admins` | Admin persona — full administrative access |

3. Confirm all 3 groups appear in the assignment list

---

## Step 4: Enable user assignment required

Still on the Enterprise Application:

1. Go to **Properties**
2. Set **User assignment required** to **Yes**
3. Save

This prevents unassigned users from authenticating through the application. Only users in one of the 3 security groups can log in.

---

## Step 5: Create Zitadel Actions

In the Zitadel admin console on mgmt01, navigate to **Actions**:

### Action 1 — External Authentication

**Trigger:** External Authentication
**Allowed to fail:** true

This action:

- Reads the group display names from the Entra ID token's `groups` claim (present as names, not GUIDs, because `cloud_displayname` is enabled — Step 1)
- Maps each display name to its clean persona name via a hardcoded allowlist (`2ITCSC1A-Studenten` → `Studenten`, `2ITCSC1A-Docenten` → `Docenten`, `2ITCSC1A-Admins` → `Admins`); a group not in the allowlist is dropped (fail-closed)
- Writes the matched clean names to the user metadata key `sase_groups` (comma-joined)

### Action 2 — Complement Token

**Trigger:** Complement Token
**Allowed to fail:** true

This action:

- Reads `sase_groups` from the user's metadata and calls `setClaim('groups', [...])` to inject those clean names into the Zitadel-issued JWT
- NetBird reads this `groups` claim to assign users to persona groups

> **Gotcha: Both actions must have `allowed-to-fail: true`.** If an action fails (e.g., Entra ID doesn't return the expected claim for a user), authentication should still succeed — the user just won't have persona groups assigned. See [Decision: GroupSync PAD B](../decisions/groupsync-pad-b.md).

---

## Step 6: Verify user login and group assignment

1. Log in as a test user who is a member of one of the 3 security groups
2. Open the NetBird Dashboard → **Peers**
3. Locate the test user's peer

**Expected:** The peer appears with the correct persona group (Studenten, Docenten, or Admins) based on their Entra ID security group membership.

---

## Step 7: Verify persona groups in NetBird

1. Open NetBird Dashboard → **Groups**
2. Verify that persona groups exist: Studenten, Docenten, Admins
3. Verify that each group contains the correct members based on Entra ID assignments

---

## Final verification

Perform end-to-end validation:

```
netbird up  (as test user)
```

1. NetBird Dashboard → Peers → test user shows correct persona group
2. Identity Bridge `/lookup` endpoint returns the correct persona for the user's IP:

```bash
curl http://192.168.122.23:<port>/lookup?ip=<user-overlay-ip>
# Expected: returns persona group name (Studenten/Docenten/Admins)
```

---

## Checklist

- [ ] `cloud_displayname` optional claim added to token configuration
- [ ] Groups claim set to "Groups assigned to the application"
- [ ] 3 security groups assigned to Enterprise Application
- [ ] User assignment required = Yes
- [ ] Zitadel Action 1 (External Authentication) created with `allowed-to-fail: true`
- [ ] Zitadel Action 2 (Complement Token) created with `allowed-to-fail: true`
- [ ] Test user login shows correct persona group on NetBird Dashboard
- [ ] Persona groups (Studenten/Docenten/Admins) exist in NetBird with correct members

---

## Related

- [Component: NetBird](../components/netbird.md)
- [Component: Identity Bridge](../components/identity-bridge.md)
- [Decision: GroupSync PAD B](../decisions/groupsync-pad-b.md)
- [Decision: Zitadel as IdP Broker](../decisions/zitadel-idp-broker.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Runbook 09: Identity Bridge](09-identity-bridge.md)
