# MERN Mini E‑Commerce — Starter Template (Backend + React Frontend)

A minimal, working e‑commerce site you can run locally: product list, cart, mock checkout, and simple admin product management.

> Tech: **MongoDB, Express, Node.js, React (Vite)**

---

## 1) Project Structure

```
mern-ecommerce/
├─ server/
│  ├─ .env               # MONGO_URI=..., PORT=5000
│  ├─ package.json
│  ├─ server.js
│  ├─ models/
│  │  └─ Product.js
│  ├─ routes/
│  │  └─ products.js
│  └─ seed.js            # optional: seed sample products
└─ client/
   ├─ .env               # VITE_API_URL=http://localhost:5000
   ├─ package.json
   ├─ index.html
   └─ src/
      ├─ main.jsx
      ├─ App.jsx
      ├─ components/
      │  ├─ ProductCard.jsx
      │  └─ Cart.jsx
      └─ store/
         └─ cart.js
```

---

## 2) Backend (Node + Express + MongoDB)

### `server/package.json`

```json
{
  "name": "server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "node --watch server.js",
    "seed": "node seed.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "mongoose": "^8.5.0",
    "morgan": "^1.10.0"
  }
}
```

### `server/server.js`

```js
import express from 'express';
import mongoose from 'mongoose';
import cors from 'cors';
import morgan from 'morgan';
import dotenv from 'dotenv';
import productRouter from './routes/products.js';

dotenv.config();
const app = express();

app.use(cors());
app.use(express.json());
app.use(morgan('dev'));

app.get('/', (req, res) => res.json({ ok: true, message: 'API running' }));
app.use('/api/products', productRouter);

const PORT = process.env.PORT || 5000;
const MONGO_URI = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/mernecom';

mongoose
  .connect(MONGO_URI)
  .then(() => {
    console.log('MongoDB connected');
    app.listen(PORT, () => console.log('Server listening on', PORT));
  })
  .catch((err) => console.error('Mongo error', err));
```

### `server/models/Product.js`

```js
import mongoose from 'mongoose';

const ProductSchema = new mongoose.Schema(
  {
    title: { type: String, required: true },
    description: { type: String, default: '' },
    price: { type: Number, required: true, min: 0 },
    image: { type: String, default: 'https://via.placeholder.com/300x200' },
    stock: { type: Number, default: 100 },
    category: { type: String, default: 'general' }
  },
  { timestamps: true }
);

export default mongoose.model('Product', ProductSchema);
```

### `server/routes/products.js`

```js
import { Router } from 'express';
import Product from '../models/Product.js';

const router = Router();

// GET /api/products
router.get('/', async (req, res) => {
  const q = req.query.q?.trim();
  const filter = q ? { title: { $regex: q, $options: 'i' } } : {};
  const items = await Product.find(filter).sort({ createdAt: -1 });
  res.json(items);
});

// GET /api/products/:id
router.get('/:id', async (req, res) => {
  const item = await Product.findById(req.params.id);
  if (!item) return res.status(404).json({ message: 'Product not found' });
  res.json(item);
});

// POST /api/products  (simple admin route — no auth here for demo)
router.post('/', async (req, res) => {
  try {
    const created = await Product.create(req.body);
    res.status(201).json(created);
  } catch (e) {
    res.status(400).json({ message: e.message });
  }
});

export default router;
```

### `server/seed.js` (optional)

```js
import mongoose from 'mongoose';
import dotenv from 'dotenv';
import Product from './models/Product.js';

dotenv.config();
const MONGO_URI = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/mernecom';

const sample = [
  { title: 'Wireless Headphones', price: 1999, category: 'electronics', image: 'https://picsum.photos/seed/1/600/400', description: 'BT 5.3, 30h battery' },
  { title: 'Running Shoes', price: 2499, category: 'fashion', image: 'https://picsum.photos/seed/2/600/400', description: 'Lightweight, breathable' },
  { title: 'Smart Watch', price: 3499, category: 'electronics', image: 'https://picsum.photos/seed/3/600/400', description: 'Heart‑rate, SpO2' }
];

(async () => {
  await mongoose.connect(MONGO_URI);
  await Product.deleteMany({});
  await Product.insertMany(sample);
  console.log('Seeded products:', sample.length);
  await mongoose.disconnect();
  process.exit(0);
})();
```

---

## 3) Frontend (React + Vite)

### Quick create

```bash
# inside /client
npm create vite@latest client -- --template react
cd client
npm i
```

### `client/package.json` (scripts)

```json
{
  "name": "client",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  }
}
```

### `client/.env`

```
VITE_API_URL=http://localhost:5000
```

### `client/src/store/cart.js`

```js
import { createContext, useContext, useEffect, useReducer } from 'react';

const CartContext = createContext();

const reducer = (state, action) => {
  switch (action.type) {
    case 'add': {
      const exists = state.items.find(i => i._id === action.item._id);
      const items = exists
        ? state.items.map(i => i._id === action.item._id ? { ...i, qty: i.qty + 1 } : i)
        : [...state.items, { ...action.item, qty: 1 }];
      return { ...state, items };
    }
    case 'inc':
      return { ...state, items: state.items.map(i => i._id === action.id ? { ...i, qty: i.qty + 1 } : i) };
    case 'dec':
      return { ...state, items: state.items.map(i => i._id === action.id ? { ...i, qty: Math.max(1, i.qty - 1) } : i) };
    case 'remove':
      return { ...state, items: state.items.filter(i => i._id !== action.id) };
    case 'clear':
      return { items: [] };
    default:
      return state;
  }
};

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, { items: JSON.parse(localStorage.getItem('cart') || '[]') });
  useEffect(() => localStorage.setItem('cart', JSON.stringify(state.items)), [state.items]);
  const total = state.items.reduce((s, i) => s + i.price * i.qty, 0);
  return (
    <CartContext.Provider value={{ ...state, total, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}

export const useCart = () => useContext(CartContext);
```

