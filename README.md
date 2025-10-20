{
  "name": "poultry-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "seed": "node seed.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "mongoose": "^7.0.0"
  }
}
PORT=5000
MONGO_URI=mongodb+srv://<user>:<pass>@cluster0.mongodb.net/poultrydb?retryWrites=true&w=majority
JWT_SECRET=your_jwt_secret
PAYMENT_KEY=your_payment_key_here
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const productsRoute = require('./routes/products');
const usersRoute = require('./routes/users');
const ordersRoute = require('./routes/orders');

const app = express();
app.use(cors());
app.use(express.json());

app.use('/api/products', productsRoute);
app.use('/api/users', usersRoute);
app.use('/api/orders', ordersRoute);

const PORT = process.env.PORT || 5000;
mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(()=> {
    console.log('MongoDB connected');
    app.listen(PORT, ()=> console.log(`Server running on ${PORT}`));
  })
  .catch(err => console.error(err));
  const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  category: String, // e.g., 'live', 'processed', 'eggs'
  weight: String,
  price: Number,
  stock: Number, // number of birds or units
  images: [String],
  options: Object, // e.g., {age: '6 months'}
  reserved: { type: Number, default: 0 }
}, { timestamps: true });

module.exports = mongoose.model('Product', productSchema);
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  passwordHash: String,
  isAdmin: { type: Boolean, default: false }
}, { timestamps: true });

module.exports = mongoose.model('User', userSchema);
const mongoose = require('mongoose');

const orderSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  items: [{
    product: { type: mongoose.Schema.Types.ObjectId, ref: 'Product' },
    title: String, price: Number, quantity: Number, options: Object
  }],
  total: Number,
  deliverySlot: { date: Date, slot: String },
  status: { type: String, default: 'Pending' }, // Pending, Confirmed, Dispatched, Delivered
  payment: { method: String, paid: Boolean, gatewayResponse: Object }
}, { timestamps: true });

module.exports = mongoose.model('Order', orderSchema);
const express = require('express');
const router = express.Router();
const Product = require('../models/Product');

// list & filter
router.get('/', async (req, res) => {
  const { category, q } = req.query;
  const filter = {};
  if (category) filter.category = category;
  if (q) filter.title = { $regex: q, $options: 'i' };
  const products = await Product.find(filter).limit(100);
  res.json(products);
});

// get single
router.get('/:id', async (req, res) => {
  const p = await Product.findById(req.params.id);
  if(!p) return res.status(404).json({msg:'Not found'});
  res.json(p);
});

// admin create/update/delete - for brevity not secured here (add auth in production)
router.post('/', async (req,res) => {
  const newP = new Product(req.body);
  await newP.save();
  res.json(newP);
});

router.put('/:id', async (req,res) => {
  const p = await Product.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(p);
});

router.delete('/:id', async (req,res) => {
  await Product.findByIdAndDelete(req.params.id);
  res.json({ ok:true });
});

module.exports = router;
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// signup
router.post('/signup', async (req,res) => {
  const { name, email, password } = req.body;
  const existing = await User.findOne({ email });
  if(existing) return res.status(400).json({msg:'Email exists'});
  const hash = await bcrypt.hash(password, 10);
  const user = new User({ name, email, passwordHash: hash });
  await user.save();
  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
  res.json({ token, user: { id: user._id, email: user.email, name: user.name }});
});

// login
router.post('/login', async (req,res) => {
  const { email, password } = req.body;
  const u = await User.findOne({ email });
  if(!u) return res.status(400).json({msg:'Invalid'});
  const ok = await bcrypt.compare(password, u.passwordHash);
  if(!ok) return res.status(400).json({msg:'Invalid'});
  const token = jwt.sign({ id: u._id }, process.env.JWT_SECRET);
  res.json({ token, user: { id: u._id, email: u.email, name: u.name }});
});

module.exports = router;
const express = require('express');
const Order = require('../models/Order');
const Product = require('../models/Product');
const router = express.Router();

// create order (checkout)
router.post('/', async (req,res) => {
  const { userId, items, deliverySlot, payment } = req.body;

  // basic stock reservation check
  for (const it of items) {
    const p = await Product.findById(it.product);
    if (!p) return res.status(400).json({msg:'Product not found'});
    if (p.stock < it.quantity) return res.status(400).json({msg:`Insufficient stock for ${p.title}`});
    // reduce stock on payment later; here we'll decrement to reserve
    p.stock -= it.quantity;
    await p.save();
  }

  const total = items.reduce((s,i)=> s + i.price*i.quantity, 0);
  const order = new Order({ user: userId, items, total, deliverySlot, payment });
  await order.save();
  res.json(order);
});

// admin: list orders
router.get('/', async (req,res) => {
  const orders = await Order.find().populate('user').sort({createdAt:-1});
  res.json(orders);
});

module.exports = router;
const mongoose = require('mongoose');
const Product = require('./models/Product');
require('dotenv').config();

const items = [
  {
    title: 'Live Country Hen (Desi) — ~1.8kg',
    description: 'Free-range desi hen, 6–8 months.',
    category: 'live',
    weight: '1.8-2.0 kg',
    price: 800,
    stock: 30,
    images: []
  },
  {
    title: 'Fresh Whole Chicken — 1.2–1.5 kg',
    description: 'Humanely processed, chilled.',
    category: 'processed',
    weight: '1.2-1.5 kg',
    price: 220,
    stock: 100,
    images: []
  }
];

