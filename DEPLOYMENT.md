# Mesho Unlocker SaaS Backend - Deployment Guide

## Architecture Overview

Hybrid Architecture:
- **Frontend**: WordPress on CyberPanel (mesho-unlocker.com)
- **Backend API**: NestJS on Render
- **Database**: PostgreSQL on Render
- **Payment**: Stripe + Paymob integration
- **Authentication**: JWT tokens

## Quick Start Deployment to Render

### Step 1: Prepare Your Local Environment

```bash
# Install Node.js 18+ and npm
node --version  # Should be v18+
npm --version   # Should be v9+

# Clone the repository
git clone https://github.com/agentive2025-pixel/mesho-unlocker-saas-backend.git
cd mesho-unlocker-saas-backend

# Install dependencies
npm install
```

### Step 2: Create .env File Locally

Create `.env` file in project root:

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/mesho_db
DATABASE_NAME=mesho_db
DATABASE_USER=mesho_user
DATABASE_PASSWORD=your_secure_password_here
DATABASE_PORT=5432

JWT_SECRET=your_jwt_secret_key_here_min_32_chars
JWT_EXPIRATION=7d

STRIPE_SECRET_KEY=sk_test_your_stripe_key_here
STRIPE_PUBLISHABLE_KEY=pk_test_your_stripe_key_here
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret

PAYMOB_API_KEY=your_paymob_api_key
PAYMOB_SECRET=your_paymob_secret

CORS_ORIGIN=https://mesho-unlocker.com
FRONTEND_URL=https://mesho-unlocker.com

ADMIN_EMAIL=admin@mesho-unlocker.com
ADMIN_PASSWORD=initial_admin_password
```

### Step 3: Create Render Account & Deploy

1. Go to https://render.com
2. Sign up with GitHub account
3. Click "New +" → "Web Service"
4. Connect your GitHub repository: `mesho-unlocker-saas-backend`
5. Configure:
   - **Name**: mesho-unlocker-api
   - **Environment**: Node
   - **Region**: Ohio (or closest to you)
   - **Branch**: main
   - **Build Command**: `npm run build`
   - **Start Command**: `npm run start:prod`

### Step 4: Configure Environment Variables in Render

After creating the web service, go to "Environment" tab and add:

```
NODE_ENV = production
DATABASE_URL = (Will be set by PostgreSQL database)
JWT_SECRET = (Generate a random 32+ char string)
STRIPE_SECRET_KEY = sk_live_your_key
PAYMOB_API_KEY = your_key
CORS_ORIGIN = https://mesho-unlocker.com
FRONTEND_URL = https://mesho-unlocker.com
```

### Step 5: Create PostgreSQL Database on Render

1. In Render Dashboard, click "New +" → "PostgreSQL"
2. Configure:
   - **Name**: mesho-db
   - **Database**: mesho_db
   - **User**: mesho_user
   - **Region**: Same as web service
3. Copy the **Internal Database URL** (ends with?ssl=require)
4. Add to Web Service environment as `DATABASE_URL`

### Step 6: Run Database Migrations

After deployment:

```bash
# Locally, test migrations:
npm run migration:run

# Or through Render's shell:
render exec -s mesho-unlocker-api npm run migration:run
```

## API Endpoints (After Deployment)

### Authentication
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login user
- `POST /api/auth/refresh` - Refresh JWT token
- `POST /api/auth/logout` - Logout user

### Users
- `GET /api/users/profile` - Get user profile
- `PUT /api/users/profile` - Update profile
- `GET /api/users/dashboard` - User dashboard data

### Services
- `GET /api/services` - List all services
- `GET /api/services/:id` - Get service details
- `POST /api/services` - Create service (admin only)

### Orders
- `POST /api/orders` - Create new order
- `GET /api/orders/:id` - Get order details
- `GET /api/orders` - List user's orders
- `PATCH /api/orders/:id/status` - Update order status (admin)

### Payments
- `POST /api/payments/stripe` - Process Stripe payment
- `POST /api/payments/paymob` - Process Paymob payment
- `POST /api/payments/webhook/stripe` - Stripe webhook
- `POST /api/payments/webhook/paymob` - Paymob webhook

### Admin Panel
- `GET /api/admin/dashboard` - Admin statistics
- `GET /api/admin/users` - Manage users
- `GET /api/admin/orders` - Manage orders
- `GET /api/admin/services` - Manage services

## WordPress Integration (Frontend)

### 1. Create Required WordPress Pages

- Dashboard: `/user-dashboard`
- Services: `/services`
- Order Form: `/place-order`
- Payment: `/checkout`
- Admin: `/admin-panel`

### 2. Install Required Plugins

```
- Code Snippets (for API integration code)
- ACF Pro (for custom fields)
- WP REST API plugins
```

### 3. Add API Integration Code

Create a custom plugin in `/wp-content/plugins/mesho-api-integration/`:

**mesho-api-integration.php**:
```php
<?php
/**
 * Plugin Name: Mesho Unlocker API Integration
 * Description: Connects WordPress frontend to NestJS backend
 * Version: 1.0
 */

