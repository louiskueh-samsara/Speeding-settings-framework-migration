# Speeding-settings-framework-migration

<!-- b0228c3e-9bba-4e47-901c-31b84ccbb585 8449186c-c95e-47bd-8d36-f70bf26ead88 -->
# Speeding Settings Framework Migration

## Overview

Migrate speeding settings to the Settings Framework following the pattern established for SafetyEventDetection and other safety settings, maintaining complete feature parity with the legacy system.

## Current State Analysis

**Legacy Storage:**

- Org-level: `safetydb.org_safety_speeding_settings` table with protobuf serialization
- Device-level: `safetydb.device_safety_settings` table with protobuf serialization
- Proto definition: `orgproto.OrganizationSettings_OrganizationSafetySettings_SpeedingSettings`

**Legacy Access Layer:**

- `speedingsettingsaccessor.Accessor` - handles reads with org/device hierarchy merging
- `safetymodels.OrgSafetySpeedingSettings` - org-level CRUD
- `safetymodels.DeviceSafetySettings` - device-level CRUD

**GraphQL Layer:**

- Current: Direct queries to legacy storage via `gqlsafetysetting/speeding.go`
- Frontend: `client/pages/org_config/safety/speeding_settings/`

## Settings Structure

Based on the proto definition, the namespace must include:

**Max Speed Settings** (absolute speed limit):

- enabled (boolean)
- kmphThreshold (float)
- timeBeforeAlertMs (integer)
- inCabAudioAlertsEnabled (boolean)
- sendToSafetyInbox (boolean, nullable)
- autoAddToCoaching (boolean, nullable)

**Severity Settings** (speed over limit) - array of 4 levels:

- severityLevel (enum: Light, Moderate, Heavy, Severe)
- speedOverLimitThreshold (float) 
- timeBeforeAlertMs (integer)
- enabled (boolean)
- sendToSafetyInbox (boolean, nullable)
- autoAddToCoaching (boolean, nullable)
- evidenceBasedSpeedingEnabled (boolean, nullable)

**Unit Configuration:**

- severitySettingsSpeedOverLimitUnit (enum: MilesPerHour, KilometersPerHour, Percentage, MilliknotsPerHour)

**In-Cab Severity Alerts:**

- enabled (boolean)
- alertAtSeverityLevel (enum: Light, Moderate, Heavy, Severe)
- timeBeforeAlertMs (integer)

**CSL (Commercial Speed Limit):**

- cslEnabled (boolean)

## Implementation Plan

### Phase 1: Schema Definition

Create `/go/src/samsaradev.io/platform/settings/settingsregistry/speeding_settings_schema.go`:

- Define `SpeedingSettingsNamespace` using `NewSettingsNamespace`
- Package: `teamnames.Safety` (or appropriate safety team)
- Include all settings matching the proto structure
- Enable options:
- `TriggerConfigPushOnUpdate()` - for device config pushes
- `UseGraphQL(&GqlOptions{Enabled: true})` - for GraphQL access
- `SupportedEntityTypes(EntityTypeDevice)` - org + device levels
- Use appropriate display names and default values from `speedingconstants`
- Generate code: `taskrunner gen/settings/schema`

### Phase 2: Backend Read Path Migration

Update services to use Settings Framework client instead of legacy accessors:

- Update `speedingsettingsaccessor.Accessor.GetActiveOrgSpeedingSettings` to call Settings Framework
- Update `speedingsettingsaccessor.Accessor.GetSpeedingSettingsAtTimeMs` for historical queries
- Maintain device/org hierarchy merging logic (now handled by Settings Framework)
- Keep feature flag checks for inbox defaults and other behaviors
- Update unit conversion logic (milliknots handling)

Services to update:

- Safety event processing
- Safety score calculations  
- Config push generation
- Any other consumers of speeding settings

### Phase 3: Backend Write Path Migration

Update mutation paths to use Settings Framework:

- Replace `safetymodels.OrgSafetySpeedingSettings.UpsertOrgSafetySpeedingSettings` with Settings Framework `UpdateSettings`
- Replace device-level updates in `DeviceSafetySettings` with Settings Framework
- Ensure audit metadata is properly populated
- Maintain backward compatibility during transition

