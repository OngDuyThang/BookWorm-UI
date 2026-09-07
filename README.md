# BookWorm UI — Storefront Web Application

[![Next.js](https://img.shields.io/badge/Next.js-13.5-000000?logo=next.js&logoColor=white)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-18.0-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Apollo Client](https://img.shields.io/badge/Apollo_Client-3.10-311C87?logo=apollographql&logoColor=white)](https://www.apollographql.com/)
[![Redux Toolkit](https://img.shields.io/badge/Redux_Toolkit-2.2-764ABC?logo=redux&logoColor=white)](https://redux-toolkit.js.org/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-3.0-06B6D4?logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)
[![Ant Design](https://img.shields.io/badge/Ant_Design-5.17-0170FE?logo=antdesign&logoColor=white)](https://ant.design/)
[![Stripe](https://img.shields.io/badge/Stripe-Elements-008CDD?logo=stripe&logoColor=white)](https://stripe.com/)
[![GraphQL Codegen](https://img.shields.io/badge/GraphQL_Codegen-5.0-CB3837?logo=graphql&logoColor=white)](https://the-guild.dev/graphql/codegen)

> A modern, responsive, and type-safe e-commerce customer storefront for the **BookWorm** distributed bookstore platform. Built with **Next.js (Pages Router)** and **TypeScript**, this application interfaces with a distributed **NestJS microservices backend** ([BookWorm-API](https://github.com/OngDuyThang/BookWorm-API)). It features dynamic multi-service GraphQL routing via Apollo Client, distributed OAuth2/2FA authentication with silent token refresh, real-time cart synchronization, faceted book catalog search, customer reviews, Stripe Elements checkout, and dynamic CMS page rendering.

---

## Table of Contents

- [Architectural Overview & System Context](#architectural-overview--system-context)
- [System Topology & Data Flow](#system-topology--data-flow)
- [Key Engineering Patterns](#key-engineering-patterns)
  - [1. Dynamic Multi-Service Apollo Client Gateway](#1-dynamic-multi-service-apollo-client-gateway)
  - [2. Distributed Authentication & Token Binding (OAuth2 + 2FA)](#2-distributed-authentication--token-binding-oauth2--2fa)
  - [3. Resilient Silent Token Refresh Interceptor](#3-resilient-silent-token-refresh-interceptor)
  - [4. Global State & Persistent Store (Redux Toolkit)](#4-global-state--persistent-store-redux-toolkit)
  - [5. Stripe Elements & Order Placement Pipeline](#5-stripe-elements--order-placement-pipeline)
- [Application Modules & Page Catalog](#application-modules--page-catalog)
  - [Home (`/home` & `/`)](#home-home---redirect-from-)
  - [Shop Catalog (`/shop`)](#shop-catalog-shop)
  - [Product Detail & Reviews (`/product/[slug]`)](#product-detail--reviews-productslug)
  - [Cart & Checkout Modal (`/cart`)](#cart--checkout-modal-cart)
  - [Order Confirmation (`/order-complete`)](#order-confirmation-order-complete)
  - [Dynamic About Us (`/about`)](#dynamic-about-us-about)
- [GraphQL Code Generation](#graphql-code-generation)
- [Workspace Directory Structure](#workspace-directory-structure)
- [Environment Configuration](#environment-configuration)
- [Getting Started & Local Development](#getting-started--local-development)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Development Scripts](#development-scripts)
- [Related Repositories](#related-repositories)

---

## Architectural Overview & System Context

**BookWorm UI** functions as the customer-facing storefront web application within the larger BookWorm e-commerce ecosystem. The backend is partitioned into independent NestJS microservices communicating over RabbitMQ RPC, each guarding its own PostgreSQL database.

Instead of funneling all requests through a monolithic API Gateway, this frontend coordinates requests directly to the responsible microservices using a hybrid client approach:
- **Apollo Client (GraphQL)**: Dispatches typed GraphQL queries and mutations dynamically to the **Product**, **Cart**, and **Order** microservices.
- **Axios (HTTP REST)**: Interacts with the **Auth Service** (for OAuth2 token exchanges, refresh, logout) and the **Asset Service** (for dynamic CMS content).

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                CLIENT TIER                                       │
│                                                                                  │
│                      BookWorm-UI (Next.js / Port 8080)                           │
│     ┌────────────────────────┐                  ┌────────────────────────┐       │
│     │  Apollo Client Gateway │                  │      Axios Client      │       │
│     │   (Dynamic Link GQL)   │                  │      (HTTP REST)       │       │
│     └───────────┬────────────┘                  └───────────┬────────────┘       │
└─────────────────┼───────────────────────────────────────────┼────────────────────┘
                  │                                           │
                  ▼                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       DISTRIBUTED MICROSERVICES CLUSTER                          │
│                                                                                  │
│   ┌───────────────────────────┐               ┌──────────────────────────────┐   │
│   │   Product Service (:3001) │               │      Auth Service (:3000)    │   │
│   │   - Books & Detail        │               │      - OAuth2 Auth Code Flow │   │
│   │   - Categories & Authors  │               │      - 2FA TOTP Validation   │   │
│   │   - Promotions & Reviews  │               │      - Silent Token Refresh  │   │
│   └─────────────┬─────────────┘               └──────────────┬───────────────┘   │
│                 │                                            │                   │
│   ┌─────────────┴─────────────┐               ┌──────────────┴───────────────┐   │
│   │    Cart Service (:3002)   │               │     Asset Service (:3005)    │   │
│   │    - Real-time user cart  │               │     - Dynamic CMS Pages      │   │
│   │    - Item quantity sync   │               │       (About Us)             │   │
│   └─────────────┬─────────────┘               └──────────────────────────────┘   │
│                 │                                                                │
│   ┌─────────────┴─────────────┐               ┌──────────────────────────────┐   │
│   │    Order Service (:3003)  │               │     Stripe Payment Gateway   │   │
│   │    - Checkout lifecycle   │──────────────►│     - PaymentElement         │   │
│   │    - Stripe PaymentIntent │               │     - Confirm Payment        │   │
│   └───────────────────────────┘               └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## System Topology & Data Flow

```
[ Customer Browser ]         [ Next.js Frontend ]        [ Microservice Backend ]         [ Third Parties ]
         │                          │                               │                            │
         │  1. Navigate to Store    │                               │                            │
         ├─────────────────────────►│                               │                            │
         │                          │  2. GQL: Recommend / OnSale   │                            │
         │                          ├──────────────────────────────►│ (Product Service: 3001)    │
         │                          │◄──────────────────────────────┤                            │
         │  3. Render Home/Catalog  │                               │                            │
         │◄─────────────────────────┤                               │                            │
         │                          │                               │                            │
         │  4. Click "Login"        │                               │                            │
         ├──────────────────────────┴──────────────────────────────►│ (Auth Service: 3000)       │
         │                                                          │ - Credentials + 2FA TOTP   │
         │  5. Redirect to /callback?code=<authCode>                │                            │
         │◄─────────────────────────────────────────────────────────┤                            │
         │                                                          │                            │
         │  6. Exchange authCode via POST /auth/token               │                            │
         ├─────────────────────────►│                               │                            │
         │                          ├──────────────────────────────►│ (Auth Service: 3000)       │
         │                          │◄──────────────────────────────┤                            │
         │                          │   Returns: access_token       │                            │
         │                          │   Set-Cookie: refresh_token   │                            │
         │                          │   Set-Cookie: fingerprint     │                            │
         │  7. Redux Session Active │                               │                            │
         │◄─────────────────────────┤                               │                            │
         │                          │                               │                            │
         │  8. Place Order (Stripe) │                               │                            │
         ├─────────────────────────►│  9. GQL: placeOrder           │                            │
         │                          ├──────────────────────────────►│ (Order Service: 3003)      │
         │                          │                               ├───────────────────────────►│ (Stripe API)
         │                          │                               │◄───────────────────────────┤ Return clientSecret
         │                          │◄──────────────────────────────┤                            │
         │  10. Mount Stripe Form   │  Return clientSecret, orderId │                            │
         │◄─────────────────────────┤                               │                            │
         │                          │                               │                            │
         │  11. Submit Payment Form │                                                            │
         ├──────────────────────────┴───────────────────────────────────────────────────────────►│ (Stripe Gateway)
         │  12. Redirect: /order-complete?orderId=...                                            │
         │◄──────────────────────────────────────────────────────────────────────────────────────┤
         │                          │                                                            │
         │  13. Update Status       │ 14. GQL: updatePaymentStatus('PAID')                       │
         ├─────────────────────────►├──────────────────────────────►│ (Order Service: 3003)      │
```

---

## Key Engineering Patterns

### 1. Dynamic Multi-Service Apollo Client Gateway

In a distributed backend where independent microservices expose distinct GraphQL ports, a standard single-URI Apollo Client configuration is insufficient.

In `src/pages/_app.tsx`, a dynamic `createHttpLink` is implemented. It inspects the GraphQL operation's context (`operation.getContext().service`) to resolve the destination service at runtime:

```typescript
// src/pages/_app.tsx
const httpLink = createHttpLink({
    uri: ({ getContext }) => {
        const { service } = getContext();

        switch (service) {
            case SERVICE.CART:
                return getGqlEndpoint(SERVICE.CART);   // http://localhost:3002/graphql
            case SERVICE.ORDER:
                return getGqlEndpoint(SERVICE.ORDER);  // http://localhost:3003/graphql
            default:
                return getGqlEndpoint(SERVICE.PRODUCT);// http://localhost:3001/graphql
        }
    },
    credentials: 'include' // Attaches cryptographic fingerprint and refresh cookies
});
```

When executing queries or mutations, components simply specify their target service:
```typescript
const [getUserCart] = useLazyQuery(GET_USER_CART, {
    context: { service: SERVICE.CART }
});
```

### 2. Distributed Authentication & Token Binding (OAuth2 + 2FA)

The application adheres to an enterprise OAuth2-style Authorization Code and TOTP Two-Factor Authentication architecture designed to resist token theft and cross-site scripting (XSS):

1. **User Login**: The user clicks **Login** and is redirected to the Auth microservice (`:3000/auth/login`).
2. **Credential & 2FA Validation**: The Auth Service validates credentials and, if 2FA is enabled, checks the 6-digit TOTP code (`otplib`).
3. **Authorization Code Generation**: The Auth Service generates a short-lived `authCode` stored in Redis (TTL: 100s) and redirects the browser back to `http://localhost:8080/callback?code=<authCode>`.
4. **Token Exchange**: `src/pages/callback.tsx` extracts `code`, dispatches `POST /auth/token` via Axios, and receives:
   - `access_token`: Short-lived (15 minutes) JWT passed via JSON body.
   - `refresh_token`: Long-lived (7 days) JWT sent as an `HttpOnly` cookie.
   - `fingerprint`: Hashed UUID browser fingerprint sent as an `HttpOnly` cookie.
5. **Claims Extraction**: The client decodes user details (`username`, `email`, `picture`) using `jwt-decode` and commits the session to Redux Toolkit.

### 3. Resilient Silent Token Refresh Interceptor

To maintain seamless user experience without disruptive session expirations, both GraphQL (Apollo) and REST (Axios) pipelines incorporate silent refresh interceptors.

#### Apollo Client Refresh Interceptor (`src/pages/_app.tsx`)
The `errorLink` catches `401 Unauthorized` responses, validates token expiration, calls the Auth Service refresh endpoint (`/auth/refresh`), updates the in-memory token, and transparently retries the failed GraphQL query:

```typescript
const errorLink = onError(({ graphQLErrors, operation, forward }) => {
    const currentAccessToken = getAccessToken();
    if (graphQLErrors?.[0]?.statusCode === 401 && isSession() && currentAccessToken) {
        const payload = jwtDecode(currentAccessToken);
        if (Date.now() >= Number(payload?.exp) * 1000) {
            return fromPromise(
                getNewAccessToken(currentAccessToken).catch(() => autoLogout())
            ).flatMap((newAccessToken) => {
                if (!newAccessToken) {
                    autoLogout();
                    return forward(operation);
                }
                replaceAccessToken(newAccessToken);
                operation.setContext(({ headers = {} }) => ({
                    headers: { ...headers, authorization: `Bearer ${newAccessToken}` }
                }));
                return forward(operation); // Retry original operation
            });
        }
    }
});
```

#### Axios Refresh Interceptor (`src/api/axios.ts`)
A matching response interceptor handles REST calls, preventing infinite refresh loops by filtering out requests targeting `/auth/refresh` and invoking `autoLogout()` upon unrecoverable authorization failure.

### 4. Global State & Persistent Store (Redux Toolkit)

Global state is managed via **Redux Toolkit** and synced with `localStorage` through `redux-persist`:
- **User Slice (`src/store/user/slice.ts`)**: Stores access token, decoded username, email, avatar image, and active session boolean flag (`isSession`).
- **Cart Slice (`src/store/cart/slice.ts`)**: Maintains the live cart badge count shown in the header, keeping local UI badge count synchronized with backend cart operations.

### 5. Stripe Elements & Order Placement Pipeline

The checkout flow provides atomic order creation with integrated payment processing:
1. **Order Creation**: `src/modules/Cart/index.tsx` submits customer shipping info, contact details, and payment selection (`COD` or `STRIPE`) to the Order microservice via GraphQL `placeOrder(order)`.
2. **Payment Intent Resolution**:
   - If **COD**: The order is confirmed immediately, cart state is cleared, and the user is routed to `/order-complete`.
   - If **STRIPE**: The Order microservice creates a Stripe `PaymentIntent` and returns a `clientSecret` and `orderId`.
3. **Stripe Elements Form**: The UI switches to an embedded Stripe `<Elements>` modal containing the official Stripe `<PaymentElement>` (`src/components/CheckoutForm`).
4. **Return URL & Confirmation**: Upon confirmation, Stripe redirects to `/order-complete?orderId=<id>`, where `src/pages/order-complete/index.tsx` fires the `updatePaymentStatus` mutation to set the order status to `PAID`.

---

## Application Modules & Page Catalog

### Home (`/home` - redirect from `/`)
- **Promotions Carousel**: Interactive slider powered by `react-slick` showcasing current discounts and featured titles.
- **On-Sale Highlight**: Displays discounted books with dynamic discount badges and original/promotional pricing.
- **Featured Books Tab**: Interactive toggle between **Recommended Books** and **Popular Books** (aggregated by the backend from historical order volumes).

### Shop Catalog (`/shop`)
- **Faceted Category Tree**: Recursive tree navigation (`src/modules/Shop/tree.tsx`) representing parent and child categories.
- **Multi-Filter Sidebar**: Filter concurrently by multiple authors, nested categories, and minimum star ratings.
- **Sorting Options**: Sort by *On Sale*, *Price Low to High*, or *Price High to Low*.
- **Pagination & Page Size**: Dynamic pagination (5, 10, 20 items per page) synchronized with the URL query parameters using shallow routing (`useRouterProductQuery`).

### Product Detail & Reviews (`/product/[slug]`)
- **Book Details**: Cover image, author, parent/child category tags, description, and dynamic price calculations.
- **Quantity Selector & Add-to-Cart**: Add items directly to the user's cart; triggers real-time header cart badge updates.
- **Rating Breakdown**: Average star rating calculation, fractional star rendering, and review count distribution.
- **Paginated Customer Reviews**: Filter reviews by star count (1-5 stars) and sort by date ascending/descending.
- **Review Submission**: Authenticated customer review form with instant GraphQL mutation (`createReview`) and automatic refetching.

### Cart & Checkout Modal (`/cart`)
- **Cart Item Management**: Adjust line item quantities or delete products with automatic recalculation of subtotals and discounts.
- **Checkout Modal**: Ant Design form capturing customer name, phone, email, delivery address, and payment method choice (`COD` vs. `STRIPE`).
- **Conditional Stripe Rendering**: Mounts Stripe Elements dynamically when card payment is selected.

### Order Confirmation (`/order-complete`)
- Catches query parameters (`orderId`) from the payment return URL.
- Invokes `updatePaymentStatus` GraphQL mutation on the Order microservice to transition order state to `PAID`.
- Purges local cart state from Redux and displays an order success banner.

### Dynamic About Us (`/about`)
- Fetches store narrative directly from the **Asset Microservice** REST endpoint (`GET /api/assets/about-page`).
- Parses and renders sanitized CMS HTML dynamically using `html-react-parser`.

---

## GraphQL Code Generation

The repository uses `@graphql-codegen/cli` to generate fully-typed TypeScript types, operation definitions, and React Apollo hooks directly from the running microservices.

### Configuration (`codegen.ts`)
```typescript
import { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
    schema: [
        'http://localhost:3001/graphql', // Product Service
        'http://localhost:3002/graphql', // Cart Service
        'http://localhost:3003/graphql'  // Order Service
    ],
    documents: ['src/**/*.ts', 'src/**/*.tsx'],
    generates: {
        "./src/__generated__/": {
            preset: "client",
            presetConfig: {
                gqlTagName: "gql",
            },
        },
    },
};

export default config;
```

To regenerate types when backend schemas change, execute:
```bash
npm run generate
```

---

## Workspace Directory Structure

```
BookWorm-UI/
├── public/
│   └── images/                     # Static media assets & logos
├── src/
│   ├── __generated__/              # Auto-generated GraphQL types & documents
│   ├── api/                        # Axios HTTP client, interceptors & REST calls
│   │   ├── auth.ts                 # Auth service REST endpoints (logout)
│   │   └── axios.ts                # Axios instance with 401 silent refresh interceptor
│   ├── components/                 # Reusable UI component library (Button, Modal, Input, etc.)
│   │   ├── CheckoutForm/           # Stripe Elements payment integration
│   │   ├── Carousel/               # React Slick carousel wrapper
│   │   ├── Drawer/                 # Antd responsive drawer
│   │   └── ...                     # Skeletons, Spinners, Dropdowns, Typography
│   ├── graphql/                    # Typed GraphQL query and mutation definitions
│   │   ├── author/                 # GET_AUTHORS query
│   │   ├── cart/                   # GET_USER_CART, ADD_TO_CART, UPDATE, REMOVE
│   │   ├── category/               # GET_CATEGORIES query
│   │   ├── order/                  # PLACE_ORDER, UPDATE_PAYMENT_STATUS
│   │   ├── product/                # GET_PRODUCT_LIST, POPULAR, RECOMMEND, ON_SALE
│   │   ├── promotion/              # GET_ALL_PROMOTIONS query
│   │   └── review/                 # GET_PRODUCT_BY_ID, CREATE_REVIEW
│   ├── hooks/                      # Custom React hooks (debounce, redux, router queries)
│   ├── layout/                     # App shell, Navbar, Footer, Mobile Sider & SEO
│   │   ├── Header/                 # Navigation, search, cart count badge, user profile
│   │   ├── Footer/                 # Store footer info
│   │   └── Components/             # HeadSEO, User avatar dropdown, mobile trigger
│   ├── modules/                    # Feature domain modules
│   │   ├── About/                  # About page with dynamic CMS content
│   │   ├── Cart/                   # Cart table, total calculation & order modal
│   │   ├── Detail/                 # Product details, ratings & review list
│   │   ├── Home/                   # Hero promotions slider & featured tabs
│   │   └── Shop/                   # Catalog listing, category tree & faceted filters
│   ├── pages/                      # Next.js Pages Router
│   │   ├── _app.tsx                # App root: Apollo Provider, Redux, multi-link gateway
│   │   ├── 404.tsx                 # Custom 404 Not Found page
│   │   ├── callback.tsx            # OAuth2 redirect handler & token exchange
│   │   ├── about/                  # /about route
│   │   ├── cart/                   # /cart route
│   │   ├── home/                   # /home route
│   │   ├── product/[slug].tsx      # /product/[slug] route
│   │   ├── shop/                   # /shop route
│   │   └── order-complete/         # /order-complete route
│   ├── store/                      # Redux Toolkit store & Redux Persist setup
│   │   ├── cart/                   # Cart item count slice
│   │   ├── user/                   # User authentication state slice
│   │   └── index.ts                # Root store configuration
│   ├── styles/                     # SCSS modules & Tailwind global styling
│   ├── types/                      # Shared TypeScript domain models & DTOs
│   └── utils/                      # Helper utilities, constants & URL formatters
├── codegen.ts                      # GraphQL Code Generator configuration
├── next.config.js                  # Next.js configuration (redirects & S3 remote images)
├── package.json                    # Workspace dependencies and scripts
├── tailwind.config.ts              # Tailwind CSS configuration
└── tsconfig.json                   # TypeScript configuration & path aliases (@/*)
```

---

## Environment Configuration

Create a `.env.local` or `.env.development` file in the root directory. You can copy `.env.example` as a starting point:

```bash
cp .env.example .env.local
```

### Configuration Variables Reference

| Variable Name                         | Type    | Default Local Value | Description                                                               |
| :------------------------------------ | :------ | :------------------ | :------------------------------------------------------------------------ |
| `NEXT_PUBLIC_API_METHOD`              | String  | `http`              | Protocol for API communication (`http` or `https`)                        |
| `NEXT_PUBLIC_API_HOST`                | String  | `localhost`         | Hostname of the backend microservices                                     |
| `NEXT_PUBLIC_API_AUTH_PORT`           | Number  | `3000`              | Port of the Auth Microservice (OAuth2, 2FA, token refresh)                |
| `NEXT_PUBLIC_API_PRODUCT_PORT`        | Number  | `3001`              | Port of the Product Microservice (GraphQL catalog, categories, reviews)   |
| `NEXT_PUBLIC_API_CART_PORT`           | Number  | `3002`              | Port of the Cart Microservice (GraphQL cart operations)                   |
| `NEXT_PUBLIC_API_ORDER_PORT`          | Number  | `3003`              | Port of the Order Microservice (GraphQL checkout & Stripe order creation) |
| `NEXT_PUBLIC_API_UPLOAD_PORT`         | Number  | `3004`              | Port of the Upload Microservice (AWS S3 asset upload pipeline)            |
| `NEXT_PUBLIC_API_ASSET_PORT`          | Number  | `3005`              | Port of the Asset Microservice (REST dynamic CMS pages)                   |
| `NEXT_PUBLIC_HOST`                    | String  | `localhost`         | Frontend application hostname                                             |
| `NEXT_PUBLIC_PORT`                    | Number  | `8080`              | Frontend application port                                                 |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`  | String  | `pk_test_...`       | Public Stripe API key for rendering Stripe Elements                      |

---

## Getting Started & Local Development

### Prerequisites

- **Node.js**: `v18.x` or `v20.x` (LTS recommended)
- **npm**: `v9.x` or later
- **Backend Services**: The backend microservices cluster ([BookWorm-API](https://github.com/OngDuyThang/BookWorm-API)) running locally or accessible via network.

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/OngDuyThang/BookWorm-UI.git
   cd BookWorm-UI
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Configure environment variables:
   ```bash
   cp .env.example .env.local
   ```
   *(Ensure the microservice port values match your running backend instances).*

### Development Scripts

| Command             | Description                                                                     |
| :------------------ | :------------------------------------------------------------------------------ |
| `npm run dev`       | Starts the Next.js development server on port `8080` (`http://localhost:8080`). |
| `npm run build`     | Compiles and builds the production bundle on port `8080`.                       |
| `npm run start`     | Runs the compiled production build on port `8080`.                              |
| `npm run generate`  | Introspects backend GraphQL endpoints and generates typed operations in `src/`. |
| `npm run lint`      | Runs Next.js ESLint checks across all TypeScript and React files.               |

---

## Related Repositories

- **Backend Platform**: [BookWorm-API](https://github.com/OngDuyThang/BookWorm-API) — The distributed NestJS monorepo microservices platform (Auth, Product, Cart, Order, Upload, Asset, SSR Admin MVC Dashboard, RabbitMQ, PostgreSQL, Redis, Stripe).
