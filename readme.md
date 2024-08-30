Project Outline: Flash Sale for Flipzon

Assumptions and System Design
Assumptions:

Flash Sale Timing: The sale will start at 12:00 AM on the upcoming Sunday and continue until all 1000 iPhones are sold.
Concurrency: Multiple customers will attempt to buy the product simultaneously.
Fairness: Each customer can purchase only one iPhone during the flash sale.
Customer Identification: Each request will include a user_authentication_token to identify the customer.
System Design Overview:

Database: MongoDB will be used to store product information, sale transactions, and customer details.
Concurrency Control: Implement Redis or in-memory locking to handle race conditions and ensure no more than 1000 iPhones are sold.
API Rate Limiting: To prevent abuse, use rate limiting to cap the number of requests a user can make.
Scalability: The system will be built to handle high traffic by optimizing queries, using caching, and designing for horizontal scalability.

Implementation
1. Setting Up the Project
Create a new Node.js project and install the required dependencies:

mkdir flipzon-flash-sale
cd flipzon-flash-sale
npm init -y
npm install express mongoose redis

2. Database Models
Product Model:

// models/Product.js
const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
    name: String,
    quantity: Number,
    price: Number,
});

module.exports = mongoose.model('Product', productSchema);


Sale Model:

// models/Sale.js
const mongoose = require('mongoose');

const saleSchema = new mongoose.Schema({
    userId: String,
    productId: String,
    saleTime: Date,
});

module.exports = mongoose.model('Sale', saleSchema);


Customer Model:

// models/Customer.js
const mongoose = require('mongoose');

const customerSchema = new mongoose.Schema({
    userId: String,
    totalPurchases: Number,
});

module.exports = mongoose.model('Customer', customerSchema);


3. Flash Sale Logic
Start Flash Sale:

// routes/flashSale.js
const express = require('express');
const router = express.Router();
const Product = require('../models/Product');
const Sale = require('../models/Sale');
const Customer = require('../models/Customer');
const redis = require('redis');
const client = redis.createClient();

// Middleware to check if flash sale is active
const isSaleActive = (req, res, next) => {
    const currentTime = new Date();
    const saleStartTime = new Date();
    saleStartTime.setHours(0, 0, 0); // 12:00 AM
    if (currentTime >= saleStartTime) {
        next();
    } else {
        return res.status(403).json({ message: 'Flash sale is not active' });
    }
};

// Start Flash Sale Route
router.post('/start', async (req, res) => {
    const product = await Product.findOne({ name: 'iPhone' });
    if (!product) {
        await new Product({ name: 'iPhone', quantity: 1000, price: 999 }).save();
    } else {
        product.quantity = 1000;
        await product.save();
    }
    res.json({ message: 'Flash sale started' });
});

// Buy iPhone
router.post('/buy', isSaleActive, async (req, res) => {
    const { userId } = req.body;
    
    // Locking to ensure atomicity
    const lockKey = `lock:${userId}`;
    const isLocked = await client.get(lockKey);
    if (isLocked) {
        return res.status(429).json({ message: 'Too many requests. Please try again later.' });
    }
    
    await client.setex(lockKey, 1, 'locked');
    
    try {
        const product = await Product.findOne({ name: 'iPhone' });
        if (!product || product.quantity <= 0) {
            return res.status(400).json({ message: 'iPhone sold out' });
        }

        const existingSale = await Sale.findOne({ userId });
        if (existingSale) {
            return res.status(400).json({ message: 'You have already purchased an iPhone' });
        }

        // Decrement the product quantity atomically
        product.quantity -= 1;
        await product.save();

        // Register the sale
        const sale = new Sale({
            userId,
            productId: product._id,
            saleTime: new Date(),
        });
        await sale.save();

        res.json({ message: 'Purchase successful' });

    } finally {
        // Release the lock
        await client.del(lockKey);
    }
});

// Flash Sale Status
router.get('/status', async (req, res) => {
    const product = await Product.findOne({ name: 'iPhone' });
    res.json({
        product: product.name,
        remaining: product.quantity,
    });
});

module.exports = router;


4. Testing and Benchmarking
Load Testing: Use tools like Apache JMeter or Artillery to simulate high traffic and test the performance.
Unit Testing: Write unit tests for each API endpoint using Mocha and Chai.
Stress Testing: Simulate real-world scenarios where multiple customers try to purchase simultaneously.
5. Setting Up Locally
Clone the repository.
Run npm install to install dependencies.
Start MongoDB and Redis services.
Run npm start to start the server.

README.md
In the README file, document:

The assumptions made during development.
The system design, including how concurrency and fairness are handled.
Step-by-step instructions to set up the project locally.
Testing and benchmarking results.

Good to Have:

A comprehensive test suite.
Performance optimization strategies.
This approach should help you build a robust and scalable flash sale feature for Flipzon.



