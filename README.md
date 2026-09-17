# 🛒 rwp-shop

**rwp-shop** is a full-featured e-commerce plugin built for the **react-wp** CMS framework[cite: 1, 2]. It provides complete online store capabilities—including product catalog browsing, dynamic cart operations, secure checkout flows, customer accounts, and customizable shop templates[cite: 1].

It seamlessly integrates with **rwp-page-builder**, allowing administrators to visually design custom layouts for product pages, store archives, cart containers, and checkout screens using drag-and-drop widgets and dynamic shortcodes[cite: 1, 2].

---

## 🚀 Key Features

* **Complete E-Commerce Storefront**: Ready-to-use views for Store Catalog, Single Product Display, Category/Tag Archives, Cart, Checkout, and Customer Account Management[cite: 1].
* **Modular Shortcode Engine**: Embed product grids, buy buttons, and live cart counters anywhere on standard pages or blog posts using simple bracket tags (e.g., `[rwp_products]`, `[rwp_add_to_cart]`)[cite: 2].
* **`rwp-page-builder` Visual Integration**: Full layout override system allowing visual drag-and-drop editing for all shop template slots (`shop`, `product`, `product_category`, `cart`, `checkout`, `my_account`)[cite: 1].
* **Hybrid Fallback Strategy**: Renders visually designed JSON layouts via `BuilderRenderer` when custom templates exist, or seamlessly falls back to optimized default React views (`ShopPage`, `ProductPage`, etc.)[cite: 1].
* **Secure Backend Business Logic**: All stock verifications, price calculations, cart logic, and order placement (`shop_calculate`, `shop_place_order`) are handled securely via Supabase PostgreSQL policies and RPC handlers[cite: 1].
* **Order & Customer Dashboard**: Customer history dashboard (`[rwp_order_history]`) allowing users to track purchase statuses and review previous orders[cite: 2].

---

## 🛠️ Tech Stack & Dependencies

| Area | Component |
| :--- | :--- |
| **Frontend UI** | React, TypeScript, Tailwind CSS[cite: 1] |
| **Icons** | Lucide-React[cite: 1] |
| **Backend & Database** | Supabase PostgreSQL (Auth, RLS, RPCs)[cite: 1] |
| **CMS Core** | `react-wp` Plugin Registry Engine[cite: 1, 3] |
| **Visual Editing** | `rwp-page-builder` & `BuilderRenderer`[cite: 1] |

---

## 📂 Plugin Directory Structure

```text
plugins/rwp-shop/
├── index.tsx                  # Plugin registration & route entry point
├── pages/
│   ├── ShopPage.tsx           # Default store catalog view
│   ├── ProductPage.tsx        # Default single product template
│   ├── CartPage.tsx           # Default shopping cart layout
│   ├── CheckoutPage.tsx       # Default checkout & payment gateway form
│   └── AccountPage.tsx        # Customer account & order history screen
├── components/
│   ├── ProductCard.tsx        # Catalog product grid item
│   ├── CartDrawer.tsx        # Slide-over mini-cart
│   └── OrderSummary.tsx      # Cart & checkout price calculation display
├── shortcodes/
│   ├── AddToCartShortcode.tsx # [rwp_add_to_cart] handler
│   ├── ProductsGridShortcode.tsx # [rwp_products] handler
│   ├── CartLinkShortcode.tsx  # [rwp_cart_link] handler
│   └── OrderHistoryShortcode.tsx # [rwp_order_history] handler
├── widgets/                   # Dynamic widgets for rwp-page-builder
│   ├── ProductTitleWidget.tsx
│   ├── ProductPriceWidget.tsx
│   ├── ProductGalleryWidget.tsx
│   ├── AddToCartWidget.tsx
│   ├── CartTableWidget.tsx
│   └── CheckoutFormWidget.tsx
└── utils/
    └── cartStore.ts           # Client-side cart state management
```[cite: 1, 2]

---

## 🧷 Available Shortcodes

