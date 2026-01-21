# Intelligems External API

> **Beta Access:** Contact support at [Intelligems Support](https://portal.usepylon.com/intelligems/forms/intelligems-support-request) to request API access and get your API key.
>
> Feature suggestions: jerica@intelligems.io

---

## Overview

The Intelligems External API provides programmatic access to your Intelligems experiences and analytics data. This RESTful API allows you to integrate Intelligems functionality into your own applications, workflows, and data pipelines.

### What You Can Do

**Retrieve Test & Experience Data**
- List all tests and experiences in your account
- Get detailed configuration for specific tests and experiences including variations, targeting rules, offer details, and content modifications
- Access test and experience metadata like status, test types, and historical actions

**Access Analytics Data**
- Pull performance metrics for any test or experience
- View results by audience segments (device type, visitor type, traffic source, geography, etc.)
- Export analytics data for custom reporting and analysis

**Automate Workflows**
- Integrate Intelligems data into your business intelligence tools
- Build custom dashboards combining Intelligems metrics with other data sources
- Automate reporting and alerting based on test performance

### Common Use Cases

- **Custom Reporting**: Build internal dashboards that combine Intelligems test results with proprietary business metrics
- **Data Warehouse Integration**: Sync experience and analytics data to your data warehouse for unified analysis
- **Automated Monitoring**: Create alerts when tests reach statistical significance or meet specific performance thresholds
- **Cross-Platform Analysis**: Combine Intelligems data with other analytics platforms for comprehensive performance views

---

## Authentication

Include your API key in the `intelligems-access-token` header:

```
intelligems-access-token: your_api_key_here
```

---

## API Reference

Base URL: `https://api.intelligems.io`

### GET /v25-10-beta/experiences-list

Retrieve a list of all experiences with optional filtering.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| category | string | Filter by `experiment` or `personalization` |
| status | string | Filter by `pending`, `started`, `ended`, or `paused` |
| limit | number/string | Number of results to return |
| page | number/string | Page number for pagination |
| random | boolean | Optional randomization |

**Response:** Returns `experiencesList` array with experience objects containing:
- `id` - UUID of the experience (use this for analytics calls)
- `name` - Experience name
- `status` - Current status
- `category` - experiment or personalization
- `type` - content/advanced, pricing, shipping, etc.
- `startedAtTs` - Timestamp in milliseconds when test started (for runtime calculation)
- `variations` - Array of variation configurations
- `audience` - Targeting rules with `filterType`, `expression`, `action`
- `testTypes` - Object with boolean flags: `hasTestPricing`, `hasTestShipping`, `hasTestCampaign`, `hasTestContent`
- `organizationId` - UUID of the organization

---

### GET /v25-10-beta/experiences/{experienceId}

Get complete details for a specific experience.

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| experienceId | UUID | The experience ID |

**Response:** Returns full `experience` object including:
- All base experience properties
- `variations` with full configuration (price changes, onsite edits, offers, redirects)
- `audience` targeting rules
- `experienceIntegrations` - Connected third-party integrations
- `experienceCustomMetrics` - Custom event tracking
- `experienceKeyMetrics` - Business metrics being tracked
- `shippingTestMethodDefinitions` - Shipping rate configurations

---

### GET /v25-10-beta/analytics/resource/{experienceId}

Access analytics data for an experience with customizable date ranges and audience segmentation.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| view | string | `overview` (default) or `audience` |
| audience | string | Segment by: `device_type`, `visitor_type`, `source_channel`, `source_site`, `country_code`, `landing_page_full_path` |
| start | string | Start date for date range |
| end | string | End date for date range |

**Response:** Returns:
- `datasetId` - "variation_overview"
- `latestRunTs` - Latest calculation timestamp
- `metrics` - Array where **each item represents one variation** (see structure below)
- `variations` - Array with variation details:
  - `id`, `experienceId`, `name`
  - `percentage` - Traffic allocation (0-100)
  - `isControl` - Boolean (true for control/baseline)
  - `order` - Display order
  - `priceChange`, `priceChangeUnit` - For price tests
- `experienceId` and `experienceName`

**Metrics Array Structure:**

Each item in `metrics` represents one variation with metric names as keys:

```json
{
  "metrics": [
    {
      "variation_id": "uuid-of-variation",
      "n_visitors": {"value": 2037},
      "n_orders": {"value": 17},
      "conversion_rate": {
        "value": 0.0083,
        "uplift": {"value": 0.27, "ci_low": -0.30, "ci_high": 1.36},
        "p2bb": 0.77,
        "p2bc": 0.77,
        "ci_low": 0.005,
        "ci_high": 0.013
      },
      "net_revenue_per_visitor": {...},
      "net_revenue_per_order": {...}
    }
  ]
}
```

**Key fields per metric:**
- `value` - The metric value
- `uplift` - Lift vs control (for non-control variations)
- `p2bb` - **Probability to Beat Baseline** (this is the confidence metric, 0-1)
- `p2bc` - Probability to Beat Control
- `ci_low`, `ci_high` - Confidence interval bounds

**Available Metrics:**

*Core Metrics:*
- `n_visitors` - Number of visitors
- `n_orders` - Number of orders
- `conversion_rate` - Conversion rate
- `net_revenue_per_visitor` - Revenue per visitor (**key metric for price tests**)
- `net_revenue_per_order` - AOV (**key metric for offer tests**)
- `gross_profit_per_visitor` - Profit per visitor (requires COGS data)
- `gross_profit_per_order` - Profit per order (requires COGS data)

*Funnel Metrics:*
- `add_to_cart_rate` - Add to cart rate
- `checkout_begin_rate` - Checkout initiation rate
- `checkout_enter_contact_info_rate` - Contact info submission rate
- `checkout_address_submitted_rate` - Address submission rate
- `abandoned_cart_rate` - Cart abandonment rate
- `abandoned_checkout_rate` - Checkout abandonment rate

*Revenue Metrics:*
- `net_revenue` - Total net revenue
- `net_product_revenue` - Product revenue only
- `net_product_revenue_per_order` - Product revenue per order
- `total_discount` - Total discounts applied
- `avg_discount_per_all_orders` - Average discount per order
- `avg_units_per_order` - Average units per order

*COGS/Profit Detection:*
- `pct_revenue_with_cogs` - **Percentage of revenue with COGS data (0-1)**
  - If `== 0`: No COGS configured, `gross_profit_*` equals `net_revenue_*`
  - If `> 0`: COGS data exists, profit metrics are meaningful

*Subscription Metrics:*
- `n_subscription_orders` - Subscription order count
- `subscription_net_revenue` - Subscription revenue
- `subscription_gross_profit` - Subscription profit
- `pct_subscription_orders` - % of orders that are subscriptions

---

## Key Data Structures

### Experience Types
- `content/advanced` - Content tests
- `content/onsiteEdits` - Onsite edit tests
- `content/url` - URL redirect tests
- `pricing` - Price tests
- `shipping` - Shipping tests
- `personalization` - Personalizations
- `personalization/offer` - Offer experiences
- `checkoutBlock` - Checkout block tests

### Variation Properties
- `isControl` - Whether this is the baseline
- `percentage` - Traffic allocation (0-100)
- `priceChange` / `priceChangeUnit` - Price adjustments
- `onsiteEdits` - Content modifications
- `offer` - Discount/offer configuration
- `redirects` - URL redirect rules

### Audience Targeting
- `filterType` - utm, url, device, visitor, country, etc.
- `expression` - Complex filter rules
- `action` - What to do when matched (assignVariation, randomVariation, excludeFromExperience)

---

## Python Code Example

Basic implementation for fetching test data:

```python
import requests
from datetime import datetime
from typing import Optional, List, Dict

API_BASE = "https://api.intelligems.io/v25-10-beta"

class IntelligemsAPI:
    def __init__(self, api_key: str):
        self.headers = {
            "intelligems-access-token": api_key,
            "Content-Type": "application/json"
        }

    def get_active_tests(self) -> List[Dict]:
        """Fetch all currently running tests."""
        url = f"{API_BASE}/experiences-list"
        params = {"status": "started"}
        response = requests.get(url, headers=self.headers, params=params)
        response.raise_for_status()
        return response.json().get("experiencesList", [])

    def get_analytics(self, experience_id: str) -> Dict:
        """Fetch analytics for a specific test."""
        url = f"{API_BASE}/analytics/resource/{experience_id}"
        params = {"view": "overview"}
        response = requests.get(url, headers=self.headers, params=params)
        response.raise_for_status()
        return response.json()

    def get_metric_value(self, metrics: List[Dict], metric_name: str, variation_id: str) -> Optional[float]:
        """Extract a metric value for a specific variation.

        Metrics structure: each item has variation_id and metric names as keys.
        Example: metrics[i].n_visitors.value
        """
        for metric in metrics:
            if metric.get("variation_id") == variation_id:
                metric_data = metric.get(metric_name, {})
                if isinstance(metric_data, dict):
                    return metric_data.get("value")
        return None

    def get_metric_confidence(self, metrics: List[Dict], metric_name: str, variation_id: str) -> Optional[float]:
        """Extract confidence (p2bb = probability to beat baseline) for a metric."""
        for metric in metrics:
            if metric.get("variation_id") == variation_id:
                metric_data = metric.get(metric_name, {})
                if isinstance(metric_data, dict):
                    return metric_data.get("p2bb")  # p2bb = probability to beat baseline
        return None

    def calculate_runtime(self, started_ts: float) -> str:
        """Convert timestamp (ms) to human-readable runtime."""
        start = datetime.fromtimestamp(started_ts / 1000)
        delta = datetime.now() - start
        return f"{delta.days} days"

# Usage
api = IntelligemsAPI("your_api_key_here")
tests = api.get_active_tests()

for test in tests:
    analytics = api.get_analytics(test["id"])
    variations = analytics.get("variations", [])
    metrics = analytics.get("metrics", [])

    # Find control and variant
    for v in variations:
        var_id = v["id"]
        visitors = api.get_metric_value(metrics, "n_visitors", var_id)
        conversion = api.get_metric_value(metrics, "conversion_rate", var_id)
        confidence = api.get_metric_confidence(metrics, "conversion_rate", var_id)

        print(f"{test['name']} - {v['name']}")
        print(f"  Visitors: {visitors}, Conversion: {conversion:.2%}, Confidence: {confidence:.0%}")
```

---

## Health Check Formulas

**Conversion Drop Calculation:**
```
drop = (control_conversion - variant_conversion) / control_conversion
```
Example: Control 5% → Variant 3.5% = (5-3.5)/5 = 30% drop

**Recommended Thresholds (Intelligems Philosophy):**
- `MIN_RUNTIME_DAYS = 10` - Don't call winners before this
- `MIN_VISITORS_FOR_SIGNIFICANCE = 100` - Minimum visitors before results meaningful
- `MIN_ORDERS_FOR_SIGNIFICANCE = 10` - Minimum orders before results meaningful
- `MIN_CONFIDENCE_LEVEL = 0.80` - **80% is enough** ("we're not making cancer medicine")
- `NEUTRAL_LIFT_THRESHOLD = 0.05` - ±5% considered flat/neutral

**Note:** Intelligems philosophy is that 80% confidence is sufficient for most pricing decisions. You can always change it back if needed.

---

## Slack Integration Tutorial

Build automated test health monitoring that sends daily updates to Slack.

### Prerequisites
- Claude Account (Pro, Max, Teams, or Enterprise)
- Intelligems API key
- Slack webhook URL

### Setup Steps

1. **Install Claude Code** in Terminal
2. **Create project directory:**
```bash
mkdir intelligems-slack-health-check
cd intelligems-slack-health-check
python3 -m venv venv
source venv/bin/activate
```

3. **Install dependencies:**
```bash
pip install requests python-dotenv
```

4. **Configure environment:**
```bash
echo "INTELLIGEMS_API_KEY=your_api_key_here" > .env
echo "SLACK_WEBHOOK_URL=your_webhook_url_here" >> .env
```

5. **Create config.py with thresholds:**
```python
MIN_RUNTIME_DAYS = 10  # Don't call winners before this
MIN_VISITORS_FOR_SIGNIFICANCE = 100
MIN_ORDERS_FOR_SIGNIFICANCE = 10
CONVERSION_DROP_ALERT_THRESHOLD = 0.20  # 20% drop triggers alert
MIN_CONFIDENCE_LEVEL = 0.80  # 80% is enough (Intelligems philosophy)
```

6. **Create health check script** (intelligems_health_check.py) that:
   - Fetches active tests via `/experiences-list?status=started`
   - Gets analytics for each via `/analytics/resource/{id}`
   - Calculates runtime, health status, metric comparisons
   - Sends formatted Slack message with results

7. **Schedule daily execution** using LaunchAgent (macOS):
```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>10</integer>
    <key>Minute</key>
    <integer>0</integer>
</dict>
```

### Customizable Thresholds

- **MIN_RUNTIME_DAYS**: Don't call winners before this (default 10)
- **MIN_VISITORS_FOR_SIGNIFICANCE**: Minimum visitors before considering results meaningful
- **MIN_ORDERS_FOR_SIGNIFICANCE**: Minimum orders before considering results meaningful
- **CONVERSION_DROP_ALERT_THRESHOLD**: Alert when variant drops this % vs control
- **MIN_CONFIDENCE_LEVEL**: Statistical confidence required (default 80% - Intelligems philosophy)

### Output Example

Slack message includes:
- Test name and runtime
- Total visitors
- Control vs variant metrics
- Conversion rate, revenue/visitor, AOV comparisons with lift %
- Statistical significance status

---

## Related Documentation

- [JavaScript API](https://docs.intelligems.io/developer-resources/javascript-api)
- [MCP Server](https://docs.intelligems.io/developer-resources/mcp-server)
- [Webhook Integration](https://docs.intelligems.io/integrations/webhook-integration)
