# R0-8: Table inventory for workspace tenancy

| Field | Value |
|---|---|
| Roadmap task | [R0-8](WORKSPACE-TENANCY-ROADMAP.md#r0-preparation-and-baseline) |
| Decisions | [ADR-001](ADR-001-workspace-tenancy-and-bank-ingestion.md) section 24 is authoritative. This document classifies tables; it decides nothing |
| Code inspected | fern-funds `master` at `c6447fc5` (2026-11-16) |
| Status | Analysis only, awaiting Product Owner review. No schema, entity or behaviour change |

## 1. Method

1. **Schema from the migrations, not from the entities.** All 178 Flyway scripts (`V1` to `V179`) were applied in version order to a throwaway PostgreSQL 16 container. The inventory was then read from `information_schema` and `pg_indexes`.
   - Tables, columns and nullability.
   - Foreign keys and their delete rules.
   - Unique indexes.
   - Triggers.
   - Columns named `*_id` that have **no** foreign key.

   The result is **98 tables**, plus Flyway's own `flyway_schema_history` in a real deployment. No production database was accessed.
2. **Code mapping.** Every `@Entity` and its `@Table` name was matched to its table, along with every `@JoinTable` and `@CollectionTable`. Every Java file containing SQL that names the table was found: JPQL and native `@Query`, `JdbcTemplate`, erasers.
3. **Classification** follows ADR-001 TEN, D-6, D-8, D-13, D-38, U27, U28 and section 28.8 (Akahu credentials are user-owned).
   - **WORKSPACE:** financial or household data that will belong to exactly one workspace.
   - **USER:** belongs to one person, whatever workspace they are in.
   - **REFERENCE:** global, system or operational data.

   Two dispositions are added where a table should not be migrated at all: **LEGACY** (removed in R6) and **RETIRED** (dead feature, drop recommended). Each such table still carries one of the three primary classes.

## 2. Reconciliation with the roadmap counts

| Roadmap statement (section 5) | Inventory finding |
|---|---|
| "`owner_id` on 65 entities" | Correct for entities. **69 tables** have an `owner_id` column. The 4 without an entity are `journal_entry`, `ledger_node` and `ledger_node_mapping` (retired ledger, erased by `user/RetiredLedgerEraser` via `JdbcTemplate`), and `stripe_checkout_session` (`connectors/stripe/StripeCheckoutSessionLog`, `JdbcTemplate`) |
| (not stated) | **29 tables have no `owner_id`**. Of these: 6 are `users` itself or user-keyed through `user_id`; 11 inherit ownership through a parent FK, a collection or a storage key; 12 are reference or system tables |
| "33 classes use `JdbcTemplate` or native SQL" | Native `@Query(nativeQuery = true)` appears in 3 repositories: `TransactionRepository`, `UserRepository` and `HouseholdRepository`. `JdbcTemplate` appears in 30 classes, mostly erasers plus `OutboxRepository`, `StripeCheckoutSessionLog`, `AccountDeletionFence`, `AccountDeletionRecords` and `job/H2BackupJob`. FQL builds its SQL in `queryexplorer/query/QueryExecutionEngine` over 5 query tables (Accounts, Budgets, Categories, Investments, Transactions) |
| Owner stamping trigger (ADR F2) | The trigger `trg_stamp_owner_id` (V34, V35) is on **23 tables**: account, account_balance_snapshot, akahu_account, budget_envelope, budget_envelope_category, category, category_item, csv_import_file_snapshot, csv_import_profile, events, import_history, institution, journal_entry, ledger_node, ledger_node_mapping, loan, loan_balance_snapshot, loan_interest_period, loan_min_payment_period, sub_category, transaction, transaction_association, transaction_attachment. Insert paths into these tables may rely on the trigger instead of setting `owner_id` themselves |

## 3. Summary

| Class | Tables | Of which need their own `workspace_id` |
|---|---|---|
| WORKSPACE | 47, of which 4 are RETIRED | 39 (the other 8: 4 inherit through their parent, 4 are retired) |
| USER | 24, of which 1 is LEGACY | 0 owned. `api_token` and `oauth_grant` gain a workspace **scope** (D-14), not ownership |
| REFERENCE | 15, of which 2 are LEGACY and 1 is blob storage (4.5) | 0 |
| AMBIGUOUS, needing a Product Owner decision | 12 | Depends on the answer (section 6) |
| **Total** | **98** | |

Counts exclude `flyway_schema_history`. The 12 AMBIGUOUS tables are listed in section 4.4 with a recommendation; they are not counted in the three classes above until decided.

## 4. Inventory

**Legend:**
- **Ownership** is the current mechanism: `owner_id` (FK to `users`, `NOT NULL` unless marked nullable), a user FK, inherited through a parent, or global.
- **Trg** means the table carries the owner stamping trigger.
- **Sens:** F is financial data, C is a credential or secret, P is other personal data.
- **Repo** names the primary JPA repository.
- **SQL** names classes with JPQL/native or `JdbcTemplate` SQL that names the table.

### 4.1 WORKSPACE

| Table | Module | Ownership | Parents and FKs | `workspace_id` | Sens | Code | Migration notes and risks |
|---|---|---|---|---|---|---|---|
| account | ledger (account) | `owner_id` FK, Trg | institution (FK) | Yes | F | `ledger/account/Account`, `AccountRepository`; FQL `AccountsQueryTable` | Unique `uk_account_owner_source_ref (owner_id, source, source_ref)` must become workspace-scoped (R2-8). The `ownership` column (OWNED/EXTERNAL, `model/Ownership`) is an account kind, **not tenancy**: a false friend for a search on "ownership" |
| account_balance_snapshot | ledger (account) | `owner_id` FK, Trg | **`account_id` has no FK** | Yes | F | `AccountBalanceSnapshot`, `AccountBalanceSnapshotRepository` | Add a composite FK to account in R2-7. Orphans are possible today |
| transaction | ledger (transaction) | `owner_id` FK, Trg | account, category, sub_category, merchant, import_history, recurring_series (SET NULL), self (parent, transfer_pair, refund_of) | Yes | F | `TransactionEntity`, `TransactionRepository` (24 `@Query`, 1 native, household-share JPQL joins); FQL `TransactionsQueryTable`; `ledger/loan/Loan` (SQL formula) | Largest table. `uk_transaction_owner_account_fit_id` must be rescoped. The self-references (transfer pairs, refunds, splits) need the same-workspace check. `ownership` column, as on account. Backfill must be batched (TD-8) |
| transaction_tag | ledger (transaction) | **Inherited** (collection of `TransactionEntity`) | transaction (CASCADE) | No, through transaction | F | `@CollectionTable` on `TransactionEntity`; `TransactionTagsEraser` (JDBC) | Missed by an `owner_id` search |
| transaction_attachment | ledger (transaction) | `owner_id` FK, Trg | transaction (CASCADE); `storage_key` → attachment_blob | Yes | F | `TransactionAttachment`, `TransactionAttachmentRepository`, `TransactionAttachmentFilesEraser` | The blob is reached through this row's key. Blob access must check the row's workspace (see attachment_blob) |
| transaction_association | ledger (transaction) | `owner_id` FK, Trg | transaction ×2 (source, related) | Yes | F | `TransactionAssociation`, `TransactionAssociationRepository` | Both ends must be in the same workspace (composite FK) |
| import_history | ledger (transaction) | `owner_id` **nullable**, Trg | **`account_id` and `profile_id` have no FK** | Yes | F | `ImportHistory`, `ImportHistoryRepository`, `TransactionRepository` | Nullable `owner_id`: rows with null would not backfill. **Count nulls before R2.** Replaced by `import_job` in ingestion 6a; migrate it anyway, because transactions reference it |
| category | ledger (category) | `owner_id` FK, Trg | none | Yes | – | `Category`, `CategoryRepository` (household-share JPQL); FQL `CategoriesQueryTable` | `uk_category_owner_name` is rescoped. Every user has their own taxonomy (no global category rows). A Household starts with a taxonomy per U25 |
| sub_category | ledger (category) | `owner_id` FK, Trg | category | Yes | – | `SubCategory`, `SubCategoryRepository`, `TransactionRepository` | A child must share its parent's workspace |
| category_item | ledger (category) | `owner_id` FK, Trg | sub_category | Yes | – | `CategoryItem`, `CategoryItemRepository` | |
| merchant | ledger (merchant) | `owner_id` FK | sub_category (default) | Yes | – | `Merchant`, `MerchantRepository` (SQL) | `uk_merchant_owner_name` is rescoped |
| merchant_pattern | ledger (merchant) | `owner_id` FK | merchant (CASCADE) | Yes | – | `MerchantPattern`, `MerchantPatternRepository` | `uk_merchant_pattern_owner_pattern` is rescoped |
| loan | ledger (loan) | `owner_id` FK, Trg | account (CASCADE), paying_account (SET NULL), loan_group (SET NULL) | Yes | F | `Loan`, `LoanRepository` (household-share JPQL), `LoanEraser` | `uk_loan_owner_source_ref` is rescoped. `account_id ON DELETE CASCADE` deletes a loan with its account |
| loan_balance_snapshot | ledger (loan) | `owner_id` FK, Trg | loan (CASCADE) | Yes | F | `LoanBalanceSnapshot`, repository | |
| loan_interest_period | ledger (loan) | `owner_id` FK, Trg | loan (CASCADE) | Yes | F | `LoanInterestPeriod`, repository | Written directly by the Akahu connector (ADR A10) |
| loan_min_payment_period | ledger (loan) | `owner_id` FK, Trg | loan (CASCADE) | Yes | F | `LoanMinPaymentPeriod`, repository | |
| loan_group | ledger (loan) | `owner_id` FK | manual_asset (SET NULL) | Yes | F | `LoanGroup`, `LoanGroupRepository` | The cross-module FK to `manual_asset` must stay within one workspace |
| loan_plan_scenario | ledger (loan) | `owner_id` FK | none (scenario JSON) | Yes | F | `LoanPlanScenario`, repository | Saved simulations. Workspace, so Household members share plans |
| recurring_series | ledger (recurring) | `owner_id` FK | merchant (CASCADE), sub_category (CASCADE), loan (SET NULL), self ×3 (SET NULL) | Yes | F | `RecurringSeries`, `RecurringSeriesRepository`, `RecurringSeriesEraser` (JDBC) | Self-references within a workspace |
| recurring_series_previous_identity | ledger (recurring) | **Inherited** (collection) | recurring_series (CASCADE) | No, through parent | – | `@CollectionTable` on `RecurringSeries` | Missed by an `owner_id` search |
| budget_envelope | budget | `owner_id` FK, Trg | none | Yes | F | `BudgetEnvelope`, repository; FQL `BudgetsQueryTable` | `uk_budget_envelope_owner_name` is rescoped. Per workspace, never consolidated (U35) |
| budget_envelope_category | budget | `owner_id` FK, Trg | budget_envelope, category, sub_category | Yes | – | `BudgetEnvelopeCategoryLink`, repository | All three parents must share the workspace |
| balance_assertion | balance | `owner_id` FK | **`account_id` has no FK** | Yes | F | `BalanceAssertion`, repository, `BalanceAssertionEraser` (JDBC) | Add the composite FK in R2-7 |
| events | event | `owner_id` FK, Trg | none | Yes | – | `event/Event`, `EventRepository` | Table name `events` (plural) |
| event_transactions | event | **Inherited** (collection) | events, transaction | No, through events | – | `@JoinTable` on `Event`, `EventLinksEraser` (JDBC) | An event and its transactions must share the workspace. Missed by an `owner_id` search |
| goal | goal | `owner_id` FK | none | Yes | F | `Goal`, `GoalRepository` (SQL) | |
| goal_contribution | goal | `owner_id` FK | goal (CASCADE) | Yes | F | `GoalContribution` (no repository; through `Goal`) | |
| manual_asset | asset | `owner_id` FK | none | Yes | F | `ManualAsset`, repository, `AssetEraser` | |
| asset_valuation | asset | `owner_id` FK | manual_asset (CASCADE) | Yes | F | `AssetValuation`, repository | |
| holding | investment | `owner_id` FK | account | Yes | F | `Holding`, `HoldingRepository` (SQL), `HoldingService` (SQL); FQL `InvestmentsQueryTable` | `account_id` link must be in the same workspace |
| holding_transaction | investment | `owner_id` FK | **`holding_id` has no FK** | Yes | F | `HoldingTransaction`, repository | Add the FK in R2-7 |
| holding_value_snapshot | investment | `owner_id` FK | **`holding_id` has no FK** | Yes | F | `HoldingValueSnapshot`, repository | Add the FK in R2-7 |
| institution | institution | `owner_id` FK, Trg | none | Yes | – | `Institution`, `InstitutionRepository` (SQL) | `uk_institution_owner_name` is rescoped. Per user today, not global |
| forecast_item_label | forecast | `owner_id` FK | **`item_id` is polymorphic, no FK** | Yes | – | `ForecastItemLabel`, repository | The label must follow its item's workspace; the backfill relies on owner = item owner |
| dashboard | dashboard | `owner_id` FK | none | Yes (U27 (a)) | – | `Dashboard`, repository, `DashboardEraser` | Shared in a Household per the U27 recommendation (still pending) |
| dashboard_component | dashboard | `owner_id` FK | none | Yes (U27) | – | `DashboardComponent`, repository | |
| dashboard_widget | dashboard | `owner_id` FK | dashboard (CASCADE), dashboard_component | Yes (U27) | – | `DashboardWidget`, repository | |
| saved_query | queryexplorer | `owner_id` FK | none | Yes (U27) | – | `SavedQuery`, repository, `SavedQueryEraser` | |
| receipt | receipt | `owner_id` FK | transaction (SET NULL), transaction_attachment (SET NULL); `storage_key` → attachment_blob | Yes | F | `Receipt`, `ReceiptRepository`, `ReceiptAttachmentLookup` (SQL), `ReceiptEraser` | A receipt and its transaction must share the workspace |
| receipt_line | receipt | `owner_id` FK | receipt (CASCADE), category, sub_category (SET NULL) | Yes | F | `ReceiptLine` (through `Receipt`) | Category references must be in the same workspace |
| receipt_photo | receipt | **Inherited** (collection) | receipt (CASCADE); `storage_key` → attachment_blob | No, through receipt | F | `@CollectionTable` on `Receipt`, `ReceiptFilesEraser` | Missed by an `owner_id` search |
| receipt_ignored_attachment | receipt | `owner_id` FK | transaction_attachment (CASCADE) | Yes | – | `ReceiptIgnoredAttachment`, repository | |
| csv_import_file_snapshot | csvimport | `owner_id` **nullable**, Trg | **`account_id` and `profile_id` have no FK** | Yes | F | `CsvImportFileSnapshot`, repository | Nullable `owner_id`: count nulls before R2 |
| journal_entry | (retired ledger) | `owner_id` FK, Trg | none | **RETIRED**: drop recommended | F | No entity. `user/RetiredLedgerEraser` (JDBC) only | Internal accounting ledger (DEBT-011, U31) |
| ledger_node | (retired ledger) | `owner_id` FK, Trg | self (parent) | RETIRED | – | `RetiredLedgerEraser` only | As above |
| ledger_node_mapping | (retired ledger) | `owner_id` FK, Trg | ledger_node; **`entity_id` is polymorphic, no FK** | RETIRED | – | `RetiredLedgerEraser` only | As above |
| posting | (retired ledger) | **Inherited** | journal_entry, ledger_node | RETIRED | F | `RetiredLedgerEraser` only | As above |

### 4.2 USER

| Table | Module | Ownership | Parents and FKs | Workspace column | Sens | Code | Notes |
|---|---|---|---|---|---|---|---|
| users | user | identity | none | No. Settings columns: see Q-7 | C, P | `User`, `UserRepository` (native SQL) | Holds `password_hash`, `two_factor_secret`, `notification_unsubscribe_token`, the deletion fence and `active_context`. `base_currency` moves to `workspace` (D-39). `avatar_storage_key` → attachment_blob |
| auth_token | user | `user_id` FK | users | No | C | `AuthToken`, repository | Session and auth tokens (hashed) |
| two_factor_recovery_code | user | `user_id` FK (CASCADE) | users | No | C | `TwoFactorRecoveryCode`, repository | |
| user_oauth_account | user | `user_id` FK | users | No | P | `UserOAuthAccount`, repository | Google and Facebook identities |
| api_token | user | `owner_id` FK | users | **Scope** column `workspace_id` (D-14, R1-11) | C | `ApiToken`, `ApiTokenRepository` | User-owned; the scope limits which workspace it acts in |
| oauth_grant | oauthserver | `owner_id` FK | users; `client_id` has no FK to oauth_client | **Scope** `workspace_id` (D-14) | C | `OAuthGrant`, repository | MCP OAuth grants |
| oauth_access_token | oauthserver | **Inherited** | oauth_grant (CASCADE) | No | C | `OAuthAccessToken`, repository | Missed by an `owner_id` search |
| oauth_authorization_code | oauthserver | **Inherited** | oauth_grant (CASCADE) | No | C | `OAuthAuthorizationCode`, repository | Missed by an `owner_id` search |
| oauth_refresh_token | oauthserver | **Inherited** | oauth_grant (CASCADE) | No | C | `OAuthRefreshToken`, repository | Missed by an `owner_id` search |
| subscription | subscription | `owner_id` FK, unique | users; `granted_by` has no FK | No (D-13) | P | `Subscription`, repository, `StripeSubscriptionSyncService` (SQL), `SubscriptionEraser` | A Household's entitlement is read from its OWNER's row |
| beta_access | subscription | `owner_id` FK | users; `granted_by` has no FK | No | – | `BetaAccess`, repository, `BetaAccessEraser` | |
| stripe_customer | connectors/stripe | `owner_id` FK, unique | users | No | P | `StripeCustomer`, repository, `StripeCustomerRowEraser` | |
| stripe_checkout_session | connectors/stripe | `owner_id` FK | users | No | P | No entity: `StripeCheckoutSessionLog` (JDBC) | Missed by an entity search |
| ai_credit_use | ai | `owner_id` FK | users | No | – | `AiCreditUse`, repository, `AiCreditUseEraser` | AI metering against the user's plan |
| receipt_reader_call | receipt | `owner_id` FK | users; receipt (SET NULL) | No (metering) | – | `ReceiptReaderCall`, repository, `ReceiptReaderCallEraser` | Usage metering. The receipt link may point into a Household workspace; that is acceptable, read only for usage |
| grocery_ai_call | grocery | `owner_id` FK | users | No (metering) | – | `GroceryAiCall`, repository, `GroceryEraser` | |
| notification | notification | `owner_id` FK | users | No | P | `Notification`, `NotificationRepository` (SQL), `NotificationEraser` | A notification about Household data still goes to one person |
| notification_preference | notification | `owner_id` FK | users | No | – | `NotificationPreference`, repository | |
| feedback_submission | feedback | `owner_id` **has no FK** | none; `screenshot_storage_key` → attachment_blob | No | P | `FeedbackSubmission`, repository | Kept on deletion per U14 (pending). No FK is deliberate |
| feedback_message | feedback | `owner_id` **has no FK** | feedback_submission (CASCADE) | No | P | `FeedbackMessage`, repository | As above |
| feedback_message_image | feedback | **Inherited** | feedback_message (CASCADE); `storage_key` → attachment_blob | No | P | `FeedbackMessageImage`, repository | Missed by an `owner_id` search |
| user_akahu_connection | connectors/akahu | `user_id` FK | users | No (ADR 28.8) | **C** | `UserAkahuConnection`, repository, `AkahuConnectionEraser` | The person's own Akahu app token and encrypted user token. Never workspace-owned. A Household account fed by it is the ingestion question C-3 |
| akahu_account | connectors/akahu | `owner_id` FK, Trg | users; `external_id` has no FK | No (provider record of the connection, D-8) | F | `AkahuAccount`, repository, `AkahuAccountEraser` | `uk_akahu_account_owner_external_id`. Replaced by `source_account` in 6a; removed in R6 |
| household_membership | ledger (household) | `user_id` FK **nullable**, `household_owner_id` FK | household, users ×2 | **LEGACY** | P | `HouseholdMembership`, repository, `HouseholdEraser` | Not migrated (Q3). Removed in R6. A nullable `user_id` is a pending invite by code |

### 4.3 REFERENCE (global, system, operational)

| Table | Module | Ownership | Notes |
|---|---|---|---|
| exchange_rate | currency | global | Shared rates |
| feature_config | subscription | global | Plan caps (`free_cap`, `plus_cap`, `pro_cap`). Workspace-agnostic: a Household reads its OWNER's plan |
| plan_price | subscription | global | Stripe price ids |
| payment_settings | subscription | global | |
| ai_credit_plan | ai | global | |
| ai_credit_setting | ai | global | |
| ai_model_price | ai | global | |
| oauth_client | oauthserver | global | Registered MCP clients |
| stripe_webhook_event | connectors/stripe | global | Idempotency of webhook ids |
| outbox_task | outbox | system | Holds no user id by design (Phase 0b) |
| account_deletion | user | system | Holds no user id by design (U30) |
| admin_audit_entry | admin | system | `actor_user_id` and `target_user_id` **have no FK** by design: entries outlive deleted users. Not workspace data (tier 1, D-38) |
| household | ledger (household) | `owner_id` FK | **LEGACY**: not migrated (Q3, U11). One row in production (U47); removed in R6 |
| household_resource_share | ledger (household) | inherited (household); `resource_id` is **polymorphic with no FK** | **LEGACY**: removed in R6. The ACCOUNT and LOAN READ shares are not carried over (C-1, C-4) |

Note: `household` and `household_resource_share` are listed here because they will not be classified for migration. They are legacy and are dropped in R6.

### 4.4 AMBIGUOUS (Product Owner decision needed, not classified)

| Table | Module | Ownership | Parents | Sens | Recommendation | Question |
|---|---|---|---|---|---|---|
| csv_import_profile | csvimport | `owner_id` FK, Trg | account (`default_account_id`) | – | WORKSPACE, because it points to an account | Q-1 |
| receipt_item_rule | receipt | `owner_id` FK | sub_category (CASCADE) | – | WORKSPACE, following the taxonomy it points into | Q-2 |
| receipt_product_category | receipt | `owner_id` FK | **`category_id` has no FK** | – | WORKSPACE, as above | Q-2 |
| receipt_product_link | receipt | `owner_id` FK | none | – | WORKSPACE, as above | Q-2 |
| coach_conversation | coach | `owner_id` FK | none | F, P | USER (U28 (a): private even inside a Household), with the workspace it was about stored as context | Q-3 |
| coach_message | coach | `owner_id` FK | coach_conversation (CASCADE) | F, P | Follows its conversation | Q-3 |
| coach_prompt | coach | `owner_id` FK | coach_conversation (SET NULL) | F, P | Follows its conversation | Q-3 |
| grocery_item | grocery | `owner_id` FK | none | – | WORKSPACE: a shared shopping list is the natural Household case | Q-4 |
| grocery_aisle_choice | grocery | `owner_id` FK | none | – | Follows grocery_item | Q-4 |
| grocery_suggestion_skip | grocery | `owner_id` FK | none | – | Follows grocery_item | Q-4 |
| chef_recipe | grocery | `owner_id` FK | none | – | USER, or WORKSPACE with grocery | Q-4 |
| chef_message | grocery | `owner_id` FK | chef_recipe (CASCADE) | – | Follows chef_recipe | Q-4 |

### 4.5 Storage

| Table | Module | Ownership | Notes |
|---|---|---|---|
| attachment_blob | storage | **No ownership column.** Keyed by a random `storage_key`, referenced without FKs from `transaction_attachment`, `receipt`, `receipt_photo`, `feedback_submission`, `feedback_message_image` and `users` (avatar) | Counted under REFERENCE in section 3 because it has no owner of its own. It is **not** shared data: every read must go through an owning row that passes the workspace or user check, never by key alone. R3 must test that a blob key from another workspace is refused. Missed by an `owner_id` search |

## 5. Tables a search for `owner_id` misses

1. **Ownership inherited through a parent or collection (11):**
   - transaction_tag, event_transactions, recurring_series_previous_identity, receipt_photo, posting;
   - oauth_access_token, oauth_authorization_code, oauth_refresh_token;
   - feedback_message_image;
   - household_resource_share;
   - attachment_blob, which is keyed by storage key only.
2. **User-keyed under another name (5):** `user_id` on auth_token, two_factor_recovery_code, user_oauth_account and user_akahu_connection; `user_id` and `household_owner_id` on household_membership.
3. **Tables with no JPA entity of their own (7, plus 4 collections):** journal_entry, ledger_node, ledger_node_mapping, posting, stripe_checkout_session, outbox_task and account_deletion; and transaction_tag, event_transactions, receipt_photo and recurring_series_previous_identity, which are mapped only through `@CollectionTable` or `@JoinTable`.
4. **Owner columns with no FK (6):** `feedback_submission.owner_id`, `feedback_message.owner_id`, `subscription.granted_by`, `beta_access.granted_by`, and `admin_audit_entry.actor_user_id` and `target_user_id`.
5. **Parent ids with no FK** (orphan and cross-tenant risk once workspaces exist):
   - `account_balance_snapshot.account_id`
   - `balance_assertion.account_id`
   - `holding_transaction.holding_id`
   - `holding_value_snapshot.holding_id`
   - `import_history.account_id` and `.profile_id`
   - `csv_import_file_snapshot.account_id` and `.profile_id`
   - `receipt_product_category.category_id`
   - `forecast_item_label.item_id` (polymorphic)
   - `ledger_node_mapping.entity_id` (polymorphic)
   - `household_resource_share.resource_id` (polymorphic)
6. **Code that filters without the column name:**
   - JPQL and derived queries use `ownerId`, not `owner_id`: about 100 owner-keyed methods in 45 repositories.
   - Household-share JPQL joins in `TransactionRepository`, `CategoryRepository` and `LoanRepository`.
   - FQL builds its own SQL in `QueryExecutionEngine`.
   - The owner stamping trigger fills `owner_id` on 23 tables without the code naming it.
7. **False friend:** the `ownership` column on account and transaction is the OWNED/EXTERNAL account kind, not tenancy.
8. **Cross-tenant admin reads:** `admin/AdminMetricsRepository` counts across many tables by design (tier 1). It must stay an aggregate and never become a per-workspace data read.

## 6. Classification questions for the Product Owner

| ID | Question | Recommendation |
|---|---|---|
| Q-1 | CSV import profiles (`csv_import_profile`): per workspace (shared in a Household, pointing to Household accounts) or per user? | Per workspace. The profile names a default account, which is workspace data |
| Q-2 | Learned receipt rules (`receipt_item_rule`, `receipt_product_category`, `receipt_product_link`): per workspace or per user? | Per workspace. They map products into the workspace's own taxonomy, and a per-user rule could point at another workspace's category |
| Q-3 | Coach conversations: confirm U28 (a), private per user even inside a Household? | Yes. USER-owned, with a context field recording which workspace the conversation was about. Answers may only draw on workspaces the user can still access |
| Q-4 | Grocery list and Chef recipes: shared in a Household, or personal? | The grocery list is per workspace (shared shopping list). Chef recipes are per user. Confirm |
| Q-5 | Dashboards and saved queries: confirm U27 (a), shared with all members? | Yes, per workspace |
| Q-6 | Retired ledger tables (`journal_entry`, `ledger_node`, `ledger_node_mapping`, `posting`): drop them (U31 (b)) before R2 instead of migrating them? | Drop them in a dedicated PR before R2. They have no entity and only an eraser touches them. Confirm they hold no data anyone needs (row count on production, read-only) |
| Q-7 | Settings on `users` that act on financial data: `base_currency` moves to the workspace (D-39). What about `merchant_auto_categorize_enabled`, `budget_totals_per_period` and `home_page`? | `base_currency` and `budget_totals_per_period` go to the workspace. `merchant_auto_categorize_enabled` is per workspace. Layout and UI preferences stay on the user |
| Q-8 | Nullable `owner_id` on `import_history` and `csv_import_file_snapshot`: is a null row valid? | Treat null as an anomaly. Count nulls (read-only) before R2. A data fix, if needed, is a hand-run script, never a migration |
| Q-9 | Child tables that inherit through a parent (transaction_tag, event_transactions, receipt_photo, recurring_series_previous_identity): own `workspace_id` column or not? | No own column. They inherit through a `CASCADE` FK to a parent that carries `workspace_id`. ADR D-6 says "every tenant-owned row"; this needs to be confirmed as an allowed exception |

## 7. Dependencies and risks for R1 and R2

**R1**
- `users` keeps identity and the deletion fence. The PERSONAL workspace takes `base_currency` (D-39) and any settings Q-7 moves.
- `api_token` and `oauth_grant` need the scope column, and the three OAuth token tables inherit it through `oauth_grant`.
- Registration paths must create the PERSONAL workspace: local registration, OAuth and the admin-created user, all in `user`.
- The workspace module cannot depend on `ledger`. The `ledger` module's legacy `household` code must not be reused for workspace memberships (C-4).

**R2**
1. **Scale:** 39 tables need their own `workspace_id` (47 WORKSPACE minus 8 inherited or retired), plus up to 12 more depending on section 6. That is one PR per module group, leaves first, following ADR TD-8.
2. **Missing FKs first:** add the parent FKs listed in section 5 item 5 (as composite FKs with `workspace_id`), or the backfill can join an orphan into the wrong workspace.
3. **The trigger:** 23 tables rely on `trg_stamp_owner_id`. `workspace_id` has no trigger (ADR 15.2), so every insert path into those tables must set it explicitly before R2-6 makes it `NOT NULL`. Find and test those insert paths first: an insert that relied on the trigger fails with `NOT NULL` in production, not in a test that sets the owner.
4. **Unique keys:** 23 unique indexes include `owner_id`. Those on WORKSPACE tables are rescoped: account, budget_envelope, category, forecast_item_label, institution, loan, merchant, merchant_pattern, receipt_ignored_attachment and transaction, plus csv_import_profile and the receipt and grocery ones if section 6 makes them WORKSPACE. The rest (akahu_account, beta_access, notification, notification_preference, stripe_customer, subscription) stay on the user, and `ledger_node_mapping` is retired. Keep the old index until R6, to avoid a window without uniqueness.
5. **Cross-module FKs:** these are the edges where a composite FK or a same-workspace check is needed:
   - `holding.account_id`
   - `loan_group.asset_id`
   - `receipt.transaction_id`
   - `receipt_line.category_id`
   - `budget_envelope_category` (three parents)
   - `csv_import_profile.default_account_id`
   - `event_transactions`
6. **Retired ledger:** migrating 4 dead tables wastes effort and keeps a dead module alive. Resolve Q-6 first.
7. **Blobs:** `attachment_blob` has no owner. Any blob read must be authorized through its referencing row (R3).
8. **Production volume (R0-13):** row counts for `transaction`, `account_balance_snapshot` and `receipt_line`, and the null counts of Q-8, are needed to size the batched backfills.

## 8. Evidence

- Schema: `src/main/resources/db/migration/V1` to `V179` applied to PostgreSQL 16; information_schema queries for tables (98), FKs (139), triggers (`trg_stamp_owner_id` on 23 tables), unique indexes containing `owner_id` (23).
- Owner stamping: `V34__stamp_owner_id_from_session.sql`, `V35__keep_owner_id_on_update.sql`.
- Recent tables:
  - `V174__owner_foreign_keys_to_users.sql` (most `owner_id` FKs);
  - `V178__account_deletion_fence_and_outbox.sql` (`account_deletion`, `outbox_task`, fence columns);
  - `V179__stripe_checkout_session.sql`.
- Code: entity, `@Table`, `@CollectionTable` and `@JoinTable` mapping, and a search for SQL naming each table across `src/main/java` (JPQL, native and `JdbcTemplate`). Native queries are in `TransactionRepository`, `UserRepository` and `HouseholdRepository`. FQL query tables are in `queryexplorer/query/*QueryTable`.