mongoose.connect(process.env.MONGO_URI).then(async ()=>{
  await Product.deleteMany({});
  await Product.insertMany(items);
  console.log('Seeded');
  process.exit();
});
{
  "name": "poultry-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.11.2",
    "react-scripts": "5.0.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}
import React from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './styles.css';

createRoot(document.getElementById('root')).render(
  <BrowserRouter><App/></BrowserRouter>
);
import axios from 'axios';
const API = axios.create({ baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000/api' });

export default API;
import React from 'react';
import { Routes, Route } from 'react-router-dom';
import Header from './components/Header';
import Home from './components/Home';
import ProductPage from './components/ProductPage';
import Cart from './components/Cart';
import Checkout from './components/Checkout';

export default function App(){
  return (
    <div>
      <Header />
      <main style={{padding:'20px'}}>
        <Routes>
          <Route path="/" element={<Home/>} />
          <Route path="/product/:id" element={<ProductPage/>} />
          <Route path="/cart" element={<Cart/>} />
          <Route path="/checkout" element={<Checkout/>} />
        </Routes>
      </main>
    </div>
  );
}
body { font-family: Arial, Helvetica, sans-serif; background:#f7f7f7; color:#222; margin:0;}
.header { background:#fff; padding:12px 20px; display:flex; align-items:center; justify-content:space-between; box-shadow:0 1px 3px rgba(0,0,0,0.06);}
.container { max-width:1100px; margin:0 auto; }
.card { background:#fff; padding:12px; border-radius:8px; box-shadow:0 1px 4px rgba(0,0,0,0.05); }
.grid { display:grid; grid-template-columns: repeat(auto-fill,minmax(220px,1fr)); gap:16px; }
.btn { background: #2b7a0b; color:white; padding:8px 12px; border-radius:6px; text-decoration:none; display:inline-block; }
.small { font-size:0.9rem; color:#555; }
import React from 'react';
import { Link } from 'react-router-dom';

export default function Header(){
  return (
    <header className="header">
      <div className="container" style={{display:'flex',alignItems:'center',justifyContent:'space-between'}}>
        <Link to="/" style={{textDecoration:'none', color:'#222'}}><h2>FarmFresh Poultry</h2></Link>
        <div>
          <Link to="/cart" className="btn" style={{background:'#444'}}>Cart</Link>
        </div>
      </div>
    </header>
  );
}
import React, {useEffect, useState} from 'react';
import API from '../api';
import { Link } from 'react-router-dom';

export default function Home(){
  const [products, setProducts] = useState([]);
  useEffect(()=> {
    API.get('/products').then(r=> setProducts(r.data)).catch(console.error);
  }, []);
  return (
    <div className="container">
      <section style={{margin:'20px 0'}}>
        <div className="card"><h3>Welcome — Order Live Hens & Fresh Chicken</h3><p className="small">Reserve live birds, choose delivery slots, and pay securely.</p></div>
      </section>

      <section>
        <h3>Products</h3>
        <div className="grid">
          {products.map(p=> (
            <div key={p._id} className="card">
              <h4>{p.title}</h4>
              <p className="small">{p.category} • ₹{p.price} • stock: {p.stock}</p>
              <p className="small">{p.weight}</p>
              <Link to={`/product/${p._id}`} className="btn">View</Link>
            </div>
          ))}
        </div>
      </section>
    </div>
  );
}
import React, {useEffect, useState} from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import API from '../api';

export default function ProductPage(){
  const { id } = useParams();
  const [product, setProduct] = useState(null);
  const [qty, setQty] = useState(1);
  const navigate = useNavigate();

  useEffect(()=> {
    API.get('/products/'+id).then(r=> setProduct(r.data)).catch(console.error);
  }, [id]);

  if(!product) return <div className="container">Loading...</div>;

  function addToCart(){
    const cart = JSON.parse(localStorage.getItem('cart')||'[]');
    cart.push({ productId: product._id, title: product.title, price: product.price, quantity: qty });
    localStorage.setItem('cart', JSON.stringify(cart));
    navigate('/cart');
  }

  return (
    <div className="container">
      <div className="card">
        <h2>{product.title}</h2>
        <p className="small">{product.description}</p>
        <p className="small">Price: ₹{product.price} • Stock: {product.stock}</p>
        <div style={{marginTop:10}}>
          <label>Quantity: </label>
          <input type="number" min="1" max={product.stock} value={qty} onChange={e=> setQty(Number(e.target.value))} />
        </div>
        <div style={{marginTop:10}}>
          <button className="btn" onClick={addToCart}>Add to cart</button>
          {' '}
          {product.category === 'live' && <span style={{marginLeft:12}} className="small">Tip: Reserve live birds with a 25% deposit at checkout.</span>}
        </div>
      </div>
    </div>
  );
}
import React from 'react';
import { Link, useNavigate } from 'react-router-dom';

export default function Cart(){
  const navigate = useNavigate();
  const cart = JSON.parse(localStorage.getItem('cart')||'[]');

  const total = cart.reduce((s,i)=> s + i.price*i.quantity, 0);

  function goCheckout(){
    navigate('/checkout');
  }

  return (
    <div className="container">
      <h3>Cart</h3>
      <div className="card">
        {cart.length===0 ? <p>No items</p> : cart.map((it,i)=> (
          <div key={i} style={{borderBottom:'1px solid #eee', padding:'8px 0'}}>
            <strong>{it.title}</strong><div className="small">Qty: {it.quantity} • ₹{it.price}</div>
          </div>
        ))}
        <div style={{marginTop:10}}>Total: ₹{total}</div>
        <div style={{marginTop:10}}>
          <button className="btn" onClick={goCheckout} disabled={cart.length===0}>Proceed to Checkout</button>
          <Link to="/" style={{marginLeft:10}}>Continue shopping</Link>
        </div>
      </div>
    </div>
  );
}
