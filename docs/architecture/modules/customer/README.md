# Customer Module

**Package:** `@medusajs/customer`  
**Module key:** `Modules.CUSTOMER`  
**Version:** Medusa v2.15.4

---

## Overview

The Customer module manages all customer-facing identity and segmentation data in a Medusa storefront. It provides first-class support for registered customer accounts, guest flows, B2B company contexts, address books, and customer groups used for promotions, pricing, and access control. Authentication credential management (password hashing, token issuance) is explicitly **delegated to the Auth module**; the Customer module only stores the resulting profile data.

The module is fully self-contained and exposes a typed service interface (`ICustomerModuleService`) that other modules and workflows consume via Medusa's dependency-injection container.

---

## Key Entities

| Entity | DB table | ID prefix | Description |
|---|---|---|---|
| `Customer` | `customer` | `cus_` | Core customer profile — name, email, phone, B2B company name |
| `CustomerGroup` | `customer_group` | `cusgroup_` | Named segment used for pricing rules and promotions |
| `CustomerGroupCustomer` | `customer_group_customer` | — | Pivot table linking customers to groups (many-to-many) |
| `CustomerAddress` | `customer_address` | `cuaddr_` | Physical address stored in the customer's address book |

### Customer Fields
`id`, `company_name` (nullable, B2B), `first_name`, `last_name`, `email` (searchable, unique when `has_account=true`), `phone`, `has_account` (boolean), `metadata` (JSON), `created_by`, `groups` (M2M), `addresses` (1:N).

### CustomerGroup Fields
`id`, `name` (unique, translatable), `metadata`, `created_by`, `customers` (M2M).

### CustomerAddress Fields
`id`, `address_name`, `is_default_shipping`, `is_default_billing`, `company`, `first_name`, `last_name`, `address_1`, `address_2`, `city`, `country_code`, `province`, `postal_code`, `phone`, `metadata`, `customer` (belongs-to).

---

## Key Service Methods

```ts
// Customer CRUD
createCustomers(data: CreateCustomerDTO | CreateCustomerDTO[]): Promise<CustomerDTO | CustomerDTO[]>
updateCustomers(id | ids | selector, data: CustomerUpdatableFields): Promise<CustomerDTO | CustomerDTO[]>
listCustomers(filters?, config?): Promise<CustomerDTO[]>
retrieveCustomer(id, config?): Promise<CustomerDTO>
deleteCustomers(ids: string[]): Promise<void>

// Group management
createCustomerGroups(data): Promise<CustomerGroupDTO | CustomerGroupDTO[]>
updateCustomerGroups(id | ids | selector, data): Promise<CustomerGroupDTO | CustomerGroupDTO[]>
addCustomerToGroup(pair: GroupCustomerPair | GroupCustomerPair[]): Promise<{ id: string } | { id: string }[]>
removeCustomerFromGroup(pair): Promise<void>

// Address management
createCustomerAddresses(data): Promise<CustomerAddressDTO | CustomerAddressDTO[]>
updateCustomerAddresses(id | ids | selector, data): Promise<CustomerAddressDTO | CustomerAddressDTO[]>
deleteCustomerAddresses(ids: string[]): Promise<void>
```

---

## API Endpoints

### Admin API
| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/customers` | List customers with filters/pagination |
| `POST` | `/admin/customers` | Create a customer |
| `GET` | `/admin/customers/:id` | Retrieve a customer |
| `POST` | `/admin/customers/:id` | Update a customer |
| `DELETE` | `/admin/customers/:id` | Delete a customer |
| `GET` | `/admin/customer-groups` | List customer groups |
| `POST` | `/admin/customer-groups` | Create a customer group |
| `POST` | `/admin/customer-groups/:id/customers` | Add customers to group |
| `DELETE` | `/admin/customer-groups/:id/customers` | Remove customers from group |

### Store API
| Method | Path | Description |
|---|---|---|
| `POST` | `/store/customers` | Register a new customer |
| `GET` | `/store/customers/me` | Retrieve authenticated customer's profile |
| `POST` | `/store/customers/me` | Update authenticated customer's profile |
| `GET` | `/store/customers/me/addresses` | List customer's addresses |
| `POST` | `/store/customers/me/addresses` | Create a new address |
| `POST` | `/store/customers/me/addresses/:id` | Update an address |
| `DELETE` | `/store/customers/me/addresses/:id` | Delete an address |

---

## Module Links (Remote Relationships)

The Customer module exposes its entities to the **Module Link** layer so that other modules can establish remote foreign keys without coupling:

- **Order module** → `customer_id` on an order references `Customer.id`
- **Cart module** → `customer_id` on a cart references `Customer.id`
- **Pricing module** → `CustomerGroup.id` used as a `PriceRule` attribute value
- **Promotion module** → `CustomerGroup.id` used as a condition on promotions

---

## Configuration

```ts
// medusa-config.ts
import { Modules } from "@medusajs/framework/utils"

export default defineConfig({
  modules: [
    { resolve: "@medusajs/customer", key: Modules.CUSTOMER }
  ]
})
```

---

## Events Emitted

| Event constant | Payload | Trigger |
|---|---|---|
| `customer.created` | `{ id }` | After `createCustomers` |
| `customer.updated` | `{ id }` | After `updateCustomers` |
| `customer.deleted` | `{ id }` | After `deleteCustomers` |
| `customer_group.created` | `{ id }` | After `createCustomerGroups` |
| `customer_group.deleted` | `{ id }` | After `deleteCustomerGroups` |

---

## Related Modules

- **Auth** — issues tokens for customer sessions; the Customer module stores the resulting profile
- **Order** — orders reference `customer_id`
- **Cart** — carts reference `customer_id`
- **Pricing** — price rules can target `customer_group_id`
- **Promotion** — promotions can be scoped to customer groups
