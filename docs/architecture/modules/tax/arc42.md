# Architecture Documentation — Tax Module (arc42)

## 1. Introduction and Goals

The Tax module is Medusa's extensible taxation engine. It models tax jurisdictions hierarchically (country → province), supports multiple overlapping rates per jurisdiction, enables product-specific rate targeting via rules, and abstracts the calculation engine behind a provider interface so merchants can plug in external compliance services.

**Quality Goals:**

| Priority | Quality Goal  | Description                                                                    |
|----------|---------------|--------------------------------------------------------------------------------|
| 1        | Correctness   | Tax lines must be calculated accurately for all applicable rate combinations   |
| 2        | Extensibility | External tax providers (TaxJar, Avalara) can be plugged in without core changes|
| 3        | Flexibility   | Support both gross (tax-included) and net (tax-excluded) pricing models        |
| 4        | Performance   | Tax calculation must not add significant latency to checkout                   |

## 2. Constraints

- Country codes must conform to ISO 3166-1 alpha-2.
- Province codes must conform to ISO 3166-2 format when used.
- A default tax rate must exist in each TaxRegion for the fallback calculation path.
- Tax providers must implement the `ITaxProvider` interface.

## 3. Context and Scope

```
External:
  [Admin Browser] ──CRUD──► [Admin API /admin/tax-regions, /admin/tax-rates]
  [TaxJar API]    ◄──calls── [TaxJar Provider Plugin]
  [Avalara API]   ◄──calls── [Avalara Provider Plugin]

Internal:
  [Tax Module] ──linked from──► [Region Module] (TaxRegion per country/region)
  [Cart Module] ──calls getTaxLines()──► [Tax Module]
  [Product Module] ──referenced by TaxRateRule──► [Tax Module]
```

## 4. Solution Strategy

| Challenge                           | Strategy                                                          |
|-------------------------------------|-------------------------------------------------------------------|
| Varying rates per jurisdiction      | Hierarchical TaxRegion (country + optional province)             |
| Reduced/zero rates for some goods   | TaxRateRule targets product/product_type/product_tag             |
| External compliance requirements    | ITaxProvider abstraction; delegate to TaxJar/Avalara when needed |
| Included vs excluded tax            | Cart module passes `is_tax_inclusive` flag in context; provider handles |

## 5. Building Block View

```
Tax Module
├── HTTP Layer
│   ├── Admin Routes: /admin/tax-regions (CRUD)
│   └── Admin Routes: /admin/tax-rates (CRUD)
│
├── Workflow Layer
│   ├── createTaxRegionsWorkflow
│   ├── updateTaxRegionsWorkflow
│   ├── deleteTaxRegionsWorkflow
│   ├── createTaxRatesWorkflow
│   └── deleteTaxRatesWorkflow
│
├── Service Layer
│   └── TaxModuleService
│       ├── createTaxRegions / deleteTaxRegions
│       ├── createTaxRates / updateTaxRates / deleteTaxRates
│       ├── createTaxRateRules / deleteTaxRateRules
│       └── getTaxLines()  ← primary runtime method
│
├── Provider Abstraction
│   └── TaxProviderService (resolves and delegates to registered ITaxProvider)
│
└── Domain Model
    ├── TaxRegion
    ├── TaxRate
    ├── TaxRateRule
    └── TaxProvider (reference entity)
```

## 6. Runtime View

**Scenario A: Checkout tax calculation**

```
Cart reaches tax calculation step
  → getTaxLines(lineItems, shippingMethods, { region, customer, address })
  │
  ├─ Resolve TaxRegion for address.country_code
  ├─ If address.province: resolve child TaxRegion for province
  ├─ For each line item:
  │   ├─ Check TaxRateRules: product_id match → product_type_id → product_tag_id
  │   └─ No rule match → use default TaxRate for region
  ├─ If TaxRegion.provider_id set:
  │   └─ Delegate to ITaxProvider.getTaxLines() (external API call)
  └─ Else: compute tax_amount = unit_price × (rate / 100)
  → Return ItemTaxLineDTO[] to Cart module
```

**Scenario B: Admin sets up German VAT with food exemption**

```
POST /admin/tax-regions { country_code: "DE" }              → TaxRegion created
POST /admin/tax-rates { tax_region_id, rate: 19, is_default: true }  → Standard rate
POST /admin/tax-rates { tax_region_id, rate: 7, name: "Reduced" }    → Reduced rate
POST /admin/tax-rates/:reduced_id/rules { reference: "product_type", reference_id: "ptyp_food" }
  → Food products now taxed at 7% instead of 19%
```

## 7. Deployment View

Single Medusa process. Tax providers are instantiated as part of the module's provider registry at startup. External tax API calls are made at checkout time (synchronous within the tax calculation step).

## 8. Cross-Cutting Concerns

| Concern         | Approach                                                               |
|-----------------|------------------------------------------------------------------------|
| Authentication  | All admin routes require JWT authentication                            |
| Transactions    | Write operations wrapped in `@InjectTransactionManager()`             |
| Provider timeout| External provider calls should be wrapped with timeout (provider responsibility) |
| Combinable rates| `is_combinable` flag on TaxRate controls additive vs standalone rates  |

## 9. Design Decisions

| ID  | Decision                             | Rationale                                                                   |
|-----|--------------------------------------|-----------------------------------------------------------------------------|
| D1  | Hierarchical TaxRegion (parent/child)| Province-level rules supplement country rules; avoids flat duplication      |
| D2  | TaxRateRule `reference` polymorphism | Single rule table handles product, product_type, product_tag targeting      |
| D3  | Provider abstraction                 | Decouples tax engine from module internals; enables 3rd-party compliance    |
| D4  | `is_default` on TaxRate              | Ensures unmatched items always get a rate without requiring catch-all rules |

## 10. Risks and Technical Debt

| Risk                                     | Mitigation                                               |
|------------------------------------------|----------------------------------------------------------|
| External provider latency at checkout    | Timeout + fallback to built-in calculation               |
| Province codes not standardised by users | Validation against ISO 3166-2 list at create time        |
| Multiple default rates in one region     | Unique constraint enforced at service level              |
