# Car taxi management System

## Project Structure
Backend (Node.js and Express)
Frontend (React)
Database (MongoDB)
Authentication (JWT)
API Endpoints (CRUD operations)

## Step-by-Step Implementation
## 1. Backend Setup
### Install dependencies
mkdir car-taxi-management
cd car-taxi-management
npm init -y
npm install express mongoose bcryptjs jsonwebtoken
npm install --save-dev nodemon


### server.js
const express = require('express');
const mongoose = require('mongoose');
const app = express();

app.use(express.json());

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/taxi-management', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const db = mongoose.connection;
db.on('error', console.error.bind(console, 'connection error:'));
db.once('open', () => {
  console.log('Connected to MongoDB');
});

// Routes
const userRoutes = require('./routes/user');
const bookingRoutes = require('./routes/booking');

app.use('/api/users', userRoutes);
app.use('/api/bookings', bookingRoutes);

// Start server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});


### Models

#### User.js

const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  isDriver: { type: Boolean, default: false },
});

UserSchema.pre('save', async function (next) {
  if (!this.isModified('password')) {
    return next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

UserSchema.methods.matchPassword = async function (enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);


#### Booking.js

const mongoose = require('mongoose');

const BookingSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  driver: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  pickupLocation: { type: String, required: true },
  dropoffLocation: { type: String, required: true },
  date: { type: Date, required: true },
  status: { type: String, default: 'pending' },
});

module.exports = mongoose.model('Booking', BookingSchema);

### Routes

#### user.js

const express = require('express');
const User = require('../models/User');
const jwt = require('jsonwebtoken');

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  const { username, password, isDriver } = req.body;
  try {
    const user = new User({ username, password, isDriver });
    await user.save();
    res.status(201).json({ message: 'User registered successfully' });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  const { username, password } = req.body;
  try {
    const user = await User.findOne({ username });
    if (!user || !(await user.matchPassword(password))) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user._id }, 'your_jwt_secret', { expiresIn: '1h' });
    res.json({ token });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;


#### booking.js

const express = require('express');
const Booking = require('../models/Booking');

const router = express.Router();

// Create a booking
router.post('/', async (req, res) => {
  const { user, driver, pickupLocation, dropoffLocation, date } = req.body;
  try {
    const booking = new Booking({ user, driver, pickupLocation, dropoffLocation, date });
    await booking.save();
    res.status(201).json({ message: 'Booking created successfully' });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get bookings
router.get('/', async (req, res) => {
  try {
    const bookings = await Booking.find().populate('user').populate('driver');
    res.json(bookings);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;

## 2. Frontend Setup
## Install dependencies

npx create-react-app client
cd client
npm install axios react-router-dom


### App.js

import React from 'react';
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';
import Home from './components/Home';
import Login from './components/Login';
import Register from './components/Register';
import Bookings from './components/Bookings';

function App() {
  return (
    <Router>
      <div className="App">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/login" element={<Login />} />
          <Route path="/register" element={<Register />} />
          <Route path="/bookings" element={<Bookings />} />
        </Routes>
      </div>
    </Router>
  );
}

export default App;


### Components

#### Home.js

import React from 'react';

function Home() {
  return <h3>Welcome to Car Taxi Management System</h3>;
}

export default Home;

#### Login.js

import React, { useState } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';

function Login() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const res = await axios.post('/api/users/login', { username, password });
      localStorage.setItem('token', res.data.token);
      navigate('/bookings');
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Login</h2>
      <input
        type="text"
        placeholder="Username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit">Login</button>
    </form>
  );
}

export default Login;

#### Register.js

import React, { useState } from 'react';
import axios from 'axios';

function Register() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [isDriver, setIsDriver] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post('/api/users/register', { username, password, isDriver });
      alert('User registered successfully');
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Register</h2>
      <input
        type="text"
        placeholder="Username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <label>
        <input
          type="checkbox"
          checked={isDriver}
          onChange={() => setIsDriver(!isDriver)}
        />
        Register as Driver
      </label>
      <button type="submit">Register</button>
    </form>
  );
}

export default Register;

#### Bookings.js

import React, { useState, useEffect } from 'react';
import axios from '
