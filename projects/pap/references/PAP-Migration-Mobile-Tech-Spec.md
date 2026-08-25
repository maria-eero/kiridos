# PAP Migration \- Mobile Tech Spec

**Author(s):** *[Adauton Heringer](mailto:adauton@eero.com)*  
**Reviewer(s):** 

# Summary

This project integrates the MAP (Mobile Authentication Platform) SDK on [iOS](https://w.amazon.com/bin/view/IdentityServices/Mobile/iOS) and [Android](https://w.amazon.com/bin/view/IdentityServices/Mobile/MAP/MAP_Android) so users sign in against the eero **Private Account Pool** (PAP) through Amazon AuthPortal. The client exchanges the resulting MAP token with eero cloud at \`POST /2.3/login\` for an eero user token. Retail Amazon-login and the new PAP flow are consolidated into a single "Amazon sign-in" button on the Welcome screen (see the [SSO Figma](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-2404)).

## References

* [Figma](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-1863&p=f&t=yZTB8zmFXosZIbEK-0)  
* [PAP Migration HDL (cloud)](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit?usp=sharing)  
* [ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit?usp=sharing)  
* [Epic](https://eeroinc.atlassian.net/browse/CORE-32410)  
* [Slack channel](https://eero.slack.com/archives/C0AA52DR28G)

# Terminology

* **MAP**: Mobile Authentication Platform (Amazon SDK).  
* **PAP**: Private Account Pool (eero's Amazon Identity account pool).  
* **AuthPortal**: Amazon's authentication web UI, rendered in a WebView by MAP.  
* **Association handle**: OpenID `assoc_handle` string identifying the account pool.  
* **CID**: Customer ID assigned by Amazon Identity.  
* **DKV**: Dynamic Key-Value store used by eero cloud for runtime configuration.  
* **HASH\_SHARD**: Consistent-hashing percentage throttle type in eero's `ConsulThrottleService`.  
* **IDFV**: iOS `identifierForVendor`.  
* **Aztec**: Amazon Identity backend service that validates MAP tokens.

# Project Details

(Jira tickets will be added when this document is discussed and approved)

## 1\. Welcome Screen

The Welcome screen is restructured to present three top-level entries, replacing the current two-button layout and its bottom-sheet pickers.

[*Figma screenshot*](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-2404)   
*![][image1]*

| Button | Label | Routes to | Association handle |
| :---- | :---- | :---- | :---- |
| Top-left pill | Technicians | Existing SSO flow | Not applicable (external partner SSO) |
| Primary button | Amazon sign-in | Existing retail Amazon-login flow | `amzn_eero_mobile_android_us` |
| Secondary button | Email or phone number | New PAP flow | `amzn_eero_mobile_us` |

### 1.a \- Android

The current Welcome screen will remain unchanged. A new *WelcomeScreenV2* will be responsible for rendering the new version. The routing between old and new is gated per device by **feature\_203**. (see Feature Flags). The current "Get started" / "Sign in" bottom-sheet pickers are unreachable from v2 and are removed once the flag reaches 100% rollout.

Add `WelcomeFragmentV2` alongside the existing `WelcomeFragment`. Route between them at `V3OnboardingActivity.checkExtrasAndRoute()`.

#### Work Scope

- Add `WelcomeFragmentV2` with the new layout.  
- Update `V3OnboardingActivity.checkExtrasAndRoute()` to route `V3OnboardingRoute.Welcome` to either `WelcomeFragment` or `WelcomeFragmentV2` based on `feature_203`.  
- Delete `SetupBottomSheetDialogFragment`, `SignInBottomSheetDialogFragment`, and their view models once `feature_203` reaches 100% rollout.

#### Interactions

- "Technicians" pill → `SsoOtherPartnerFragment`.  
- "Amazon sign-in" → `SignInWithAmazonFragment` with with the platform's existing retail association handle (unchanged).  
- "Email or phone number" → `SignInWithEeroWebFragment` with () association handle `amzn_eero_mobile_us`.

#### Dependencies

- Cloud API route: `POST /2.3/login` (see Cloud Support).  
- MAP SDK (`android-map-lib` on Android; `MAPiOSLib` on iOS).

#### Flags to check

- `feature_203`: \- renders `WelcomeFragmentV2` when enabled; renders `WelcomeFragment` when disabled.

### 

### 1b. iOS

The current Welcome screen remains in place and renders both versions. It is already SwiftUI \+ TCA (Welcome.swift), and the new version differs only in the bottom call-to-action region, so that region is what swaps: signInOptionsrenders in place of ctaButtons, and the Technicians SSO entry moves from the picker to a top-bar item. Everything above it — the animation, wordmark, subtitle, and age-assurance blocking alert — is shared by both versions and unchanged. The routing between old and new is gated per device by papSignInOptions (see Feature Flags), read as shared state in Welcome.State so the screen and its toolbar react to it together. The current “Get started” / “Sign in” pair and the action sheet they present (WelcomeActionSheetViewController, the iOS counterpart of the bottom-sheet pickers) are unreachable from the new version and are removed once the flag reaches 100% rollout.

##### The screen is hosted by WelcomeCoordinator (Eero/EeroCore/Sources/Coordinators/WelcomeCoordinator.swift) via SwappableView(forKey: WelcomeViewKey.self, input: store), and driven by the Welcome reducer (Eero/EeroCore/Sources/Components/Welcome/Welcome.swift). The current UX presents WelcomeActionSheetViewController — the iOS counterpart of the bottom-sheet pickers — with five destinations across two sheet types: .new (setup with Amazon / with email or phone) and .signIn (sign in with Amazon / with email or phone / with SSO). It is unreachable from the new version and is removed once the flag reaches 100% rollout.

#### Work Scope

- Extend the existing Welcome reducer and WelcomeView: a signInOptions view builder in place of ctaButtons, a Technicians ToolbarItem on .topBarLeading, and an account-linking footnote below the buttons. Two branch points in the view; no conditional state and no conditional effects.  
- Add the three new intents (`.amazonSignIn, .emailOrPhoneSignIn, .technicianSignIn`) to WelcomeContinuationEffect and route them in WelcomeCoordinator (Eero/EeroCore/Sources/Coordinators/WelcomeCoordinator.swift), tracking LoginAnalyticsEvents.StartedLogin with the authentication type for each. The coordinator does not branch on the flag — it routes whichever intents arrive. Reading the flag at construction time would sample it once per coordinator and lose live toggling.  
- Update `WelcomeCoordinator` to branch on `feature_203` at construction time and route the three new intents.  
- Add a `newWelcomeScreen` `FeatureFlag?` property to `AppConfigurationFeatures` (`EeroFeatureFlags/Sources/AppConfiguration.swift`) with `debugMenuName` and `jiraTicket`. Add `case papWelcomeScreen = "feature_203"` to the private `FeatureFlagCodingKeys` enum.  The flag is read as shared state in Welcome.State, so the layout and toolbar react together and the debug menu can toggle it without a relaunch.  
- Add snapshot coverage for the flag-on states across the device variants (including the flag-on × Leo “Start setup” combination).  
- Delete `WelcomeActionSheetViewController` and its supporting views once the flag reaches 100%.

#### Interactions

- "Technicians" pill → `SSOCoordinator`.  
- "Amazon sign-in" → `AmazonLoginCoordinator` (retail handle from `AmazonLogin.configure(...)`; no changes to global MAP config).  
- "Email or phone number" → `AmazonLoginCoordinator. privateAccountPool(loginSessionID:)`

#### Dependencies

- [`iOS Map Lib`](https://github.com/eero-inc/ios-map-lib).  
- Cloud route: `POST /2.3/login` (see [Cloud Support](#cloud-support)).

#### Flags to check

- `feature_203 — papSignInOptions: renders the new sign-in options in the Welcome screen’s bottom call-to-action region, plus the Technicians toolbar item, when enabled; renders the existing “Get started” / “Sign in” pair when disabled. Read as shared state in Welcome.State, so it takes effect without a relaunch.`  
- `feature_204` `—` `forceAppUpgradeToUsePap`: presents the “app update required” screen (ErrorViewController.swift, with an App Store button) modally in place of the eero auth sign-in and sign-up flows when on; no effect when off.

## 2\. MAP integration

The MAP SDK is configured with three OpenID parameters that determine which account pool the AuthPortal WebView authenticates against:

- **`associationHandle`** — OpenID `assoc_handle` string identifying the account pool.  
- **AuthPortal domain** — the host where AuthPortal is served. MAP always prepends `https://` and appends `/ap/signin`, so the value must be a bare hostname on both platforms.

Retail Amazon-login continues to use each platform's existing global MAP configuration. The new PAP flow overrides these three parameters per-call, so retail is unaffected.

**PageId** \- We don't have to add pageId since it will pick the default one, which is the same as the association handle. We need to pass null, otherwise it will overide the default value.

### 

### Parameter values:

| Parameter | Retail (existing) | PAP (new) |
| :---- | :---- | :---- |
| associationHandle | `amzn_eero_mobile_android_us` | `amzn_eero_mobile_us` |
| AuthPortal domain | MAP default (`www.amazon.com`) | `ap.account.eero.com` |

### 2a. Android implementation

Android exposes the three parameters via a per-call `AuthParameters` struct in `android-map-lib`. MAP reads them from a `Bundle` at invocation time.

#### Work Scope

- Add `(assoc_handle to be confirmed)` to the `AssociationHandle` enum in `android-map-lib`. Wire the three per-handle fields:  
  - `associationHandle` string: `"to be confirmed"`.  
  - `pageId`: `to be confirmed`.  
  - `domain`: `to be confirmed`.  
- In `AmazonAccountManagerImpl.createMAPActivityParams(authParameters)`, populate the bundles:  
  - `authParameters.associationHandle` → `openid.assoc_handle` on the **inner** OpenID params bundle.  
  - `authParameters.pageId?.let { … }` → `pageId` on the **inner** bundle (skipped when null).  
  - `authParameters.domain?.let { … }` → `KEY_SIGN_IN_ENDPOINT` on the **outer** options bundle. Must be a bare hostname.  
- Do **not** set `KEY_REGISTRATION_DOMAIN`. MAP's `getPandaHost()` validator rejects values outside its allowlist, and Panda is routed per handle. Verified failure mode.

### 

### 2b. iOS implementation

iOS today configures MAP globally at app startup via `AIMAPiOSLibSettings` in `Eero/Sources/Application/AmazonLogin.swift`. Those globals stay untouched — retail continues to work as-is.

For the new PAP flow, invoke MAP with a per-call options dictionary rather than mutating the globals. This isolates PAP configuration to the new coordinator and prevents leakage into retail.

#### Work Scope

- Add `EeroWebLoginCoordinator` in `Eero/Sources/Coordinators/`, modeled after `AmazonLoginCoordinator`.  
- At the call site, pass an options dictionary with the three keys:

### 

```swift
let options: [String: Any] = [
    MAPKeyOpenIdAssociationHandle: "to be confirmed",
    MAPKeyAuthPortalDomain: "to be confirmed",
]
```

### 

  Must be a bare hostname on `MAPKeyAuthPortalDomain`.


- Consider a small helper (e.g. `MAPOptions.papDogfood`) that returns the dictionary above, analogous to Android's per-handle wiring — keeps the values in one place.

Retail invocations continue to use the globals in `AmazonLogin.swift` — no change to the retail path.

### Dependencies (both platforms)

- Amazon Identity `PAP` marketplace configuration.  
- AuthPortal reverse-proxy setup under `auth.eero.com` for production.  
- iOS MAP debug-variant scoping.

### 

## 3\. New PAP login endpoint

The new PAP flow POSTs the MAP token to `/2.3/login`. **The existing `/2.2/login/amazon` call is preserved** — retail Amazon-login continues to hit it(confirm), and users on the legacy welcome UX (where `feature_203 = false`) continue to reach it. During the rollout, both endpoints coexist. Removal of the legacy call is a follow-up once the cutover (step 5 of the rollout timeline) is complete and the legacy endpoint is fully deactivated.

The request body shape is the same on both endpoints (`auth_token`, `create_account`). `/2.3/login` ignores `create_account`; `/2.2/login/amazon` continues to consume it as before.

### 3a. Android implementation

#### Work Scope

- Add a new `IUserService.papLogin` Retrofit method in `core/src/main/java/com/eero/android/core/api/user/IUserService.kt`:

```kotlin
@FormUrlEncoded
@POST("2.3/login")
fun papLogin(
    @Field("auth_token") authToken: String,
): Single<DataResponse<Token>>
```

- Add the corresponding `UserService.papLogin` wrapper in `UserService.kt`.  
- Route the new PAP flow (`SignInWithEeroWebFragment` → `AuthenticateEeroWebAccountUseCase`, or the existing `AuthenticateAmazonAccountUseCase` parameterized for PAP) to call `papLogin` instead of `amazonLogin`.  
- Keep `IUserService.amazonLogin` and its `/2.2/login/amazon` URL unchanged. All existing callers (`SignInWithAmazonViewModel`, `CreateAmazonAccountViewModel`) continue to use this endpoint.

### 3b. iOS implementation

#### Work Scope

- Add a new `EeroAPIRoute.papLogin` in `Eero/Sources/Services/EeroAPI/EeroAPIRoute+Extensions.swift`:

```swift
static let papLogin = EeroAPIRoute(path: "/2.3/login")
```

- Add a call-site helper (analogous to the existing `EeroAPI.postDecodable(.loginAmazon, …)` pair in `EeroAPI+Extensions.swift`) that posts `["auth_token": mapToken]` to `.papLogin`.  
- Route the new PAP coordinator (`EeroWebLoginCoordinator` from §2b) to call the new helper.  
- Keep `EeroAPIRoute.loginAmazon` and its `/2.2/login/amazon` path unchanged. The existing sign-in and create-account call sites in `EeroAPI+Extensions.swift` continue to use this endpoint.

### Contract

Applies to the new `/2.3/login` endpoint.

| Element | Description |
| :---- | :---- |
| Body | `auth_token` (required; URL-encode the `|` character). `is_pro` (optional; consumer mobile omits or sends `false`). |
| Header | `X-Client-Device-Id: <IDFV on iOS, app-scoped UUID on Android>` from the corresponding client releases. |
| 200 | `{ "data": { "user_token": "…", "is_new_user": <bool> } }`. The client stores `user_token`. `is_new_user` drives the marketing-consent fallback prompt. |

## Cloud Support {#cloud-support}

The new PAP login route `POST /2.3/login` is delivered by the eero `user-service`. It replaces `/2.2/login`, `/2.2/register`, and `/2.2/pro/login`.

Feature-flag changes required:

- `feature_203` — rename to `papLoginRequired` (cloud, Android, iOS). Add `X-Client-Device-Id` bucketing by switching the base class to `AppVersionAndDeviceIdDependentFeatureFlag`. Set min-version to the PAP-complete release (Q9).  
- `feature_204` — rename to `forceAppUpgradeToUsePap` (cloud, Android, iOS). Move min-version from `26.12.0` to `26.10.0`.

See §Visibility for the file-level changes.

Cloud [PAP Migration HLD](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit?usp=sharing)*.*

## Visibility (Feature Flags)

Two feature flags coordinate the rollout. They're independent, activated at different phases, and read from the unauthenticated `GET /2.2/app_configuration` endpoint.

| Flag | Purpose | Min version | Client behavior when `true` | Client change required |
| :---- | :---- | :---- | :---- | :---- |
| `feature_203` | `papLoginRequired`: enables the new welcome screen \+ PAP flow. | `26.14.0`   | 26.12+: renders `WelcomeFragmentV2` / `WelcomeViewV2` and routes email/phone taps to the PAP flow. | Yes — new field, accessor, and welcome-routing check. |
| `feature_204` | `forceAppUpgradeToUsePap`: forces pre-PAP clients to update before entering credentials. | `26.10.0` (moved from `26.12.0`). | 26.10 / 26.11: shows `UpdateRequiredFragment` (Android) / `ErrorViewController` (iOS) before eero-auth credential entry. 26.12+ never reaches this check when `feature_203 = true`. | Yes — rename the shipped accessor. |

The two-flag design isolates the *feature* rollout from the *cutover* trigger. Any intermediate version that ships with a partial PAP implementation never enrolls in `feature_203` (its min-version can be raised as needed) and simply follows the existing eero-auth path until `feature_204` fires the update prompt. This is the reason we don't combine both concerns onto a single flag.

### `feature_203` — `papLoginRequired` (repurposed)

`feature_203` currently maps to `AuthXUpgradeMessagingFeatureToggle` (a deprecation-warning flag). Neither the Android nor the iOS client parses `feature_203` today, so the number is safely repurposed for the PAP feature gate.

**Cloud changes** (`eero-inc/cloud`):

- [ ] Rename `modules/user/app/eero/user/feature/AuthXUpgradeMessagingFeatureToggle.scala` → `PapLoginRequiredFeatureToggle.scala`.  
- [ ] Change the base class from `AppVersionDependentFeatureFlag.Builder` to `AppVersionAndDeviceIdDependentFeatureFlag.Builder` to enable `X-Client-Device-Id` bucketing.  
- [ ] Rename the throttle-set feature-name argument from `"AuthXUpgradeMessaging"` to `"PapLoginRequired"`. DKV path becomes `.../PapLoginRequired/enabledForDevice`.  
- [ ] Add `papLoginRequiredCapable = isMobileParityCapable(26, X, 0)` in `MobileAppVersion.scala`, where `X` is the PAP-complete release (Q9). Remove the old `authXUpgradeMessagingCapable`.  
- [ ] Update `ConfigurationView.scala`: swap the builder injection to `papLoginRequiredFeatureToggleBuilder`, and pass `clientDeviceId` in the `.apply(...)` call.  
- [ ] Keep the JSON field name `"feature_203"` in `ConfigurationView.scala`.  
- [ ] Update the toggle's `debugMenuName` to "PAP Login Required".

**Android changes** (`eero-inc/android`):

- [ ] Add `papLoginRequired: SimpleFeature? = null` to `core/src/main/java/com/eero/android/core/model/api/appconfiguration/features/Features.kt`, annotated `@SerializedName("feature_203")`.  
- [ ] Add `isPapLoginRequired` accessor in `AppConfigurationRepositoryExtensions.kt` and `FeatureAvailabilityManager.kt`.  
- [ ] Add `WelcomeFragmentV2` with the three-entry layout described in §Project Details 1a.  
- [ ] Update `V3OnboardingActivity.checkExtrasAndRoute()` to route `V3OnboardingRoute.Welcome` to either `WelcomeFragment` (existing) or `WelcomeFragmentV2` (new) based on `isPapLoginRequired`.  
- [ ] Wire the debug-menu override for the new flag.

**iOS changes** (`eero-inc/ios-client`):

- [ ] Add `papLoginRequired: FeatureFlag?` to `AppConfigurationFeatures` in `Eero/EeroFeatureFlags/Sources/AppConfiguration.swift`, with a `debugMenuName` and `jiraTicket`.  
- [ ] Add `case papLoginRequired = "feature_203"` to the private `FeatureFlagCodingKeys` enum in the same file.  
- [ ] Add `WelcomeV2` reducer \+ `WelcomeViewV2` in `Eero/Sources/Components/Welcome/` per §Project Details 1b.  
- [ ] Update `WelcomeCoordinator.init(ageRange:)` to read `@SharedReader(.featureFlag(\.papLoginRequired))` and construct either the existing or the v2 store.

### `feature_204` — `forceAppUpgradeToUsePap` (renamed)

`feature_204` maps to `AuthXLoginFeatureToggle` and was shipped in 26.10.0 on both platforms. The client already handles `feature_204 = true` by showing `UpdateRequiredFragment` (Android: `SignInBottomSheetViewModel` / `SetupBottomSheetViewModel`) or `ErrorViewController` (iOS: `WelcomeCoordinator.makeUpdateRequiredViewController`) before any credential entry on the email/phone path. The functional behavior is correct; only the naming needs to change so the flag matches its actual role — forcing pre-PAP clients (26.10 / 26.11) to update before credential entry.

Min-version moves from `26.12.0` to `26.10.0` so that 26.10 and 26.11 clients receive the flag once we enable it at cutover.

**Cloud changes** (`eero-inc/cloud`):

- [ ] Rename `modules/user/app/eero/user/feature/AuthXLoginFeatureToggle.scala` → `ForceAppUpgradeToUsePapFeatureToggle.scala`.  
- [ ] Rename the throttle-set feature-name argument from `"AuthXLogin"` to `"ForceAppUpgradeToUsePap"`. DKV paths shift to `.../ForceAppUpgradeToUsePap/...` accordingly.  
- [ ] Rename `authXLoginCapable` in `modules/playdata/src/main/scala/eero/playdata/mobile/MobileAppVersion.scala` to `forceAppUpgradeToUsePapCapable`, and change the version to `isMobileParityCapable(26, 10, 0)`.  
- [ ] Rename `authXLoginFeatureToggleBuilder` and its `apply(...)` variable in `modules/user/app/eero/user/views/ConfigurationView.scala` to `forceAppUpgradeToUsePapFeatureToggleBuilder`.  
- [ ] Keep the JSON field name `"feature_204"` in `ConfigurationView.scala`.  
- [ ] Update the toggle's `debugMenuName` to "Force App Upgrade To Use PAP".

**Android changes** (`eero-inc/android`):

- [ ] Rename `papAuthxLoginRequired` → `forceAppUpgradeToUsePap` in `Features.kt` (keep `@SerializedName("feature_204")`).  
- [ ] Rename `isPapAuthxLoginRequiredEnabled` → `isForceAppUpgradeToUsePap` in `AppConfigurationRepositoryExtensions.kt`.  
- [ ] Rename `isPapAuthxLoginRequired` → `isForceAppUpgradeToUsePap` in `FeatureAvailabilityManager.kt`.  
- [ ] Update the two call sites — `SignInBottomSheetViewModel.onSignInWithEeroClicked` and `SetupBottomSheetViewModel.continueWithEmailOrPhone` — to reference the new accessor. Behavior unchanged.  
- [ ] Update related unit tests (`SignInBottomSheetViewModelTest`, `SetupBottomSheetViewModelTest`, `AmazonAccountErrorViewModelTest`) to reference the new accessor.

**iOS changes** (`eero-inc/ios-client`):

- [ ] Rename `papAuthXLogin` → `forceAppUpgradeToUsePap` in `AppConfigurationFeatures` (`Eero/EeroFeatureFlags/Sources/AppConfiguration.swift`).  
- [ ] Rename `case papAuthXLogin = "feature_204"` → `case forceAppUpgradeToUsePap = "feature_204"` in `FeatureFlagCodingKeys`.  
- [ ] Update `WelcomeCoordinator.makeUpdateRequiredViewController` (`Eero/EeroCore/Sources/Coordinators/WelcomeCoordinator.swift`) to read `@SharedReader(.featureFlag(\.forceAppUpgradeToUsePap))`. Behavior unchanged.  
- [ ] Update related tests (`WelcomeCoordinatorUpdateRequiredTests`, snapshot tests) to reference the new key path.

Note: 26.10.0 and 26.11.0 shipped with the old accessor names and receive the JSON key `"feature_204"` — the wire format is unchanged, so their behavior is preserved. The rename is a code-level change in 26.12+ only.

### Rollout timeline

1. **Ship 26.12** with the full `feature_203` client wiring, `WelcomeFragmentV2` / `WelcomeViewV2`, the PAP flow, and the endpoint migration to `/2.3/login`. Both flags default off server-side.  
2. **Phased rollout of PAP.** Ramp `feature_203` device-ID throttle: `5% → 25% → 50% → 100%`. Only PAP-complete clients enroll (via `papLoginRequiredCapable`). Watch metrics on PAP token exchange, `/2.3/login` success rate, and sign-in completion.  
   - Unenrolled 26.12+ users continue on the existing welcome \+ eero-auth path.  
   - 26.10 / 26.11 users continue on the existing welcome \+ eero-auth path (below min-version, so `feature_203` is invisible).  
   - Legacy endpoints continue to serve traffic.  
3. **Activate** `forceAppUpgradeToUsePap` **for pre-PAP clients.** When `feature_203` is at `100%` and metrics are healthy:  
   - Enable `feature_204` — either `enabledForPublic = true` or ramp its device throttle to `100%`. Because min-version was moved to `26.10.0` in the cloud code, 26.10 / 26.11 clients qualify without any DKV override.  
   - Result: 26.10 / 26.11 users start seeing `UpdateRequiredFragment` / `ErrorViewController` before credential entry. 26.12+ users are on the new welcome \+ PAP and never reach the check.  
4. **Wait for install-base decay** on 26.10 / 26.11 as users update.  
5. **Cut over legacy endpoints.** Flip `eeroAuthLoginDisabled` and `eeroAuthRegistrationDisabled` DKVs. `/2.2/login*` and `/2.2/register*` return 404\. Any remaining 26.10 / 26.11 user who ignored the update prompt hits the raw 404 as the last-resort backstop.

### Rollback strategy

Each flag is independent, which gives clean per-concern rollback:

- **PAP has issues during ramp:** drop the `feature_203` device-ID throttle to `0%`. 26.12+ users fall back to the existing welcome \+ eero-auth. `feature_204` is unaffected. No user is stranded because legacy endpoints are still live.  
- `forceAppUpgradeToUsePap` **fires prematurely:** disable the flag (`enabledForPublic = false` or set the device throttle to `0%`). 26.10 / 26.11 users stop seeing the update prompt on their next `app_configuration` refresh.

## Observability (Analytics) (TBD)

## Security (TBD)

[image1]: (Welcome Screen V2 mockup — see Figma link above)
