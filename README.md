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

Migrate read paths to use Settings Framework client with delegated resolver pattern. This enables gradual migration with zero downtime - the delegated resolver provides seamless fallback to legacy storage while allowing Settings Framework to serve migrated data. Feature flag controls which path is used without requiring code changes.

#### 2.1 Define Feature Config

Create feature config to control resolution strategy (framework vs legacy):

- Add `speeding_settings_use_framework` feature flag in appropriate feature config file
- Feature flag will gate whether Settings Framework or legacy database is used
- Settings client will evaluate this flag automatically - no need to check it in application code

#### 2.2 Implement Delegated Resolver

Create delegated resolver to interface with legacy storage:

- Define `DelegatedResolver[SpeedingSettings]` with `GetOrgSettings` and `GetDeviceSettings` functions
- Lift-n-shift existing read logic from `speedingsettingsaccessor.Accessor` into resolver functions
- Map legacy proto structures to new Settings Framework schema types
- Handle device/org hierarchy merging logic within resolver
- Maintain unit conversion logic (milliknots handling)
- Preserve feature flag checks for inbox defaults and other behaviors

Example structure:

```go
speedingSettingsResolver := settingsclient.DelegatedResolver[speeding.SpeedingSettings]{
    GetOrgSettings: func() (*speeding.SpeedingSettings, error) {
        // Lift from speedingsettingsaccessor.GetActiveOrgSpeedingSettings
        // Map orgproto.SpeedingSettings -> speeding.SpeedingSettings
        return mappedSettings, nil
    },
    GetDeviceSettings: func() (*speeding.SpeedingSettings, error) {
        // Lift from device-level accessor logic
        // Map device proto -> speeding.SpeedingSettings
        return mappedSettings, nil
    },
}
```

#### 2.3 Update Read Path Call Sites

Replace direct accessor calls with Settings Framework client:

**Before:**
```go
settings, err := speedingsettingsaccessor.GetActiveOrgSpeedingSettings(ctx, orgId, deviceId)
```

**After:**
```go
settings, err := settingsClient.GetSettings(ctx, &settingsclient.GetSettingsRequest[speeding.SpeedingSettings]{
    Id: &settingsclient.Id{
        OrgId: uint64(orgId),
        EntityId: &settingsclient.EntityId{
            Type: settingsclient.EntityTypeDevice,
            Id:   uint64(deviceId),
        },
    },
}, speedingSettingsResolver)
```

Services to update:

- Safety event processing
- Safety score calculations  
- Config push generation
- Any other consumers of speeding settings

#### 2.4 Handle Historical Queries

For time-based queries (e.g., `GetSpeedingSettingsAtTimeMs`):

- Settings Framework has built-in temporal support via `settingsclient.Timestamp`
- Update historical query logic to use Settings Framework temporal APIs
- Ensure delegated resolver handles historical fallback to legacy storage

#### 2.5 Testing Read Path

- Port existing data-mocking tests to delegated resolver
- Refactor unit tests to use settings client mocking instead of legacy accessor mocking
- Unit tests should NOT distinguish between legacy/framework resolution
- Add tests to verify feature flag controls resolution path
- Verify device/org hierarchy merging produces identical results

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

Handle existing data in legacy tables. The delegated resolver enables flexible migration strategies:

- **Option A**: One-time migration script to copy all existing settings to Settings Framework
- **Option B**: Lazy migration - keep legacy storage, delegated resolver serves unmigrated data seamlessly
- **Option C**: Feature-flagged hybrid - use legacy for some orgs, new framework for others during rollout

Recommend **Option B** (lazy migration) for safety:

- Delegated resolver automatically serves legacy data when Settings Framework has no data
- New updates go to Settings Framework only
- Historical reads leverage `IsMigrated` flag in setting metadata
- Gradual natural migration as orgs update settings
- No coordination needed between read and write path migrations

### Phase 7: Testing & Validation

**Unit Tests:**

- Port existing data-mocking tests for delegated resolver
- Refactor tests to mock settings client instead of legacy accessors
- Tests should NOT distinguish between legacy/framework resolution
- Verify feature flag properly controls resolution strategy
- Test org/device hierarchy merging produces identical results

**Integration Tests:**

