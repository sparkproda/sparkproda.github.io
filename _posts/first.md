---
layout: post
title:  "Integrating razorpay into your webapp"
date:   2019-03-23 21:03:36 +0530
categories: Javascript NodeJS
---
안녕하세요

```javascript
const Razorpay = require('razorpay');

let rzp = Razorpay({
	key_id: 'KEY_ID',
	secret: 'name'
});

// capture request
rzp.capture(payment_id, cost)
	.then(function (data) {
		return 2;
	})
```
