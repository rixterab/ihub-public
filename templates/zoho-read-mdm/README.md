# Zoho Read MDM

Fetch users, groups, and devices from Zoho Mobile Device Management (MDM).

## What it does

Runs three read operations against the Zoho MDM API (ManageEngine):

1. **Get MDM Users** — fetches all users enrolled in Zoho MDM
2. **Get MDM Groups** — fetches all device groups
3. **Get MDM Devices** — fetches all managed devices

All three actions handle paginated responses automatically, following `paging.next` cursors until all records are retrieved.

## Authentication

Uses OAuth2 with Zoho's authorization server, requesting scopes for device management and user read access (`MDMOnDemand.MDMDeviceMgmt.ALL`, `MDMOnDemand.MDMUser.READ`).

## Trigger

Designed to be triggered via an Atlassian webhook (e.g. from Jira/Confluence automation).
