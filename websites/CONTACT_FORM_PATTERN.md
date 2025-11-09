# Contact Form Pattern

A framework-agnostic guide to implementing a contact form with serverless email functionality using Netlify Functions and Mailgun (or similar email service).

## Architecture Overview

This pattern uses a three-layer architecture:

1. **Frontend Form Component** - Captures user input and handles submission
2. **Serverless Function** - Processes form data and sends emails
3. **Email Service (Mailgun)** - Delivers the email to your inbox

```
[User Form] → [Netlify Function] → [Mailgun API] → [Your Email]
```

## Prerequisites

- Netlify account (or similar serverless platform)
- Mailgun account (or alternative: SendGrid, AWS SES, Postmark, etc.)
- Node.js environment for the serverless function

## Part 1: Email Service Setup (Mailgun)

### 1.1 Mailgun Configuration

1. Sign up for a Mailgun account at https://www.mailgun.com/
2. Get your API credentials from the Mailgun dashboard:
   - API Key (private key)
   - Domain (sandbox domain for testing or your verified domain)
3. Optional: Create an email template in Mailgun for consistent formatting

### 1.2 Environment Variables

Set the following environment variables in your Netlify dashboard (or `.env` for local development):

```
MG_KEY=your-mailgun-api-key
MG_FROM=mailgun@sandbox1234.mailgun.org
MG_DOMAIN=sandbox1234.mailgun.org
MG_TO=your-email@example.com
MG_TEMPLATE=template-name (optional)
```

**Security Note:** Never commit API keys to version control. Always use environment variables.

## Part 2: Project Configuration

### 2.1 Netlify Configuration

Create a `netlify.toml` file in your project root:

```toml
[build]
  command = "npm run build"  # Your build command
  functions = "netlify/functions"  # Or "functions"
  publish = "dist"  # Your output directory
```

### 2.2 Dependencies

For the serverless function, install:

```bash
npm install mailgun.js form-data
```

Or add to your `package.json`:

```json
{
  "dependencies": {
    "mailgun.js": "^latest",
    "form-data": "^latest"
  }
}
```

## Part 3: Serverless Function Implementation

### 3.1 Create the Function

Create a file at `functions/contact.js` (or `netlify/functions/contact.js` depending on your config):

```javascript
const formData = require('form-data');
const Mailgun = require('mailgun.js');

/**
 * Sends an email using Mailgun with the provided form data
 * @param {string} data - JSON stringified form data
 */
async function sendEmailWithFormInfo(data) {
  const emailBody = JSON.parse(data);

  // Get environment variables
  const {
    MG_FROM: emailFrom,
    MG_TO: emailTo,
    MG_KEY: apiKey,
    MG_DOMAIN: domain,
    MG_TEMPLATE: template
  } = process.env;

  // Initialize Mailgun client
  const mailgun = new Mailgun(formData);
  const mg = mailgun.client({ username: 'api', key: apiKey });

  // Prepare email data
  const mailData = {
    from: emailFrom,
    to: emailTo,
    subject: `New contact form submission from ${emailBody.name}`,
    template: template,  // Optional: use Mailgun template
    'h:X-Mailgun-Variables': data  // Pass variables to template
  };

  // Alternative: Send plain text/HTML without template
  // const mailData = {
  //   from: emailFrom,
  //   to: emailTo,
  //   subject: `New contact form submission from ${emailBody.name}`,
  //   text: `Name: ${emailBody.name}\nEmail: ${emailBody.email}\nComments: ${emailBody.comments}`,
  //   html: `<h3>New Contact Form Submission</h3>
  //          <p><strong>Name:</strong> ${emailBody.name}</p>
  //          <p><strong>Email:</strong> ${emailBody.email}</p>
  //          <p><strong>Comments:</strong> ${emailBody.comments}</p>`
  // };

  try {
    await mg.messages.create(domain, mailData);
  } catch (error) {
    console.error("Error sending email: ", error);
    throw new Error("Error on server, please contact site admin");
  }
}

/**
 * Netlify Function handler
 * @param {Object} event - Netlify event object
 */
const handler = async (event) => {
  // Only allow POST requests
  if (event.httpMethod !== 'POST') {
    return {
      statusCode: 405,
      body: JSON.stringify({ error: 'Method Not Allowed' })
    };
  }

  try {
    await sendEmailWithFormInfo(event.body);
    return {
      statusCode: 200,
      body: JSON.stringify({ message: 'Email sent successfully' })
    };
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.toString() })
    };
  }
}