- Verify Settings Framework client with delegated resolver end-to-end
- Test fallback behavior when settings not yet migrated
- Validate historical settings queries with temporal support
- Test that feature flag toggle switches between framework/legacy smoothly

**GraphQL & API Tests:**

- GraphQL mutation/query tests
- Verify config push triggers work correctly
- Test API compatibility with existing consumers

**Frontend Tests:**

- E2E tests for settings CRUD operations
- Verify UI behavior unchanged from user perspective

**Performance & Monitoring:**

- Benchmark settings read latency (framework vs legacy)
- Monitor settings access patterns after deployment
- Ensure no performance regression from delegated resolver overhead

### Phase 8: Cleanup (Post-Migration)

After full rollout and verification:

- Deprecate legacy accessor methods
- Archive legacy tables (don't delete - keep for audit/rollback)
- Remove old GraphQL resolvers
- Update CODEREVIEW files for ownership

## Key Files to Create/Modify

**New files:**

- `go/src/samsaradev.io/platform/settings/settingsregistry/speeding_settings_schema.go` - schema definition
- `go/src/samsaradev.io/safety/safetyaccessors/speedingsettingsaccessor/delegated_resolver.go` - delegated resolver implementation
- Generated: `settings/generated/safety/graphql/speeding_settings_server.go`
- Generated: `settings/generated/safety/speeding_settings.go`

**Files to modify:**

- Feature config file (add `speeding_settings_use_framework` flag)
- `go/src/samsaradev.io/safety/safetyaccessors/speedingsettingsaccessor/accessor.go` - update to use settings client
- `go/src/samsaradev.io/safety/gql/gqlsafetysetting/speeding.go`
- All services consuming speeding settings (event processing, score calculations, config push, etc.)
- `client/pages/org_config/safety/speeding_settings/speeding_settings.tsx`
- `client/pages/org_config/safety/speeding_settings/org_fragments.graphql`
- `client/pages/org_config/safety/speeding_settings/individual_speeding_settings.tsx`

**Files to reference:**

- `go/src/samsaradev.io/settings/orgproto/organization_settings.proto` (current structure)
- `go/src/samsaradev.io/safety/speedingconstants/speedingconstants.go` (default values)
- `go/src/samsaradev.io/platform/settings/settingsregistry/safety_event_detection_schema.go` (pattern reference)

## Risks & Mitigations

**Risk**: Data inconsistency during migration  
**Mitigation**: Use feature flags, canary rollout, delegated resolver ensures legacy data always accessible during transition

**Risk**: Performance regression from framework overhead  
**Mitigation**: Benchmark before/after, leverage Settings Framework batch APIs, monitor read latency metrics

**Risk**: Breaking changes to GraphQL schema  
**Mitigation**: Version GraphQL carefully, coordinate with frontend team, maintain backward compatibility

**Risk**: Config push behavior changes  
**Mitigation**: Thoroughly test TriggerConfigPushOnUpdate with device configs

**Risk**: Read path migration breaks existing functionality  
**Mitigation**: Delegated resolver provides exact same logic as legacy accessor, comprehensive test coverage

## Rollback Strategy

The delegated resolver pattern with feature flags provides safe rollback capability:

**Immediate Rollback:**
- Toggle `speeding_settings_use_framework` feature flag to `false`
- Settings client automatically falls back to legacy resolution via delegated resolver
- No code deployment required for rollback

**Canary Rollout:**
- Enable feature flag for subset of orgs initially
- Monitor metrics: read latency, error rates, config push success
- Gradually increase rollout percentage based on metrics
- Instant rollback for problematic orgs by toggling flag

**Post-Migration Validation:**
- After all orgs migrate and stabilize, prepare to remove delegated resolver
- Ensure Settings Framework data fully populated before removing fallback
- Archive legacy storage only after extended validation period

## Success Criteria

- All speeding settings CRUD operations use Settings Framework
- Org and device-level hierarchy works identically to legacy system
- GraphQL API maintains backward compatibility
- Frontend functionality unchanged from user perspective
- Config pushes trigger correctly on setting updates
- Historical settings queries work for compliance/audit
- No performance degradation
- All existing tests pass + new tests for Settings Framework integration