### `client/src/components/ProductCard.jsx`

```jsx
export default function ProductCard({ p, onAdd }) {
  return (
    <div style={{border:'1px solid #eee', borderRadius:16, padding:16, display:'flex', flexDirection:'column', gap:8}}>
      <img src={p.image} alt={p.title} style={{width:'100%', height:180, objectFit:'cover', borderRadius:12}} />
      <h3 style={{margin:0}}>{p.title}</h3>
      <p style={{margin:0, color:'#666'}}>{p.description}</p>
      <strong>₹{p.price}</strong>
      <button onClick={() => onAdd(p)} style={{padding:'10px 12px', borderRadius:12, border:'1px solid #ddd', cursor:'pointer'}}>Add to Cart</button>
    </div>
  );
}
```

### `client/src/components/Cart.jsx`

```jsx
import { useCart } from '../store/cart';

export default function Cart() {
  const { items, total, dispatch } = useCart();
  if (!items.length) return <div>Your cart is empty.</div>;
  return (
    <div>
      {items.map(i => (
        <div key={i._id} style={{display:'flex', alignItems:'center', gap:12, padding:'8px 0', borderBottom:'1px solid #eee'}}>
          <img src={i.image} width={60} height={40} style={{objectFit:'cover', borderRadius:8}} />
          <div style={{flex:1}}>
            <div style={{fontWeight:600}}>{i.title}</div>
            <div>₹{i.price} × {i.qty}</div>
          </div>
          <div style={{display:'flex', gap:6}}>
            <button onClick={() => dispatch({type:'dec', id:i._id})}>-</button>
            <button onClick={() => dispatch({type:'inc', id:i._id})}>+</button>
            <button onClick={() => dispatch({type:'remove', id:i._id})}>Remove</button>
          </div>
        </div>
      ))}
      <h3>Total: ₹{total}</h3>
      <button onClick={() => { alert('Checkout successful (mock)!'); dispatch({type:'clear'}); }}>Checkout</button>
    </div>
  );
}
```

### `client/src/App.jsx`

```jsx
import { useEffect, useState } from 'react';
import ProductCard from './components/ProductCard';
import Cart from './components/Cart';
import { useCart } from './store/cart';

const API = import.meta.env.VITE_API_URL || 'http://localhost:5000';

export default function App() {
  const [products, setProducts] = useState([]);
  const [query, setQuery] = useState('');
  const { dispatch } = useCart();

  useEffect(() => { fetch(`${API}/api/products`).then(r => r.json()).then(setProducts); }, []);

  const filtered = products.filter(p => p.title.toLowerCase().includes(query.toLowerCase()));

  return (
    <div style={{maxWidth:1100, margin:'0 auto', padding:24}}>
      <header style={{display:'flex', justifyContent:'space-between', alignItems:'center', marginBottom:24}}>
        <h1 style={{margin:0}}>ShopEasy</h1>
        <input placeholder="Search products..." value={query} onChange={e=>setQuery(e.target.value)} style={{padding:10, borderRadius:10, border:'1px solid #ddd', width:260}} />
      </header>

      <div style={{display:'grid', gridTemplateColumns:'2fr 1fr', gap:24}}>
        <div style={{display:'grid', gridTemplateColumns:'repeat(auto-fill, minmax(220px, 1fr))', gap:16}}>
          {filtered.map(p => (
            <ProductCard key={p._id} p={p} onAdd={(item)=>dispatch({type:'add', item})} />
          ))}
        </div>
        <aside style={{border:'1px solid #eee', borderRadius:16, padding:16, height:'fit-content'}}>
          <h2>Cart</h2>
          <Cart />
        </aside>
      </div>
    </div>
  );
}
```

### `client/src/main.jsx`

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import { CartProvider } from './store/cart.js'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <CartProvider>
      <App />
    </CartProvider>
  </React.StrictMode>
)
```

### `client/index.html`

```html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ShopEasy</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## 4) How to Run (Local)

### Backend

```bash
cd server
npm i
# create .env
cat > .env << EOF
MONGO_URI=mongodb://127.0.0.1:27017/mernecom
PORT=5000
EOF
npm run seed    # optional: adds sample products
npm run dev
```

### Frontend

```bash
cd client
npm i
# create .env with API URL if needed
echo "VITE_API_URL=http://localhost:5000" > .env
npm run dev
# Vite will show a local URL (e.g., http://localhost:5173)
```

---

## 5) Minimal Admin (no auth – demo only)

Add a product via API quickly:

```bash
curl -X POST http://localhost:5000/api/products \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "Backpack",
    "price": 1299,
    "description": "Durable 20L daypack",
    "image": "https://picsum.photos/seed/pack/600/400",
    "stock": 50,
    "category": "accessories"
  }'
```

---

## 6) Next Steps (nice upgrades)

* Add **JWT Auth** (users, admins) and protected admin routes.
* Save orders & implement real checkout (Stripe/Razorpay).
* Pagination, sorting, categories & filters.
* Image uploads (Cloudinary/S3) and inventory stock checks.
* Unit tests (Jest) and API tests (Supertest).

> This starter is intentionally small so you can run it fast, then iterate.
# MERN-Mini-E-Commerce
A minimal, working e‑commerce site you can run locally: product list, cart, mock checkout, and simple admin product management.