module.exports = { handler };
```

### 3.2 Function Endpoint

Once deployed, your function will be available at:
```
https://your-site.netlify.app/.netlify/functions/contact
```

For local development with Netlify CLI:
```
http://localhost:8888/.netlify/functions/contact
```

## Part 4: Frontend Implementation

### 4.1 Core Concepts (Framework Agnostic)

The frontend needs to:
1. Capture form field values
2. Validate input (email format, required fields)
3. Submit data as JSON to the serverless function
4. Display success/error notifications to the user
5. Clear the form on successful submission

### 4.2 Example: Vanilla JavaScript

```html
<!DOCTYPE html>
<html>
<body>
  <form id="contactForm">
    <label for="name">Name:</label>
    <input type="text" id="name" name="name" required>

    <label for="email">Email:</label>
    <input type="email" id="email" name="email" required>

    <label for="comments">Comments:</label>
    <textarea id="comments" name="comments" required></textarea>

    <button type="submit">Send</button>
  </form>

  <div id="notification" style="display: none;"></div>

  <script>
    const form = document.getElementById('contactForm');
    const notification = document.getElementById('notification');

    function notify(message) {
      notification.textContent = message;
      notification.style.display = 'block';
      setTimeout(() => {
        notification.style.display = 'none';
      }, 3000);
    }

    function clearForm() {
      form.reset();
    }

    async function handleSubmit(e) {
      e.preventDefault();

      const formData = {
        name: document.getElementById('name').value,
        email: document.getElementById('email').value,
        comments: document.getElementById('comments').value
      };

      try {
        const response = await fetch('/.netlify/functions/contact', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(formData)
        });

        if (response.ok) {
          notify('Email sent successfully!');
          clearForm();
        } else {
          notify('Failed to send email. Please try again.');
        }
      } catch (error) {
        console.error('Error:', error);
        notify('An error occurred. Please try again.');
      }
    }

    form.addEventListener('submit', handleSubmit);
  </script>
</body>
</html>
```

### 4.3 Example: React

```jsx
import { useState } from 'react';

function ContactForm() {
  const [form, setForm] = useState({
    name: '',
    email: '',
    comments: ''
  });

  const [notification, setNotification] = useState({ show: false, message: '' });

  const notify = (message) => {
    setNotification({ show: true, message });
    setTimeout(() => {
      setNotification({ show: false, message: '' });
    }, 3000);
  };

  const clearForm = () => {
    setForm({ name: '', email: '', comments: '' });
  };

  const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();

    try {
      const response = await fetch('/.netlify/functions/contact', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(form)
      });

      if (response.ok) {
        notify('Email sent successfully!');
        clearForm();
      } else {
        notify('Failed to send email. Please try again.');
      }
    } catch (error) {
      console.error('Error:', error);
      notify('An error occurred. Please try again.');
    }
  };

  return (
    <div>
      {notification.show && <div className="notification">{notification.message}</div>}

      <form onSubmit={handleSubmit}>
        <label htmlFor="name">Name:</label>
        <input
          type="text"
          id="name"
          name="name"
          value={form.name}
          onChange={handleChange}
          required
        />

        <label htmlFor="email">Email:</label>
        <input
          type="email"
          id="email"
          name="email"
          value={form.email}
          onChange={handleChange}
          required
        />

        <label htmlFor="comments">Comments:</label>
        <textarea
          id="comments"
          name="comments"
          value={form.comments}
          onChange={handleChange}
          required
        />

        <button type="submit">Send</button>
      </form>
    </div>
  );
}

export default ContactForm;
```

### 4.4 Example: Vue 3

```vue
<template>
  <div>
    <div v-if="notification.show" class="notification">
      {{ notification.message }}
    </div>

    <form @submit.prevent="handleSubmit">
      <label for="name">Name:</label>
      <input
        type="text"
        id="name"
        v-model="form.name"
        required
      />

      <label for="email">Email:</label>
      <input
        type="email"
        id="email"
        v-model="form.email"
        required
      />

      <label for="comments">Comments:</label>
      <textarea
        id="comments"
        v-model="form.comments"
        required
      />

      <button type="submit">Send</button>
    </form>
  </div>
</template>

<script setup>
import { reactive } from 'vue';

const form = reactive({
  name: '',
  email: '',
  comments: ''
});

const notification = reactive({
  show: false,
  message: ''
});

const notify = (message) => {
  notification.message = message;
  notification.show = true;
  setTimeout(() => {
    notification.show = false;
  }, 3000);
};