You can paste these shortcodes anywhere in standard content pages, posts, or page builder text blocks[cite: 2]:

### 1. `[rwp_products]`
Displays a dynamic grid of store products with query filtering options[cite: 2].
* **Attributes**:
  * `limit`: Number of products to show (default: `"4"`)[cite: 2].
  * `category`: Filter by category slug (e.g., `category="shirts"`)[cite: 2].
  * `featured`: Set to `"1"` to display only featured items[cite: 2].
  * `on_sale`: Set to `"1"` to filter discounted items[cite: 2].
  * `orderby`: Sort order (`menu_order`, `popularity`, `date`, `price`, `price-desc`, `title`)[cite: 2].
  * `ids`: Comma-separated list of product IDs (e.g., `ids="10,12,15"`)[cite: 2].

### 2. `[rwp_add_to_cart]`
Renders a standalone "Add to Cart" button for a single product[cite: 2].
* **Attributes**:
  * `id`: Target product ID (Required)[cite: 2].
  * `label`: Custom button text (default: `"Add to cart"`)[cite: 2].
  * `quantity`: Initial quantity to add (default: `"1"`)[cite: 2].

### 3. `[rwp_cart_link]`
Outputs a dynamic header link showing the current item count in the user's cart[cite: 2].

### 4. `[rwp_checkout]`
Renders the interactive payment and shipping checkout form on dedicated checkout pages[cite: 2].

### 5. `[rwp_order_history]`
Prints a list of past orders, payment statuses, and tracking links for the logged-in customer[cite: 2].

---

## 🗄️ Database Setup (Supabase SQL)

Run the following SQL migration script in your Supabase SQL Editor to provision the required e-commerce tables and extend the `pages` table for shop templates[cite: 1]:

```sql
-- 1. Extend pages table to support shop templates and visual builder data
ALTER TABLE pages 
ADD COLUMN IF NOT EXISTS is_shop_page BOOLEAN DEFAULT false,
ADD COLUMN IF NOT EXISTS shop_page_type VARCHAR(50) DEFAULT NULL, -- 'shop', 'product', 'product_category', 'cart', 'checkout', 'my_account'
ADD COLUMN IF NOT EXISTS builder_data JSONB DEFAULT '{}'::jsonb,
ADD COLUMN IF NOT EXISTS is_builder_enabled BOOLEAN DEFAULT false;

-- 2. Create products table
CREATE TABLE IF NOT EXISTS products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  sale_price DECIMAL(10,2),
  sku TEXT UNIQUE,
  stock_quantity INT DEFAULT 0,
  is_featured BOOLEAN DEFAULT false,
  images JSONB DEFAULT '[]'::jsonb,
  category_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Create orders table
CREATE TABLE IF NOT EXISTS orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'processing', 'completed', 'cancelled'
  total_amount DECIMAL(10,2) NOT NULL,
  shipping_address JSONB NOT NULL,
  billing_address JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. Create order items junction table
CREATE TABLE IF NOT EXISTS order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE SET NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL
);
```[cite: 1]

---

## ⚡ Routing & Dynamic Layout Override

When a user accesses an e-commerce route, `rwp-shop` verifies whether a custom visual layout was generated with `rwp-page-builder` before falling back to the default component[cite: 1]:

```tsx
import React from 'react';
import { BuilderRenderer } from '../rwp-page-builder/renderers/BuilderRenderer';
import { ShopPage } from './pages/ShopPage';

export const ShopRouteResolver = ({ pageData }) => {
  // 1. If visual builder is active and JSON layout exists, render visual layout
  if (pageData?.is_builder_enabled && pageData?.builder_data) {
    return <BuilderRenderer data="{pageData.builder_data}"/>;
  }

  // 2. Fall back to standard hardcoded React component
  return <ShopPage/>;
};
```[cite: 1]

---

## 📄 License
Distributed under the MIT License. Part of the **react-wp** CMS ecosystem[cite: 1, 2].