### Phase 4: GraphQL Layer Migration

Use auto-generated GraphQL from Settings Framework:

- Settings Framework will generate: `settings/generated/safety/graphql/speeding_settings_server_generated.go`
- Implement `CheckWritePermissionBatch` in `speeding_settings_server.go` for proper authorization
- Update GraphQL queries in `client/pages/org_config/safety/speeding_settings/org_fragments.graphql`
- Integrate with existing GQL server structure

### Phase 5: Frontend Migration

Update React components to use new GraphQL schema:

- Update `speeding_settings.tsx` to query new Settings Framework GraphQL types
- Update `individual_speeding_settings.tsx` for device-level settings
- Use Settings Framework widgets where applicable (from `@settingswidgets`)
- Maintain all existing UI behaviors (inheritance, validation, etc.)
- Test org-level and device-level setting interactions

### Phase 6: Data Migration

Handle existing data in legacy tables:

- Option A: One-time migration script to copy all existing settings to Settings Framework
- Option B: Lazy migration - keep legacy storage, use default values from schema for orgs without migrated data
- Option C: Feature-flagged hybrid - use legacy for some orgs, new framework for others during rollout

Recommend **Option B** (lazy migration) for safety:

- New updates go to Settings Framework only
- Historical reads can still access legacy storage if needed
- Gradual natural migration as orgs update settings

### Phase 7: Testing & Validation

- Unit tests for Settings Framework client usage
- Integration tests for org/device hierarchy
- GraphQL mutation/query tests
- Frontend E2E tests for settings CRUD
- Verify config push triggers work correctly
- Validate historical settings queries
- Performance testing (ensure no regression)

### Phase 8: Cleanup (Post-Migration)

After full rollout and verification:

- Deprecate legacy accessor methods
- Archive legacy tables (don't delete - keep for audit/rollback)
- Remove old GraphQL resolvers
- Update CODEREVIEW files for ownership

## Key Files to Create/Modify

**New files:**

- `go/src/samsaradev.io/platform/settings/settingsregistry/speeding_settings_schema.go`
- Generated: `settings/generated/safety/graphql/speeding_settings_server.go`
- Generated: `settings/generated/safety/speeding_settings.go`

**Files to modify:**

- `go/src/samsaradev.io/safety/safetyaccessors/speedingsettingsaccessor/accessor.go`
- `go/src/samsaradev.io/safety/gql/gqlsafetysetting/speeding.go`
- `client/pages/org_config/safety/speeding_settings/speeding_settings.tsx`
- `client/pages/org_config/safety/speeding_settings/org_fragments.graphql`
- `client/pages/org_config/safety/speeding_settings/individual_speeding_settings.tsx`

**Files to reference:**

- `go/src/samsaradev.io/settings/orgproto/organization_settings.proto` (current structure)
- `go/src/samsaradev.io/safety/speedingconstants/speedingconstants.go` (default values)
- `go/src/samsaradev.io/platform/settings/settingsregistry/safety_event_detection_schema.go` (pattern reference)

## Risks & Mitigations

**Risk**: Data inconsistency during migration
**Mitigation**: Use feature flags, canary rollout, maintain legacy read paths initially

**Risk**: Performance regression from framework overhead
**Mitigation**: Benchmark before/after, leverage Settings Framework batch APIs

**Risk**: Breaking changes to GraphQL schema
**Mitigation**: Version GraphQL carefully, coordinate with frontend team

**Risk**: Config push behavior changes
**Mitigation**: Thoroughly test TriggerConfigPushOnUpdate with device configs

## Success Criteria

- All speeding settings CRUD operations use Settings Framework
- Org and device-level hierarchy works identically to legacy system
- GraphQL API maintains backward compatibility
- Frontend functionality unchanged from user perspective
- Config pushes trigger correctly on setting updates
- Historical settings queries work for compliance/audit
- No performance degradation
- All existing tests pass + new tests for Settings Framework integration
