# How to secure your API with Node and Express

APIs (Application Programming Interfaces) have become essential components for modern application development. However, as APIs become more crucial, they also become more susceptible to security threats. Protecting your API is crucial to ensuring the integrity of your data and the privacy of your users. In this article, we will explore how you can secure your API using Node.js and Express.

* [Use HTTPS](#use-https)
* [Validate input data](#validate-input-data)
* [Authentication and Authorization](#authentication-and-authorization)
* [Limit requests](#limit-requests)
* [Implement CORS](#implement-cors)
* [Use Helmet](#use-helmet)
* [Monitor and Record Activities](#monitor-and-record-activities)
* [Keep dependencies up to date](#keep-dependencies-up-to-date)

### Use https

The first step in protecting your API is to ensure that communications between the client and server are encrypted. Using HTTPS (Hypertext Transfer Protocol Secure) is essential to prevent data from being intercepted by malicious third parties. You can obtain an SSL/TLS certificate from a trusted certificate authority to enable HTTPS on your Node.js server.


```javascript
const https = require('https');
const fs = require('fs');
const express = require('express');

const app = express();
const options = {
  key: fs.readFileSync('ruta/a/clave-privada.key'),
  cert: fs.readFileSync('ruta/a/certificado.crt')
};

const server = https.createServer(options, app);

server.listen(3000, () => {
  console.log('Listening...');
});
```

### Validate input data

Input data validation is essential to prevent attacks such as [SQL injection](https://es.wikipedia.org/wiki/Inyecci%C3%B3n_SQL "SQL injection") and [cross-site scripting (XSS)](https://es.wikipedia.org/wiki/Cross-site_scripting "cross site scripting"). Use libraries such as express-validator to validate and sanitize input data before processing it in your API.

```javascript
const { body, validationResult } = require('express-validator');
const express = require('express');
const app = express();

app.post('/singup', [
  body('email').isEmail(),
  body('password').isStrongPassword(),
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errores: errors.array() });
  }
  // Process data...
});
```

### Authentication and Authorization

Implement a robust authentication system to ensure that only authorized users can access your API. You can use authentication strategies such as JWT (JSON Web Tokens) or OAuth.

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const app = express();

// Authentication middleware
const verificarToken = (req, res, next) => {
  const token = req.header('Authorization');

  if (!token) {
    return res.status(401).json({ mensaje: 'Access denied.' });
  }

  try {
    const decodificado = jwt.verify(token, 'secret');
    req.usuario = decodificado.usuario;
    next();
  } catch (error) {
    res.status(401).json({ mensaje: 'Invalid token.' });
  }
};

app.get('/protected-data', verificarToken, (req, res) => {
  // Access allowed for authenticated users only
});
```
### Limit requests

To prevent [denial-of-service (DoS)](https://es.wikipedia.org/wiki/Ataque_de_denegaci%C3%B3n_de_servicio "DoS attack") attacks, you can implement limits on the requests your API can handle in a given time interval using libraries such as express-rate-limit.

#### Limit by IP

```javascript
const express = require('express');
const rateLimit = require('express-rate-limit');
const app = express();

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // maximum 100 requests per IP in the interval,
  standardHeaders: true,
});

app.use(limiter);
```

#### Limit by usuario

```javascript
const express = require('express');
const rateLimit = require('express-rate-limit');
const app = express();

// Authentication middleware
// ...

// Rate limiting middleware
const userRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Maximum requests per time window
  keyGenerator: (req) => req.user.id, // Key generator based on the user ID
  handler: (req, res) => {
    res.status(429).json({ error: 'Speed limit reached. Try again later.' });
  },
});

app.use(userRateLimiter);

app.get('/protected', [verificarToken, userRateLimiter], (req, res) => {
  res.json({ message: 'Acceso concedido a la ruta protegida' });
});


app.listen(3000, () => {
  console.log('listen on port 3000');
});
```

#### Limit by header size

This serves to protect your application against [buffer overflow attacks](https://es.wikipedia.org/wiki/Desbordamiento_de_b%C3%BAfer "buffer overflow attack").

```javascript
const express = require('express');
const app = express();
app.use(express.json());

const limitPayloadSize = (req, res, next) => {
  const MAX_PAYLOAD_SIZE = 1024 * 1024; // 1MB
  if (req.headers['content-length'] && parseInt(req.headers['content-length']) > MAX_PAYLOAD_SIZE){
    return res.status(413).json({ error: 'Payload size exceeds the limit' });
  }
  next();
}

app.use(limitPayloadSize);

app.listen(3000, () => {
  console.log('listen on port 3000')
});
```

It can also be done using body-parser or a reverse proxy.

```javascript
const express = require('express');
const bodyParser = require('body-parser');
const app = express();

app.use(bodyParser.json({ limit: '1mb' }));

app.listen(3000, () => {
  console.log('listen on port 3000');
});
```

### Implement [CORS](https://developer.mozilla.org/es/docs/Web/HTTP/CORS "cors")

By default, browsers enforce the [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy "Same-Origin Policy"), which prevents a script on one website from accessing resources on another domain. But if your API allows requests from different domains (CORS), carefully configure the CORS options to prevent unauthorized sites from accessing your API.

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

const opcionesCors = {
  origin: ['https://domain1.com', 'https://domain2.com'],
  methods: 'GET,PUT,POST,DELETE',
};

app.use(cors(opcionesCors));
```

### Use [Helmet](https://helmetjs.github.io/ "Helmet")

* csp sets the Content-Security-Policy header to prevent cross-site scripting attacks and other cross-site injections.
* hidePoweredBy removes the X-Powered-By header.
* hsts sets the Strict-Transport-Security header, which forces secure connections (HTTP over SSL/TLS) with the server.
* ieNoOpen sets ***X-Download-Options*** for IE8+.
* noCache sets ***Cache-Control*** and ***Pragma*** headers to disable client-side caching.
* noSniff sets ***X-Content-Type-Options*** to prevent browsers from sniffing a response of the declared content type using MIME.
* frameguard sets the ***X-Frame-Options*** header to provide protection against clickhacking.
* xssFilter sets ***X-XSS-Protection*** to enable cross-site scripting (XSS) filtering in the latest web browsers.


```javascript
const express = require('express');
const helmet = require('helmet');

const app = express(); // This will set 13 HTTP response headers in your app.

app.use(helmet());

app.listen(3000, () => {
  console.log('Server is running on port 3000');
});
```
If you do not want to use Helmet, disable the X-Powered-By header. Attackers can use this header (which is enabled by default) to detect applications running Express and launch targeted attacks.


```javascript
app.disable('x-powered-by');
```

### Monitor and Log Activities

Logging and monitoring are incredibly important for consistent security in Node.js. Monitoring your logs gives you insight into what is happening in your application so you can investigate anything suspicious.

```javascript
const express = require("express");
const winston = require("winston");
const app = express();

const logger = winston.createLogger({
  level: "debug",
  format: winston.format.json(),
  transports: [new winston.transports.Console()],
});

app.get("/", (req, res, next) => {
  logger.debug("Home '/' route.");
  res.status(200).send("Logging Hello World..");
});

app.get("/data", (req, res, next) => {
  try {
    throw new Error("Not found!");
  } catch (error) {
    logger.error("Data Error: Not found");
    res.status(404).send("Error!");
  }
});

app.listen(3000, () => {
  logger.info("Server Listening On Port 3000");
});
```

### Keep dependencies up to date

Security vulnerabilities can arise from outdated dependencies. Use tools such as npm audit to check your packages for vulnerabilities and make sure to keep your dependencies up to date.

#### Npm audit

![](/assests/npm-audit.png "npm audit image")

#### Npm audit fix
![](/assests/npm-audit-fix.png "npm audit fix image")