# R0-8: Table inventory for workspace tenancy

| Field | Value |
|---|---|
| Roadmap task | [R0-8](WORKSPACE-TENANCY-ROADMAP.md#r0-preparation-and-baseline) |
| Decisions | [ADR-001](ADR-001-workspace-tenancy-and-bank-ingestion.md) section 24 is authoritative. This document classifies tables; it decides nothing. The Product Owner's answers to Q-1 to Q-9 are recorded in section 6.1 and in ADR-001 section 28.9 |
| Code inspected | fern-funds `master` at `c6447fc5` (2026-11-16); the review clarifications of section 9 at the same commit (2026-11-17) |
| Status | **VERIFIED (2026-11-17).** Product Owner review of `e17a4ed` accepted the method and classification as the baseline, decided Q-1 to Q-9 (section 6.1), and asked for four clarifications, answered in section 9. Documentation only: no schema, entity, migration or behaviour change, and no production access |
| Earlier status | Analysis only, awaiting Product Owner review (`e17a4ed`, 2026-11-16) |

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

After the Product Owner decisions of 2026-11-17 (section 6.1) and the clarifications of section 9:

| Class | Tables | Of which need their own `workspace_id` |
|---|---|---|
| WORKSPACE | 54, of which 4 are RETIRED | **47.** The other 7: 3 inherit through a single parent (Q-9, section 9.3) and 4 are retired (Q-6) |
| USER | 29, of which 1 is LEGACY | 0 owned. `api_token` and `oauth_grant` gain a workspace **scope** (D-14), and `coach_conversation` a workspace **context** (Q-3), not ownership |
| REFERENCE | 15, of which 2 are LEGACY and 1 is blob storage (4.5) | 0 |
| AMBIGUOUS | 0 | All 12 decided |
| **Total** | **98** | |

Where the 12 formerly AMBIGUOUS tables went:
- **WORKSPACE (7):** `csv_import_profile` (Q-1); `receipt_item_rule`, `receipt_product_category`, `receipt_product_link` (Q-2); `grocery_item` (Q-4), and `grocery_aisle_choice` and `grocery_suggestion_skip` (Q-4, with section 9.1).
- **USER (5):** `coach_conversation`, `coach_message`, `coach_prompt` (Q-3); `chef_recipe`, `chef_message` (Q-4).

One change inside WORKSPACE: `event_transactions` moves from "inherits, no column" to "own `workspace_id`", because a join between two parents cannot meet the Q-9 condition otherwise (section 9.2). So 39 + 7 + 1 = **47** tables need their own column.

Counts exclude `flyway_schema_history`.

### 3.1 Summary as submitted for review (`e17a4ed`, kept for the record)

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
| transaction_tag | ledger (transaction) | **Inherited** (collection of `TransactionEntity`) | transaction (CASCADE) | No, through transaction (Q-9; guarantee in 9.3) | F | `@CollectionTable` on `TransactionEntity`; `TransactionTagsEraser` (JDBC) | Missed by an `owner_id` search |
| transaction_attachment | ledger (transaction) | `owner_id` FK, Trg | transaction (CASCADE); `storage_key` → attachment_blob | Yes | F | `TransactionAttachment`, `TransactionAttachmentRepository`, `TransactionAttachmentFilesEraser` | The blob is reached through this row's key. Blob access must check the row's workspace (see attachment_blob) |
| transaction_association | ledger (transaction) | `owner_id` FK, Trg | transaction ×2 (source, related) | Yes | F | `TransactionAssociation`, `TransactionAssociationRepository` | Both ends must be in the same workspace (composite FK) |
| import_history | ledger (transaction) | `owner_id` **nullable**, Trg | **`account_id` and `profile_id` have no FK** | Yes | F | `ImportHistory`, `ImportHistoryRepository`, `TransactionRepository` | Nullable `owner_id`: rows with null would not backfill. **Count nulls before R2.** A null is an anomaly (Q-8): no invented workspace, no automatic repair. Replaced by `import_job` in ingestion 6a; migrate it anyway, because transactions reference it |
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
| recurring_series_previous_identity | ledger (recurring) | **Inherited** (collection) | recurring_series (CASCADE) | No, through parent (Q-9; guarantee in 9.3) | – | `@CollectionTable` on `RecurringSeries` | Missed by an `owner_id` search |
| budget_envelope | budget | `owner_id` FK, Trg | none | Yes | F | `BudgetEnvelope`, repository; FQL `BudgetsQueryTable` | `uk_budget_envelope_owner_name` is rescoped. Per workspace, never consolidated (U35) |
| budget_envelope_category | budget | `owner_id` FK, Trg | budget_envelope, category, sub_category | Yes | – | `BudgetEnvelopeCategoryLink`, repository | All three parents must share the workspace |
| balance_assertion | balance | `owner_id` FK | **`account_id` has no FK** | Yes | F | `BalanceAssertion`, repository, `BalanceAssertionEraser` (JDBC) | Add the composite FK in R2-7 |
| events | event | `owner_id` FK, Trg | none | Yes | – | `event/Event`, `EventRepository` | Table name `events` (plural) |
| event_transactions | event | **Inherited** (join table, no `owner_id`) | events, transaction (both plain FKs, no CASCADE) | **Yes, integrity column** (section 9.2; was "No, through events" at `e17a4ed`) | – | `@JoinTable` on `Event`, `EventService` (assign, remove), `EventLinksEraser` (JDBC) | An event and its transactions must share the workspace: composite FKs `(event_id, workspace_id)` and `(transaction_id, workspace_id)` over one shared column. Today only `EventService` enforces it. Missed by an `owner_id` search |
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
| receipt_photo | receipt | **Inherited** (collection) | receipt (CASCADE); `storage_key` → attachment_blob | No, through receipt (Q-9; guarantee and the blob caveat in 9.3) | F | `@CollectionTable` on `Receipt`, `ReceiptFilesEraser` | Missed by an `owner_id` search |
| receipt_ignored_attachment | receipt | `owner_id` FK | transaction_attachment (CASCADE) | Yes | – | `ReceiptIgnoredAttachment`, repository | |
| receipt_item_rule | receipt | `owner_id` FK | sub_category (CASCADE) | Yes (Q-2) | – | `ReceiptItemRule`, repository | `uq_receipt_item_rule_owner_product` is rescoped. The sub-category must be in the same workspace |
| receipt_product_category | receipt | `owner_id` FK | **`category_id` has no FK** | Yes (Q-2) | – | `ReceiptProductCategory`, repository, `ReceiptProductService` | `uq_receipt_product_category_owner_category` is rescoped. Add the composite FK to category in R2-7 |
| receipt_product_link | receipt | `owner_id` FK | none (`product_key` text) | Yes (Q-2) | – | `ReceiptProductLink`, repository, `ReceiptProductService` | `uq_receipt_product_link` is rescoped. The product catalog it shapes is also what grocery suggestions read (9.1) |
| csv_import_file_snapshot | csvimport | `owner_id` **nullable**, Trg | **`account_id` and `profile_id` have no FK** | Yes | F | `CsvImportFileSnapshot`, repository | Nullable `owner_id`: count nulls before R2. A null is an anomaly (Q-8): no invented workspace, no automatic repair |
| csv_import_profile | csvimport | `owner_id` FK, Trg | account (`default_account_id`) | Yes (Q-1) | – | `CsvImportProfile`, repository | `uk_csv_import_profile_owner_name` is rescoped. The default account must be in the same workspace |
| grocery_item | grocery | `owner_id` FK | none (`product_key` is a soft reference to the receipt product catalog) | Yes (Q-4) | – | `GroceryItem`, `GroceryItemRepository`, `GroceryService`, `GroceryEraser` | The shared shopping list |
| grocery_aisle_choice | grocery | `owner_id` FK | **none**: not a child of grocery_item (9.1) | Yes (Q-4, 9.1) | – | `GroceryAisleChoice`, repository, `GroceryService`, `GroceryEraser` | `uq_grocery_aisle_choice_owner_term` becomes `(workspace_id, term)`. The list's learned aisle per item wording |
| grocery_suggestion_skip | grocery | `owner_id` FK | **none**: not a child of grocery_item (9.1) | Yes (Q-4, 9.1) | – | `GrocerySuggestionSkip`, repository, `GroceryService`, `GroceryEraser` | `uq_grocery_suggestion_skip_owner_product` becomes `(workspace_id, product_key)`. A "not now" on a running-low suggestion |
| journal_entry | (retired ledger) | `owner_id` FK, Trg | none | **RETIRED**: not migrated; dropped in a dedicated cleanup change (Q-6, conditional; 9.4) | F | No entity. `user/RetiredLedgerEraser` (JDBC) only | Internal accounting ledger (DEBT-011, U31) |
| ledger_node | (retired ledger) | `owner_id` FK, Trg | self (parent) | RETIRED | – | `RetiredLedgerEraser` only | As above |
| ledger_node_mapping | (retired ledger) | `owner_id` FK, Trg | ledger_node; **`entity_id` is polymorphic, no FK** | RETIRED | – | `RetiredLedgerEraser` only | As above |
| posting | (retired ledger) | **Inherited** | journal_entry, ledger_node | RETIRED | F | `RetiredLedgerEraser` only | As above |

### 4.2 USER

| Table | Module | Ownership | Parents and FKs | Workspace column | Sens | Code | Notes |
|---|---|---|---|---|---|---|---|
| users | user | identity | none | No. Settings columns decided by Q-7 | C, P | `User`, `UserRepository` (native SQL) | Holds `password_hash`, `two_factor_secret`, `notification_unsubscribe_token`, the deletion fence and `active_context`. **Move to `workspace` (Q-7):** `base_currency` (D-39), `budget_totals_per_period`, `merchant_auto_categorize_enabled`. **Stay on the user:** `home_page` and the visual and layout preferences (`theme_preference`, `display_mode`). `avatar_storage_key` → attachment_blob |
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
| coach_conversation | coach | `owner_id` FK | users | **Context** column: the workspace the conversation is about (Q-3). Not ownership | F, P | `CoachConversation`, `CoachConversationRepository`, `CoachService`, `CoachEraser` | Private to its user even inside a Household (Q-3, U28 (a)). Answers may only draw on workspaces the user can still access |
| coach_message | coach | `owner_id` FK | coach_conversation (CASCADE) | No, follows its conversation (Q-3) | F, P | `CoachMessage`, repository | |
| coach_prompt | coach | `owner_id` FK | coach_conversation (SET NULL) | No, follows its conversation (Q-3) | F, P | `CoachPrompt`, repository | |
| chef_recipe | grocery | `owner_id` FK | users | No (Q-4) | – | `ChefRecipe`, repository, `GroceryEraser`, MCP `GroceryTools` | A person's saved recipes, not the shared list |
| chef_message | grocery | `owner_id` FK | chef_recipe (CASCADE) | No, follows its recipe (Q-4) | – | `ChefMessage`, repository, `GroceryEraser` | |

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

### 4.4 Formerly AMBIGUOUS: decided by the Product Owner on 2026-11-17

Kept as submitted, with the outcome added. The decided rows also appear in 4.1 or 4.2, which are the working lists.

| Table | Module | Ownership | Parents | Sens | Recommendation | Question | Outcome |
|---|---|---|---|---|---|---|---|
| csv_import_profile | csvimport | `owner_id` FK, Trg | account (`default_account_id`) | – | WORKSPACE, because it points to an account | Q-1 | WORKSPACE (4.1) |
| receipt_item_rule | receipt | `owner_id` FK | sub_category (CASCADE) | – | WORKSPACE, following the taxonomy it points into | Q-2 | WORKSPACE (4.1) |
| receipt_product_category | receipt | `owner_id` FK | **`category_id` has no FK** | – | WORKSPACE, as above | Q-2 | WORKSPACE (4.1) |
| receipt_product_link | receipt | `owner_id` FK | none | – | WORKSPACE, as above | Q-2 | WORKSPACE (4.1) |
| coach_conversation | coach | `owner_id` FK | none | F, P | USER (U28 (a): private even inside a Household), with the workspace it was about stored as context | Q-3 | USER, with workspace context (4.2) |
| coach_message | coach | `owner_id` FK | coach_conversation (CASCADE) | F, P | Follows its conversation | Q-3 | USER (4.2) |
| coach_prompt | coach | `owner_id` FK | coach_conversation (SET NULL) | F, P | Follows its conversation | Q-3 | USER (4.2) |
| grocery_item | grocery | `owner_id` FK | none | – | WORKSPACE: a shared shopping list is the natural Household case | Q-4 | WORKSPACE (4.1) |
| grocery_aisle_choice | grocery | `owner_id` FK | none | – | Follows grocery_item | Q-4 | WORKSPACE, own `workspace_id`; the "follows grocery_item" wording was imprecise, see 9.1 |
| grocery_suggestion_skip | grocery | `owner_id` FK | none | – | Follows grocery_item | Q-4 | WORKSPACE, own `workspace_id`; see 9.1 |
| chef_recipe | grocery | `owner_id` FK | none | – | USER, or WORKSPACE with grocery | Q-4 | USER (4.2) |
| chef_message | grocery | `owner_id` FK | chef_recipe (CASCADE) | – | Follows chef_recipe | Q-4 | USER (4.2) |

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
   - `grocery_aisle_choice` and `grocery_suggestion_skip` reference **nothing**: no parent FK at all, only natural keys (`term`, `product_key`). They are classified on their own (9.1).
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

### 6.1 Product Owner decisions (2026-11-17)

Recorded from the Product Owner's review of `e17a4ed`. The method and overall classification were accepted as the baseline. ADR-001 section 28.9 records the same decisions in the ADR's register; where they differ, the ADR governs.

| ID | Decision | Kind | Rationale | Effect on the inventory |
|---|---|---|---|---|
| Q-1 | `csv_import_profile` is WORKSPACE-owned | Approved | The profile names a default account, which is workspace data; a per-user profile could point at another workspace's account | WORKSPACE, own `workspace_id`; unique key and default-account FK scoped to the workspace |
| Q-2 | `receipt_item_rule`, `receipt_product_category` and `receipt_product_link` are WORKSPACE-owned | Approved | They map products into the workspace's own taxonomy; a per-user rule could point at another workspace's category | WORKSPACE, own `workspace_id`; `receipt_product_category.category_id` gains a composite FK |
| Q-3 | Coach conversations stay USER-owned and private, with workspace context | Approved; decides U28 (a) | A conversation is closer to personal correspondence than to shared financial data | `coach_conversation`, `coach_message`, `coach_prompt` are USER. The conversation records the workspace it was about; it is never shared by being in a Household |
| Q-4 | Grocery lists are WORKSPACE-owned; Chef recipes and conversations are USER-owned | Approved | A shared shopping list is the natural Household case; recipes and the chat about them are personal | `grocery_item`, `grocery_aisle_choice`, `grocery_suggestion_skip` WORKSPACE (9.1); `chef_recipe`, `chef_message` USER. `grocery_ai_call` stays USER as metering |
| Q-5 | Dashboards and saved queries are WORKSPACE-owned | Approved; decides U27 (a) | Consistent with Q4 and Q6: a Household owns its resources and members see them by role | `dashboard`, `dashboard_component`, `dashboard_widget`, `saved_query` WORKSPACE, as already listed |
| Q-6 | The retired ledger tables are not migrated. They are dropped in a dedicated cleanup change, after a read-only production row-count and data-retention assessment | **Conditional approval**; decides U31 (b) | No entity, no reader and no writer remain; the data was a projection of transactions (9.4) | `journal_entry`, `ledger_node`, `ledger_node_mapping`, `posting` stay RETIRED and get no `workspace_id`. Nothing is dropped by this task and production was not accessed |
| Q-7 | `base_currency`, `budget_totals_per_period` and `merchant_auto_categorize_enabled` move to the workspace. `home_page` and the visual and layout preferences stay on the user | Approved; extends D-39 | They change how a workspace's money is shown or processed; the rest is how one person likes the app to look | `users` row in 4.2; the R1 backfill copies the three values into the PERSONAL workspace |
| Q-8 | A null `owner_id` on `import_history` or `csv_import_file_snapshot` is an anomaly | Approved, with conditions | A row with no owner has no defensible workspace | Read-only null counts before R2 (R0-13). No invented workspace assignment and no automatic repair. Any remediation needs a documented, reviewed and auditable plan |
| Q-9 | A child collection or join table may inherit workspace ownership without its own `workspace_id` | **Conditional approval**: only where database constraints guarantee parent ownership and prevent cross-workspace associations | Avoids redundant columns where the schema already makes a cross-workspace row impossible | `transaction_tag`, `receipt_photo`, `recurring_series_previous_identity` inherit (9.3). `event_transactions` cannot meet the condition without a column, so it gets one (9.2) |

## 7. Dependencies and risks for R1 and R2

**R1**
- `users` keeps identity and the deletion fence. The PERSONAL workspace takes `base_currency` (D-39) and any settings Q-7 moves.
- `api_token` and `oauth_grant` need the scope column, and the three OAuth token tables inherit it through `oauth_grant`.
- Registration paths must create the PERSONAL workspace: local registration, OAuth and the admin-created user, all in `user`.
- The workspace module cannot depend on `ledger`. The `ledger` module's legacy `household` code must not be reused for workspace memberships (C-4).

**R2**
1. **Scale:** **47** tables need their own `workspace_id` (54 WORKSPACE minus 3 inherited and 4 retired; section 3). That is one PR per module group, leaves first, following ADR TD-8. At `e17a4ed` this read: "39 tables need their own `workspace_id` (47 WORKSPACE minus 8 inherited or retired), plus up to 12 more depending on section 6".
2. **Missing FKs first:** add the parent FKs listed in section 5 item 5 (as composite FKs with `workspace_id`), or the backfill can join an orphan into the wrong workspace.
3. **The trigger:** 23 tables rely on `trg_stamp_owner_id`. `workspace_id` has no trigger (ADR 15.2), so every insert path into those tables must set it explicitly before R2-6 makes it `NOT NULL`. Find and test those insert paths first: an insert that relied on the trigger fails with `NOT NULL` in production, not in a test that sets the owner.
4. **Unique keys:** 23 unique indexes include `owner_id`. Those on WORKSPACE tables are rescoped: account, budget_envelope, category, forecast_item_label, institution, loan, merchant, merchant_pattern, receipt_ignored_attachment and transaction, plus, after Q-1, Q-2 and Q-4, csv_import_profile, receipt_item_rule, receipt_product_category, receipt_product_link, grocery_aisle_choice and grocery_suggestion_skip. The rest (akahu_account, beta_access, notification, notification_preference, stripe_customer, subscription) stay on the user, and `ledger_node_mapping` is retired. Keep the old index until R6, to avoid a window without uniqueness.
5. **Cross-module FKs:** these are the edges where a composite FK or a same-workspace check is needed:
   - `holding.account_id`
   - `loan_group.asset_id`
   - `receipt.transaction_id`
   - `receipt_line.category_id`
   - `budget_envelope_category` (three parents)
   - `csv_import_profile.default_account_id`
   - `event_transactions`: both parents, through one shared `workspace_id` column (9.2)
   - `receipt_product_category.category_id` (Q-2; has no FK today)
6. **Retired ledger:** not migrated (Q-6). Dropped in a dedicated cleanup change, after a read-only production row-count and data-retention assessment (9.4). That change is not part of R2 and nothing in this document authorizes it.
7. **Blobs:** `attachment_blob` has no owner. Any blob read must be authorized through its referencing row (R3).
8. **Production volume (R0-13), read-only:** row counts for `transaction`, `account_balance_snapshot` and `receipt_line` to size the batched backfills, plus three counts this review added:
   - the null `owner_id` rows of Q-8 in `import_history` and `csv_import_file_snapshot`;
   - `event_transactions` rows whose event and transaction have different `owner_id` (9.2);
   - row counts of the four retired ledger tables (Q-6, 9.4).
9. **Q-8 anomalies:** if any null `owner_id` row exists, R2 does not assign it a workspace. Remediation needs a documented, reviewed and auditable plan first; per the repository's rule, a per-user data fix is a hand-run script, never a migration.
10. **Coach context (Q-3):** the context column's lifecycle (what happens to it when its workspace is dissolved or the user leaves) is an R2 design point, not decided here.

## 8. Evidence

- Schema: `src/main/resources/db/migration/V1` to `V179` applied to PostgreSQL 16; information_schema queries for tables (98), FKs (139), triggers (`trg_stamp_owner_id` on 23 tables), unique indexes containing `owner_id` (23).
- Owner stamping: `V34__stamp_owner_id_from_session.sql`, `V35__keep_owner_id_on_update.sql`.
- Recent tables:
  - `V174__owner_foreign_keys_to_users.sql` (most `owner_id` FKs);
  - `V178__account_deletion_fence_and_outbox.sql` (`account_deletion`, `outbox_task`, fence columns);
  - `V179__stripe_checkout_session.sql`.
- Code: entity, `@Table`, `@CollectionTable` and `@JoinTable` mapping, and a search for SQL naming each table across `src/main/java` (JPQL, native and `JdbcTemplate`). Native queries are in `TransactionRepository`, `UserRepository` and `HouseholdRepository`. FQL query tables are in `queryexplorer/query/*QueryTable`.
- Review clarifications (section 9), read at fern-funds `c6447fc5`, no database or production access:
  - `V164__grocery_item_quantity_and_aisle.sql`, `V168__grocery_suggestion_skip.sql`, `V174__owner_foreign_keys_to_users.sql` (grocery owner FKs); `grocery/GroceryService.java`, `GroceryAisleChoice*`, `GrocerySuggestionSkip*`, `GroceryEraser`; `receipt/ReceiptProductService.java`.
  - `V1__initial_schema.sql` (`event_transactions`, `transaction_tag`), `V128__recurring_series_identity_by_description.sql`, `V137__receipt_photos.sql`; `event/Event.java`, `event/EventService.java`, `event/EventLinksEraser.java`; `websecurity/DefaultAuthorizationService.requireOwned`.
  - `V12`, `V13`, `V14`, `V16`, `V34`, `V35`, `V49` (retired ledger); fern-funds #720 (`6361abf3`, 2026-11-08); `user/RetiredLedgerEraser.java`; `AccountDeletionCoverageTest`, `OwnerScopingBoundaryTest`; `docs/ENGINEERING_DEBT.md` DEBT-011.

## 9. Review clarifications (2026-11-17)

The four clarifications the Product Owner required before R0-8 could be VERIFIED. Each states what the code and schema show today, and what R2 must build. Nothing here changes code, schema or data.

### 9.1 `grocery_aisle_choice` and `grocery_suggestion_skip`

**Relationship.** Neither table has a foreign key to `grocery_item` or to anything else except `owner_id → users` (V174). They are **not children** of `grocery_item`. The submitted inventory's "follows grocery_item" meant "follows the grocery list's classification", and that wording was imprecise.

| | `grocery_aisle_choice` (V164) | `grocery_suggestion_skip` (V168) |
|---|---|---|
| Key | `(owner_id, term)` unique; `term` is the normalized wording of an item (`GroceryAisle.term`) | `(owner_id, product_key)` unique; `product_key` is a soft reference into the receipt product catalog |
| Meaning | The aisle someone last moved an item with that wording to, so the next item with the same words lands there | A "not now" on a running-low suggestion, hidden while the product stays due on the same day |
| Written by | `GroceryService.update` → `rememberAisle`, when an item's aisle is changed | `GroceryService.skipSuggestion` |
| Read by | `GroceryService.add` → `aisleFor` (default aisle of a new item), and `GroceryService.month` | `GroceryService.suggestions`, which `addSuggestions` also uses |
| Erased by | `GroceryEraser`, by `owner_id` | `GroceryEraser`, by `owner_id` |
| Depends on | the shared list's items (Q-4) | suggestions computed from the workspace's receipts and product links (Q-2), minus the list's open items (Q-4) |

**Ownership semantics.** Both are learned state of the shopping list, not personal preferences: one decides where new list items go, the other filters the suggestions offered for that list. Both feed only list behaviour that Q-4 made WORKSPACE, from sources Q-2 made WORKSPACE.

**Classification.** WORKSPACE under Q-4, each with **its own `workspace_id`**, because there is no parent to inherit from. Unique keys become `(workspace_id, term)` and `(workspace_id, product_key)`. The query paths `findByOwnerIdAndTerm`, `findAllByOwnerId` and `findByOwnerIdAndProductKey` move to workspace equivalents in R3.

**Consequence to know.** In a Household, one member moving an item's aisle or skipping a suggestion changes it for every member. That is what a shared list means (Q-4). If per-person behaviour were ever wanted, these two tables would become USER with workspace context, as coach conversations are. That would be a new Product Owner decision.

### 9.2 `event_transactions`: same-workspace integrity

**Today.**
- Schema (V1): `PRIMARY KEY (event_id, transaction_id)`, a plain FK to `events(id)` and a plain FK to `transaction(id)`. No `owner_id` column, no `ON DELETE CASCADE`, and nothing at the database level relating the two owners.
- The same-owner rule is enforced **only in the application**. `EventService.assignTransaction` and `removeTransaction` call `requireOwned` on both the event and the transaction; `assignTransactions` keeps only transactions owned by the caller. `DefaultAuthorizationService.requireOwned` is the strict check (the caller's own rows only, no Household widening), so no legacy household path could have created a cross-owner link through this service.
- `EventLinksEraser` deletes the links by the transaction's owner, before either side goes (`ErasureStep.EVENT_LINKS`).

**Why inheritance cannot satisfy Q-9 here.** A join table has two parents. Inheriting "through events" says nothing about the transaction side, and a plain FK on each side cannot compare the two parents' workspaces. The only declarative guarantee is a shared column.

**Required enforcement (R2-7):**
1. `event_transactions.workspace_id NOT NULL`, set by the application on insert and backfilled from `events.workspace_id`. It is an integrity column, not a second authorization input.
2. `UNIQUE (id, workspace_id)` on `events` and on `transaction`, as targets for composite FKs (the same pattern ADR-001 15.3 uses for every child).
3. `FOREIGN KEY (event_id, workspace_id) REFERENCES events (id, workspace_id)` and `FOREIGN KEY (transaction_id, workspace_id) REFERENCES transaction (id, workspace_id)`. Both FKs use the **same** `workspace_id` column, so PostgreSQL rejects any row whose event and transaction are in different workspaces, on insert and on any later update.
4. Before step 3, a read-only count of existing rows where `events.owner_id <> transaction.owner_id` (R0-13). Expected zero, given the application check. If it is not zero, those rows follow the Q-8 rule: no automatic repair, and a reviewed and auditable plan first.
5. `EventService` keeps its application check (NO-RLS: the database guarantees integrity, not access). R3 moves it from `requireOwned` to the workspace check.

The delete behaviour of the two FKs stays as today unless R2 decides otherwise; it does not affect the same-workspace guarantee.

### 9.3 The inherited WORKSPACE tables and their database guarantee

At `e17a4ed` four WORKSPACE tables inherited ownership. After 9.2, three remain.

| Table | Parent FK today | Other references to tenant rows | Guarantee |
|---|---|---|---|
| `transaction_tag` (V1) | `transaction_id NOT NULL` → `transaction(id)` `ON DELETE CASCADE`, part of the PK `(transaction_id, tag)` | none: `tag` is free text | Holds |
| `recurring_series_previous_identity` (V128) | `recurring_series_id NOT NULL` → `recurring_series(id)` `ON DELETE CASCADE`, part of the PK `(recurring_series_id, position)` | none: `description_group` is text | Holds |
| `receipt_photo` (V137) | `receipt_id NOT NULL` → `receipt(id)` `ON DELETE CASCADE`, part of the PK `(receipt_id, position)` | `storage_key` → `attachment_blob`, **no FK**; the blob table has no owner today | Holds while `attachment_blob` has no workspace (caveat below) |
| `event_transactions` (V1) | two parents | the other parent | **Does not hold without a column**: own `workspace_id` and composite FKs (9.2) |

**The exact guarantee for the three single-parent tables.** A cross-workspace association needs two references to tenant rows in different workspaces. Each of these rows has exactly one:
1. The parent FK is `NOT NULL` and part of the primary key, so every row has exactly one parent, and that parent exists.
2. After R2-6 the parent carries `workspace_id NOT NULL`, so the row's workspace is its parent's, by construction.
3. No other column references a tenant-owned row, so there is no second endpoint that could sit in another workspace.
4. `ON DELETE CASCADE` removes the row with its parent: no orphan outlives its owner.
5. Workspace moves cannot split them: resources never move between workspaces (D-23), and the composite FKs of R2-7 on the parent stop a parent's `workspace_id` from changing while children reference it.

**Keeping it true (R2-11).** The guarantee rests on condition 3, which a future migration could break, for example by adding a `category_id` to `transaction_tag`. R2-11 must include a schema test: every WORKSPACE table without its own `workspace_id` has exactly one FK to a tenant table, to its parent, which is `NOT NULL`, in the primary key and `ON DELETE CASCADE`, and no other `*_id` column. A table that fails must take its own `workspace_id` and composite FKs, as `event_transactions` does.

**Caveat, `receipt_photo` and blobs.** This inventory (4.5) records `attachment_blob` as having no owner, while ADR-001 15.3 says it "gains `workspace_id` (O1)". If R2 adopts O1, `storage_key` becomes a second tenant reference. Then either `receipt_photo` takes its own `workspace_id` with a composite reference to the blob, or the blob's workspace is checked through the owning row as 4.5 describes. That choice belongs to R2 design (open item OI-1 in ADR-001 28.9) and does not change the classification.

### 9.4 Retired ledger tables: no active behaviour, no retention dependency

**Active application behaviour: none.** fern-funds #720 (`6361abf3`, 2026-11-08, "remove the internal double-entry ledger nothing read") deleted the `accounting` package: its entities and repositories, the projection service that wrote the tables on every transaction write, the admin reset and rebuild endpoints, and the Settings Advanced tab. What still names the tables at `c6447fc5`:

| Reference | What it does | Behaviour? |
|---|---|---|
| `user/RetiredLedgerEraser` (`ErasureStep.RETIRED_LEDGER`) | Deletes a deleted user's rows from the four tables | Deletion only; it reads nothing and writes nothing new |
| `AccountDeletionCoverageTest` | Lists the tables so deletion coverage stays complete | Test |
| `OwnerScopingBoundaryTest` | Exempts the entity name `Posting`, which no longer exists | Test (stale name, harmless) |
| `trg_stamp_owner_id` on `journal_entry`, `ledger_node`, `ledger_node_mapping` (V34, V35) | Stamps `owner_id` on insert or update | Fires only on a write, and no write path exists |
| `DisplayMode.LEDGER`, a comment in `frontend/src/lib/types.ts` | A stored display preference value | Not connected to the tables |

No live table has a foreign key into them. Their only FKs are internal (`posting` to `journal_entry` and `ledger_node`, `ledger_node_mapping` to `ledger_node`, `ledger_node` to itself). They are not FQL query tables, not reached by MCP tools, not exported and not read by admin metrics.

**Retention dependency: none found in the code or documents.** The tables held a projection of transactions; ADR U31 (a) itself described keeping them as "rebuilt from records". The source records are the transactions, which are kept and migrated. No feature, report, export or balance read the projection, and no retention rule names it. General legal retention of financial data and backups is still open as U12; it governs the source records and the backups, not this derived copy. The daily `pg_dump` backups keep their own lifecycle.

**Still required before the cleanup change (not done here, by instruction):**
- the read-only production row counts of the four tables (R0-13);
- a written data-retention assessment that confirms the above against those counts;
- then one dedicated change that drops the four tables and their triggers, removes `RetiredLedgerEraser` and its `ErasureStep`, and updates the two tests.

No production access, drop or code change was made by this task.

**Stale document found.** fern-funds `docs/ENGINEERING_DEBT.md` still lists DEBT-011 ("decide whether the internal ledger is worth keeping") as "Not started", although #720 removed the ledger and Q-6 now decides U31. It is outside this repository and was not changed.

## 10. Addendum: schema added by R1 (V180 to V184, 2026-11-17)

The inventory above was taken at fern-funds `c6447fc5` (V1 to V179). R1 has since merged and deployed V180 to V184 (fern-funds #801, #802, #803, #804; production at `471c7a79`). This addendum covers exactly that delta, read from the migrations on `master` and the code that uses them. Sections 3 to 9 are unchanged: no earlier classification moves.

### 10.1 New tables

| Table | Module | Ownership | FKs and relationships | Proposed class | Own `workspace_id` | Sens | Code | Notes and risks |
|---|---|---|---|---|---|---|---|---|
| workspace | workspace | none: it **is** the tenant. Owned by its one ACTIVE OWNER membership; there is deliberately no owner column (V180) | target of `workspace_membership (workspace_id, type)`, `api_token.workspace_id`, `oauth_grant.workspace_id` (both CASCADE) | **Tenant root** (see Q-10) | No: its `id` is the tenant key | P (name), F-adjacent settings | `Workspace`, `WorkspaceRepository`, `PersonalWorkspaces`, `MemberWorkspaces`, `WorkspaceEraser` (JDBC) | Holds the Q-7 money settings (`base_currency`, `budget_totals_per_period`, `merchant_auto_categorize_enabled`, V182). Type immutable by trigger. Exactly one ACTIVE OWNER enforced by a deferred constraint trigger. Every WORKSPACE table of R2 will reference it |
| workspace_membership | workspace | `user_id` FK (RESTRICT) and `workspace_id` | composite FK `(workspace_id, workspace_type)` to `workspace (id, type)`; `user_id` to users | **Tenant root** (see Q-10) | It has one: it is the join between a person and a workspace | P | `WorkspaceMembership`, its repository, `ActiveMemberships`, `MembershipExits`, `WorkspaceEraser` (JDBC) | Partial unique indexes: one ACTIVE OWNER per workspace, one member per PERSONAL, one PERSONAL per user, one Household slot (INVITED or ACTIVE) per user (V183). This is the only authorization gate for the workspace context (R1-9, R1-11, R1-12) |

Neither table carries `owner_id`, so a search for `owner_id` misses both (section 5).

### 10.2 Changed tables

| Table | Change | Classification effect |
|---|---|---|
| api_token | V184: `workspace_id` FK to `workspace` (CASCADE), backfilled to the PERSONAL workspace; `CHECK (session_token = (workspace_id IS NULL))`; index | None. Stays USER (4.2); the column is the D-14 **scope** the inventory anticipated, now built |
| oauth_grant | V184: `workspace_id NOT NULL` FK to `workspace` (CASCADE), backfilled to PERSONAL; index | None. Stays USER (4.2), same reason |
| users | No column change. V182 **copies** the three Q-7 settings to `workspace`; the `users` columns remain and are kept equal through `UserMoneySettingsChanged` in the same transaction | None. Removing the `users` copies is later work, not R0-8 |

### 10.3 Counts after the addendum

| Class | Tables |
|---|---|
| WORKSPACE | 54 (unchanged) |
| USER | 29 (unchanged) |
| REFERENCE | 15 (unchanged) |
| Tenant root, pending Q-10 | 2 (`workspace`, `workspace_membership`) |
| **Total** | **100** (excluding `flyway_schema_history`) |

### 10.4 Question for the Product Owner

| # | Question | Recommendation |
|---|---|---|
| Q-10 | How should `workspace` and `workspace_membership` be classified? They fit none of the three classes: they are not data that belongs to a workspace, nor to one person, nor global reference data | Record them as a fourth, structural class "tenant root" that is never migrated by R2 and never gains a `workspace_id` of its own. No code depends on the answer |

### 10.5 Effect on R1 and R2

- R2 composite foreign keys (`(workspace_id, ...)` per Q-9 and 9.2) target `workspace (id)`. `uk_workspace_id_type` exists if a WORKSPACE table ever needs the type as well.
- Production on 2026-11-17 (read-only, aggregates only): 8 users, 8 PERSONAL workspaces, 8 memberships, 0 Households, every API token and OAuth grant bound to a workspace the owner is an ACTIVE member of. Every invariant check was zero.
- Account deletion: `workspace_membership.user_id` is RESTRICT, so `WorkspaceEraser` must stay the last erasure step (R1-14). Any R2 table referencing `workspace` with RESTRICT would have to be erased before it.

## 11. Change log

| Date | Change | Evidence |
|---|---|---|
| 2026-11-16 | Inventory created: 98 tables classified from V1 to V179 on PostgreSQL 16; 12 AMBIGUOUS tables and questions Q-1 to Q-9 raised for the Product Owner | `e17a4ed` |
| 2026-11-17 | Product Owner decisions Q-1 to Q-9 recorded (6.1). The 12 AMBIGUOUS tables classified: 7 WORKSPACE, 5 USER. Clarifications answered (9): the grocery learned-state tables have no parent and take their own `workspace_id`; `event_transactions` takes an integrity `workspace_id` with two composite FKs; the guarantee for the three single-parent tables is written down, with a schema test for R2-11; the retired ledger has no active behaviour or retention dependency. Counts are now 54 WORKSPACE (47 with their own column), 29 USER, 15 REFERENCE. The submitted summary is kept in 3.1. Status VERIFIED | This commit |
| 2026-11-17 | Addendum (section 10) for the R1 schema, V180 to V184: `workspace` and `workspace_membership` added as tenant root (Q-10 raised); `api_token` and `oauth_grant` scope columns recorded. Total 100 tables. No earlier classification changed. Documentation only | This commit |