define('MESHO_API_URL', 'https://mesho-unlocker-api.onrender.com');
define('MESHO_API_KEY', get_option('mesho_api_key'));

// Fetch services from API
function mesho_get_services() {
    $response = wp_remote_get(
        MESHO_API_URL . '/api/services',
        array(
            'headers' => array('Accept' => 'application/json')
        )
    );
    
    if (!is_wp_error($response)) {
        return json_decode(wp_remote_retrieve_body($response), true);
    }
    return array();
}

// Create order via API
function mesho_create_order($user_data, $service_id) {
    $response = wp_remote_post(
        MESHO_API_URL . '/api/orders',
        array(
            'headers' => array(
                'Content-Type' => 'application/json',
                'Authorization' => 'Bearer ' . get_user_meta(get_current_user_id(), 'api_token', true)
            ),
            'body' => json_encode(array(
                'service_id' => $service_id,
                'user_data' => $user_data
            ))
        )
    );
    
    return json_decode(wp_remote_retrieve_body($response), true);
}

// Handle Stripe payment callback
function mesho_process_payment($order_id, $payment_method) {
    // Implementation for payment processing
}

add_action('wp_ajax_mesho_create_order', function() {
    if (!is_user_logged_in()) {
        wp_send_json_error('Not authenticated', 401);
    }
    
    $service_id = $_POST['service_id'];
    $order = mesho_create_order($_POST['user_data'], $service_id);
    
    wp_send_json_success($order);
});
?>
```

### 4. Create Order Form Page

Add shortcode in WordPress page:

```html
[mesho_order_form]

<script>
document.addEventListener('DOMContentLoaded', function() {
    const apiUrl = 'https://mesho-unlocker-api.onrender.com';
    
    // Fetch and display services
    fetch(apiUrl + '/api/services')
        .then(r => r.json())
        .then(services => {
            const select = document.getElementById('service-select');
            services.forEach(s => {
                const option = document.createElement('option');
                option.value = s.id;
                option.textContent = s.name + ' - ' + s.price + ' EGP';
                select.appendChild(option);
            });
        });
    
    // Handle form submission
    document.getElementById('order-form').addEventListener('submit', async(e) => {
        e.preventDefault();
        
        const formData = new FormData(e.target);
        const response = await fetch(apiUrl + '/api/orders', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': 'Bearer ' + localStorage.getItem('token')
            },
            body: JSON.stringify(Object.fromEntries(formData))
        });
        
        const order = await response.json();
        
        if (order.payment_required) {
            // Redirect to payment gateway
            window.location.href = apiUrl + '/api/payments/redirect?order_id=' + order.id;
        }
    });
});
</script>
```

## Deployed URLs

### After Render Deployment, You'll Have:

- **Backend Base URL**: `https://mesho-unlocker-api.onrender.com`
- **API Docs**: `https://mesho-unlocker-api.onrender.com/api/docs` (Swagger UI)
- **Admin Panel**: `https://mesho-unlocker-api.onrender.com/admin`
- **Database URL**: Available in Render PostgreSQL details
- **GitHub Repo**: https://github.com/agentive2025-pixel/mesho-unlocker-saas-backend

## Stripe Setup

1. Go to https://stripe.com → Create account
2. Get API keys from Dashboard → API Keys
3. Set up webhook endpoint: `https://mesho-unlocker-api.onrender.com/api/payments/webhook/stripe`
4. Add events: `charge.succeeded`, `charge.failed`

## Paymob Setup

1. Register at Paymob.com
2. Get API key and secret
3. Add to environment variables
4. Configure webhook: `https://mesho-unlocker-api.onrender.com/api/payments/webhook/paymob`

## Testing

```bash
# Local testing
npm run start:dev

# Run tests
npm test

# Coverage report
npm run test:cov
```

## Troubleshooting

### Database Connection Issues
- Verify DATABASE_URL is correct in Render environment
- Check PostgreSQL is running
- Ensure SSL is required (add ?ssl=require to URL)

### CORS Errors
- Update CORS_ORIGIN to match WordPress domain
- Check API returns proper CORS headers

### Payment Gateway Issues
- Verify API keys are correct
- Check webhook endpoints are accessible
- Test with Stripe test mode first

## Support & Documentation

- Render Docs: https://render.com/docs
- NestJS Docs: https://docs.nestjs.com
- Stripe API: https://stripe.com/docs/api
- Paymob API: https://paymob.readme.io/