const clearForm = () => {
  form.name = '';
  form.email = '';
  form.comments = '';
};

const handleSubmit = async () => {
  try {
    const response = await fetch('/.netlify/functions/contact', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(form)
    });

    if (response.ok) {
      notify('Email sent successfully!');
      clearForm();
    } else {
      notify('Failed to send email. Please try again.');
    }
  } catch (error) {
    console.error('Error:', error);
    notify('An error occurred. Please try again.');
  }
};
</script>
```

### 4.5 Example: Svelte (Reference Implementation)

```svelte
<script>
  let form = {
    name: '',
    email: '',
    comments: ''
  };

  let notifying = false;
  let notifyMessage = '';

  const notify = (message) => {
    notifyMessage = message;
    notifying = true;
    setTimeout(() => {
      notifying = false;
    }, 3000);
  };

  const clearForm = () => {
    form = {
      name: '',
      email: '',
      comments: ''
    };
  };

  const handleSubmit = async () => {
    try {
      const response = await fetch('/.netlify/functions/contact', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(form)
      });

      if (response.ok) {
        notify('Email sent successfully!');
        clearForm();
      } else {
        notify('Failed to send email. Please try again.');
      }
    } catch (error) {
      console.error('Error:', error);
      notify('An error occurred. Please try again.');
    }
  };
</script>

{#if notifying}
  <div class="notification">{notifyMessage}</div>
{/if}

<form on:submit|preventDefault={handleSubmit}>
  <label for="name">Name:</label>
  <input
    type="text"
    id="name"
    bind:value={form.name}
    required
  />

  <label for="email">Email:</label>
  <input
    type="email"
    id="email"
    bind:value={form.email}
    required
  />

  <label for="comments">Comments:</label>
  <textarea
    id="comments"
    bind:value={form.comments}
    required
  />

  <button type="submit">Send</button>
</form>
```

## Part 5: Customization & Extension

### 5.1 Adding Custom Fields

To add additional form fields (e.g., phone number, company, subject):

**Frontend:** Add new input fields and include them in the form state:
```javascript
const formData = {
  name: '...',
  email: '...',
  phone: '...',      // New field
  company: '...',    // New field
  subject: '...',    // New field
  comments: '...'
};
```

**Backend:** The serverless function automatically handles any fields you pass. Update the email template or HTML to include the new fields:
```javascript
text: `
  Name: ${emailBody.name}
  Email: ${emailBody.email}
  Phone: ${emailBody.phone}
  Company: ${emailBody.company}
  Subject: ${emailBody.subject}
  Comments: ${emailBody.comments}
`
```

### 5.2 Using Alternative Email Services

#### SendGrid
```javascript
const sgMail = require('@sendgrid/mail');

sgMail.setApiKey(process.env.SENDGRID_API_KEY);

const msg = {
  to: process.env.TO_EMAIL,
  from: process.env.FROM_EMAIL,
  subject: `New contact from ${emailBody.name}`,
  text: `Name: ${emailBody.name}...`,
};

await sgMail.send(msg);
```

#### AWS SES
```javascript
const AWS = require('aws-sdk');
const ses = new AWS.SES({ region: 'us-east-1' });

const params = {
  Source: process.env.FROM_EMAIL,
  Destination: { ToAddresses: [process.env.TO_EMAIL] },
  Message: {
    Subject: { Data: `New contact from ${emailBody.name}` },
    Body: { Text: { Data: `Name: ${emailBody.name}...` } }
  }
};

await ses.sendEmail(params).promise();
```

### 5.3 Adding Validation

**Frontend validation:**
```javascript
function validateEmail(email) {
  const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return re.test(email);
}

function validateForm(formData) {
  if (!formData.name || formData.name.trim().length < 2) {
    return { valid: false, error: 'Name must be at least 2 characters' };
  }

  if (!validateEmail(formData.email)) {
    return { valid: false, error: 'Invalid email address' };
  }

  if (!formData.comments || formData.comments.trim().length < 10) {
    return { valid: false, error: 'Comments must be at least 10 characters' };
  }

  return { valid: true };
}
```

**Backend validation:**
```javascript
function validateFormData(data) {
  const { name, email, comments } = data;

  if (!name || !email || !comments) {
    throw new Error('Missing required fields');
  }

  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    throw new Error('Invalid email format');
  }

  // Add spam detection, rate limiting, etc.
}
```

### 5.4 Adding Rate Limiting / Spam Protection

```javascript
// Simple in-memory rate limiting (consider using Redis for production)
const submissions = new Map();

function checkRateLimit(identifier) {
  const now = Date.now();
  const windowMs = 60 * 1000; // 1 minute
  const maxRequests = 3;

  const userSubmissions = submissions.get(identifier) || [];
  const recentSubmissions = userSubmissions.filter(time => now - time < windowMs);

  if (recentSubmissions.length >= maxRequests) {
    throw new Error('Too many requests. Please try again later.');
  }

  recentSubmissions.push(now);
  submissions.set(identifier, recentSubmissions);
}

// In handler:
const identifier = event.headers['x-forwarded-for'] || 'unknown';
checkRateLimit(identifier);
```

### 5.5 Adding CAPTCHA

**Frontend (reCAPTCHA example):**
```html
<script src="https://www.google.com/recaptcha/api.js" async defer></script>

<form>
  <!-- form fields -->
  <div class="g-recaptcha" data-sitekey="your-site-key"></div>
  <button type="submit">Send</button>
</form>
```

```javascript
const formData = {
  // ... form fields
  recaptchaToken: grecaptcha.getResponse()
};
```

**Backend verification:**
```javascript
async function verifyRecaptcha(token) {
  const response = await fetch('https://www.google.com/recaptcha/api/siteverify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: `secret=${process.env.RECAPTCHA_SECRET}&response=${token}`
  });

  const data = await response.json();
  return data.success;
}
```

## Part 6: Testing

### 6.1 Local Development

Install Netlify CLI:
```bash
npm install -g netlify-cli
```

Run locally:
```bash
netlify dev
```

This will:
- Start your development server
- Make functions available at `http://localhost:8888/.netlify/functions/contact`
- Load environment variables from `.env` file

### 6.2 Testing the Function

Using cURL:
```bash
curl -X POST http://localhost:8888/.netlify/functions/contact \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","comments":"Test message"}'
```

Using JavaScript:
```javascript
fetch('/.netlify/functions/contact', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'Test User',
    email: 'test@example.com',
    comments: 'Test message'
  })
})
.then(res => res.json())
.then(console.log)
.catch(console.error);
```

## Part 7: Deployment

### 7.1 Deploy to Netlify

1. Push your code to a Git repository (GitHub, GitLab, Bitbucket)
2. Connect your repository to Netlify
3. Set environment variables in Netlify dashboard:
   - Site Settings → Build & deploy → Environment
   - Add: `MG_KEY`, `MG_FROM`, `MG_DOMAIN`, `MG_TO`, `MG_TEMPLATE`
4. Deploy!

### 7.2 Alternative Serverless Platforms

**Vercel:**
- Create `api/contact.js` instead of `functions/contact.js`
- Export handler as default: `module.exports = handler;` → `export default handler;`

**AWS Lambda:**
- Package function with dependencies
- Set up API Gateway endpoint
- Configure environment variables in Lambda console

**Cloudflare Workers:**
- Adapt handler to Cloudflare Workers API
- Use Workers KV or environment variables for config

## Part 8: Troubleshooting

### Common Issues

**Issue:** "Function not found" error
- **Solution:** Check `netlify.toml` functions path matches your directory structure

**Issue:** Emails not sending
- **Solution:** Verify environment variables are set correctly in Netlify dashboard
- **Solution:** Check Mailgun logs in dashboard for errors

**Issue:** CORS errors
- **Solution:** Ensure you're calling the same domain (functions are on same origin as your site)

**Issue:** Form submits but no notification
- **Solution:** Remove `mode: 'no-cors'` from fetch options to properly handle response

**Issue:** Empty emails received
- **Solution:** Verify form field names match what the function expects

## Security Best Practices

1. **Never expose API keys** - Always use environment variables
2. **Validate input** - Both client and server side
3. **Rate limiting** - Prevent spam/abuse
4. **Sanitize data** - Prevent injection attacks
5. **Use HTTPS** - Netlify provides this by default
6. **CAPTCHA** - Consider adding for public forms
7. **Email validation** - Verify email format before sending

## Summary

This pattern provides a secure, scalable way to handle contact forms without exposing email addresses or API keys to the frontend. The serverless function acts as a secure intermediary between your form and email service.

**Key Benefits:**
- No backend server to maintain
- Secure API key handling
- Framework agnostic (works with any frontend)
- Easy to customize and extend
- Cost-effective (Netlify/Mailgun free tiers available)

**Reference Implementation:**
- Frontend: `/src/components/sections/ContactForm.svelte`
- Backend: `/functions/contact.js`
- Config: `/netlify.toml`
