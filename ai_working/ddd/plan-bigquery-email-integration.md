# Phase 2: BigQuery Integration for Email Reporting

## Project Overview

Integrate BigQuery data into the Maizzle email reporting app (`extra-point-data/apps/client-report`) to enable report generation in Google Cloud Functions.

**Current State:** MVP loads from local JSON files (`data/processed/weekly_email_data/*.json`)
**Target State:** Pull data directly from BigQuery or via Cloud Storage bucket

---

## Data Structure Mapping

### Template Requirements vs BigQuery Schema

#### Revenue Summary Data

**Template expects:**
```json
{
  "summary": {
    "rb_revenue": {"weekly": "$10,874", "total": "$64,837"},
    "ep_fees": {"weekly": "$1,208", "total": "$7,204"},
    "tickets_sold": {"weekly": "49", "total": "282"},
    "avg_ticket_price": {"weekly": "$247", "total": "$255"}
  }
}
```

**BigQuery provides (rpt_revenue_weekly):**
```sql
- order_week_end_date (STRING)
- client_revenue (FLOAT)        → maps to "rb_revenue"
- ep_fees (FLOAT)                → direct match
- tickets_sold (INTEGER)         → direct match
- average_ticket_price (FLOAT)   → maps to "avg_ticket_price"
- face_value (FLOAT)             → NEW DATA (not in template)
- profit (FLOAT)                 → NEW DATA (not in template)
- lift (FLOAT)                   → NEW DATA (not in template)
```

#### Section Performance Data

**Template expects:**
```json
{
  "section_performance": {
    "categories": [
      {
        "name": "Lower Endzone",
        "inventory": "492",
        "sell_through": "14.2%",
        "sections": [
          {"name": "10-H", "inventory": "161", "sold": "23"}
        ]
      }
    ]
  }
}
```

**BigQuery provides (rpt_section_performance):**
```sql
- price_category (STRING)        → maps to category "name"
- section (STRING)               → maps to section "name"
- inventory (INTEGER)            → direct match
- sold (INTEGER)                 → available but not in current template
- available (INTEGER)            → NEW DATA
- sell_through_pct (FLOAT)       → maps to "sell_through"
- avg_price (FLOAT)              → NEW DATA
- revenue (FLOAT)                → NEW DATA
- face_value_total (FLOAT)       → NEW DATA
- profit (FLOAT)                 → NEW DATA
- lift (FLOAT)                   → NEW DATA
```

### Critical Data Gaps

1. **`event_name`** - NOT in BigQuery! Need to add to schema or pass as parameter
2. **`report_date`** - Can derive from `order_week_end_date` but needs formatting
3. **Weekly vs All-Time** - Current template shows "this_week" vs "all_time" but BigQuery only has weekly snapshots
   - **Options:**
     - Query multiple weeks and aggregate for "all_time" (complex, slow)
     - Add cumulative columns to BigQuery views (recommended)
     - Simplify template to weekly-only metrics (easiest MVP)

### Data Transformations Required

1. **Field Renaming:** `client_revenue` → `rb_revenue`, `average_ticket_price` → `avg_ticket_price`
2. **Currency Formatting:** FLOAT → "$10,874" string format
3. **Percentage Formatting:** 0.142 → "14.2%" string format
4. **Grouping:** Aggregate sections by `price_category` into nested structure
5. **Date Formatting:** "2025-11-14" → "November 14, 2025"

---

## Option A: BigQuery Node SDK Direct Integration

### Architecture

```
Cloud Function (Node.js)
  ↓
@google-cloud/bigquery
  ↓
BigQuery API
  ↓
Transform data in-memory
  ↓
Maizzle email generation
  ↓
Return HTML email
```

### Implementation Approach

**1. Install Dependencies:**
```bash
cd apps/client-report
npm install @google-cloud/bigquery
```

**2. Replace `scripts/load-data.js`:**
```javascript
import { BigQuery } from '@google-cloud/bigquery';

const bigquery = new BigQuery({
  projectId: 'turing-nature-473514-m6'
});

export async function loadWeeklyData(eventName, weekEndDate) {
  // Query revenue summary
  const revenueSummary = await querySummary(weekEndDate);

  // Query section performance
  const sectionPerformance = await querySectionPerformance(weekEndDate);

  // Transform to template format
  return {
    event_name: eventName,
    report_date: formatReportDate(weekEndDate),
    summary: transformSummary(revenueSummary),
    section_performance: transformSectionPerformance(sectionPerformance)
  };
}

async function querySummary(weekEndDate) {
  const query = `
    SELECT
      client_revenue,
      ep_fees,
      tickets_sold,
      average_ticket_price,
      face_value,
      profit,
      lift
    FROM \`extrapoint_marts.rpt_revenue_weekly\`
    WHERE order_week_end_date = @weekEndDate
  `;

  const [rows] = await bigquery.query({
    query,
    params: { weekEndDate }
  });

  return rows[0];
}

async function querySectionPerformance(weekEndDate) {
  const query = `
    SELECT
      price_category,
      section,
      inventory,
      sold,
      sell_through_pct,
      avg_price,
      revenue
    FROM \`extrapoint_marts.rpt_section_performance\`
    WHERE order_week_end_date = @weekEndDate
    ORDER BY price_category, section
  `;

  const [rows] = await bigquery.query({ query, params: { weekEndDate } });
  return rows;
}

function transformSummary(row) {
  return {
    rb_revenue: {
      weekly: formatCurrency(row.client_revenue),
      total: "N/A" // Need to add cumulative to BigQuery or query multiple weeks
    },
    ep_fees: {
      weekly: formatCurrency(row.ep_fees),
      total: "N/A"
    },
    tickets_sold: {
      weekly: row.tickets_sold.toString(),
      total: "N/A"
    },
    avg_ticket_price: {
      weekly: formatCurrency(row.average_ticket_price),
      total: "N/A"
    }
  };
}

function transformSectionPerformance(rows) {
  const categories = {};

  for (const row of rows) {
    if (!categories[row.price_category]) {
      categories[row.price_category] = {
        name: row.price_category,
        inventory: 0,
        sell_through: 0,
        sections: []
      };
    }

    const category = categories[row.price_category];
    category.inventory += row.inventory;
    category.sections.push({
      name: row.section,
      inventory: row.inventory.toString(),
      sold: row.sold.toString()
    });
  }

  // Calculate category-level sell_through
  for (const category of Object.values(categories)) {
    const totalSold = category.sections.reduce((sum, s) => sum + parseInt(s.sold), 0);
    category.sell_through = ((totalSold / category.inventory) * 100).toFixed(1) + '%';
    category.inventory = category.inventory.toString();
  }

  return { categories: Object.values(categories) };
}

function formatCurrency(amount) {
  return '$' + Math.round(amount).toLocaleString();
}

function formatReportDate(isoDate) {
  const date = new Date(isoDate);
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
}
```

**3. Update `config.js` Maizzle config:**
```javascript
export default {
  build: {
    beforeCreate: async ({ config }) => {
      // Load data from BigQuery
      const eventName = process.env.EVENT_NAME || 'Rose Bowl 2025';
      const weekEndDate = process.env.WEEK_END_DATE || '2025-11-14';

      const weeklyData = await loadWeeklyData(eventName, weekEndDate);
      config.weekly = weeklyData;
    }
  }
}
```

**4. Cloud Function Entry Point:**
```javascript
import { loadWeeklyData } from './scripts/load-data.js';
import { build } from '@maizzle/framework';

export async function generateEmail(req, res) {
  const { eventName, weekEndDate } = req.body;

  // Load data from BigQuery
  const weeklyData = await loadWeeklyData(eventName, weekEndDate);

  // Build email with Maizzle
  const { html } = await build({
    maizzle: { weekly: weeklyData },
    templates: ['emails/weekly-report.html']
  });

  res.status(200).send(html);
}
```

### Pros

✅ **Single Runtime:** Node.js only - no Python/Node interop
✅ **Direct Data Access:** No intermediate storage, always fresh data
✅ **Simpler Pipeline:** Fewer moving parts (no BigQuery → bucket export job)
✅ **Dynamic Queries:** Can parameterize by event, date range, filters
✅ **Easier Development:** Test queries directly, immediate feedback
✅ **Lower Latency:** No storage I/O, just BigQuery → transform → email

### Cons

❌ **Cold Start Impact:** BigQuery SDK adds ~3-5 seconds to cold start
❌ **Larger Bundle:** `@google-cloud/bigquery` is 5-10 MB
❌ **Query Cost:** Every invocation = BigQuery query (minimal $ but tracked)
❌ **Error Handling:** Must handle query failures, timeouts, schema changes
❌ **Authentication:** Need service account with BigQuery read permissions
❌ **Development Complexity:** More code paths to test (queries, transforms, error cases)

---

## Option B: BigQuery Export to Cloud Storage Bucket

### Architecture

```
Scheduled BigQuery Job (daily/weekly)
  ↓
EXPORT DATA AS JSON
  ↓
Cloud Storage Bucket (gs://extrapoint-reports/weekly_email_data/)
  ↓
Cloud Function (Node.js)
  ↓
Read JSON from bucket
  ↓
Minimal transformation
  ↓
Maizzle email generation
  ↓
Return HTML email
```

### Implementation Approach

**1. Create BigQuery Export Job (SQL scheduled query):**

```sql
EXPORT DATA OPTIONS(
  uri='gs://extrapoint-reports/weekly_email_data/data_*.json',
  format='JSON',
  overwrite=true
) AS

WITH weekly_summary AS (
  SELECT
    order_week_end_date,
    FORMAT('%s', DATE(order_week_end_date)) as report_date,
    FORMAT('$%s', FORMAT('%\'.0f', client_revenue)) as client_revenue_formatted,
    FORMAT('$%s', FORMAT('%\'.0f', ep_fees)) as ep_fees_formatted,
    CAST(tickets_sold AS STRING) as tickets_sold_str,
    FORMAT('$%s', FORMAT('%\'.0f', average_ticket_price)) as avg_ticket_price_formatted
  FROM `extrapoint_marts.rpt_revenue_weekly`
  WHERE order_week_end_date = CURRENT_DATE() - 7  -- or parameterized
),

section_summary AS (
  SELECT
    price_category,
    section,
    CAST(inventory AS STRING) as inventory_str,
    CAST(sold AS STRING) as sold_str,
    FORMAT('%.1f%%', sell_through_pct * 100) as sell_through_formatted
  FROM `extrapoint_marts.rpt_section_performance`
  WHERE order_week_end_date = CURRENT_DATE() - 7
)

SELECT
  'Rose Bowl 2025' as event_name,  -- Need to parameterize or add to BigQuery
  ws.report_date,
  STRUCT(
    STRUCT(ws.client_revenue_formatted as weekly, 'N/A' as total) as rb_revenue,
    STRUCT(ws.ep_fees_formatted as weekly, 'N/A' as total) as ep_fees,
    STRUCT(ws.tickets_sold_str as weekly, 'N/A' as total) as tickets_sold,
    STRUCT(ws.avg_ticket_price_formatted as weekly, 'N/A' as total) as avg_ticket_price
  ) as summary,
  ARRAY_AGG(
    STRUCT(
      ss.price_category as name,
      SUM(CAST(ss.inventory_str AS INT64)) as inventory,
      ss.sell_through_formatted as sell_through,
      ARRAY_AGG(STRUCT(ss.section as name, ss.inventory_str as inventory, ss.sold_str as sold)) as sections
    )
  ) as section_performance
FROM weekly_summary ws
CROSS JOIN section_summary ss
GROUP BY ws.report_date, ws.client_revenue_formatted, ws.ep_fees_formatted,
         ws.tickets_sold_str, ws.avg_ticket_price_formatted
```

**Note:** BigQuery EXPORT JSON structure may not match template exactly - may need post-processing.

**2. Simpler Alternative: Python Export Script (more control):**

```python
# extra_point/scripts/export_weekly_email_data.py
from google.cloud import bigquery
from google.cloud import storage
import json
from datetime import datetime, timedelta

def export_weekly_email_data(week_end_date: str, event_name: str = "Rose Bowl 2025"):
    client = bigquery.Client(project="turing-nature-473514-m6")

    # Query revenue summary
    revenue_query = f"""
    SELECT
      client_revenue,
      ep_fees,
      tickets_sold,
      average_ticket_price
    FROM `extrapoint_marts.rpt_revenue_weekly`
    WHERE order_week_end_date = '{week_end_date}'
    """
    revenue_result = client.query(revenue_query).to_dataframe().iloc[0]

    # Query section performance
    section_query = f"""
    SELECT
      price_category,
      section,
      inventory,
      sold,
      sell_through_pct
    FROM `extrapoint_marts.rpt_section_performance`
    WHERE order_week_end_date = '{week_end_date}'
    ORDER BY price_category, section
    """
    section_result = client.query(section_query).to_dataframe()

    # Transform to JSON structure matching template
    data = {
        "event_name": event_name,
        "report_date": datetime.strptime(week_end_date, '%Y-%m-%d').strftime('%B %d, %Y'),
        "summary": {
            "rb_revenue": {
                "weekly": f"${revenue_result['client_revenue']:,.0f}",
                "total": "N/A"
            },
            "ep_fees": {
                "weekly": f"${revenue_result['ep_fees']:,.0f}",
                "total": "N/A"
            },
            "tickets_sold": {
                "weekly": str(int(revenue_result['tickets_sold'])),
                "total": "N/A"
            },
            "avg_ticket_price": {
                "weekly": f"${revenue_result['average_ticket_price']:,.0f}",
                "total": "N/A"
            }
        },
        "section_performance": {
            "categories": []
        }
    }

    # Group sections by category
    for category in section_result['price_category'].unique():
        category_data = section_result[section_result['price_category'] == category]

        category_obj = {
            "name": category,
            "inventory": str(category_data['inventory'].sum()),
            "sell_through": f"{category_data['sell_through_pct'].mean() * 100:.1f}%",
            "sections": [
                {
                    "name": row['section'],
                    "inventory": str(row['inventory']),
                    "sold": str(row['sold'])
                }
                for _, row in category_data.iterrows()
            ]
        }
        data['section_performance']['categories'].append(category_obj)

    # Upload to Cloud Storage
    storage_client = storage.Client()
    bucket = storage_client.bucket('extrapoint-reports')
    blob = bucket.blob(f'weekly_email_data/{week_end_date}.json')
    blob.upload_from_string(json.dumps(data, indent=2))

    print(f"Exported data to gs://extrapoint-reports/weekly_email_data/{week_end_date}.json")

if __name__ == "__main__":
    # Run weekly via Cloud Scheduler
    week_end_date = (datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d')
    export_weekly_email_data(week_end_date)
```

**3. Cloud Function reads from bucket:**

```javascript
import { Storage } from '@google-cloud/storage';
import { build } from '@maizzle/framework';

const storage = new Storage();

export async function generateEmail(req, res) {
  const { weekEndDate } = req.body;

  // Read pre-generated JSON from bucket
  const [fileContents] = await storage
    .bucket('extrapoint-reports')
    .file(`weekly_email_data/${weekEndDate}.json`)
    .download();

  const weeklyData = JSON.parse(fileContents.toString());

  // Build email with Maizzle
  const { html } = await build({
    maizzle: { weekly: weeklyData },
    templates: ['emails/weekly-report.html']
  });

  res.status(200).send(html);
}
```

**4. Schedule Export with Cloud Scheduler:**

```bash
gcloud scheduler jobs create http export-weekly-email-data \
  --schedule="0 9 * * 1" \  # Every Monday at 9 AM
  --uri="https://us-central1-turing-nature-473514-m6.cloudfunctions.net/export-weekly-data" \
  --http-method=POST \
  --message-body='{"week_end_date": "2025-11-14"}'
```

### Pros

✅ **Faster Cloud Function:** No BigQuery SDK, just read JSON from bucket
✅ **Smaller Bundle:** Only `@google-cloud/storage` (~1-2 MB vs 5-10 MB)
✅ **Faster Cold Start:** 1-2 seconds vs 3-5 seconds
✅ **Decoupled Data Pipeline:** BigQuery export runs separately from email generation
✅ **Cached Data:** Can generate multiple emails from same data without re-querying
✅ **Lower Query Cost:** Export once, read many times
✅ **Easier Testing:** JSON files can be downloaded and tested locally

### Cons

❌ **Data Staleness:** JSON is snapshot from last export (not real-time)
❌ **Two-Step Process:** Must export before generating email
❌ **Additional Infrastructure:** Cloud Scheduler job + Storage bucket + export script
❌ **Storage Costs:** Small but adds GCS storage charges
❌ **Scheduling Complexity:** Must coordinate export timing with email generation
❌ **Less Flexible:** Harder to parameterize queries (event, date range filters)

---

## Trade-Off Analysis for Cloud Functions

### Performance Comparison

| Metric | Option A (Direct) | Option B (Bucket) |
|--------|------------------|------------------|
| **Cold Start** | 3-5 seconds | 1-2 seconds |
| **Warm Invocation** | 500-800ms | 200-400ms |
| **Bundle Size** | 10-15 MB | 3-5 MB |
| **Data Latency** | Real-time | Last export (stale) |
| **Query Cost** | $0.005 per invocation | $0.05 per export (amortized) |
| **Storage Cost** | $0 | $0.026 per GB/month |

### Cost Analysis (1000 emails/month)

**Option A:**
- BigQuery queries: 1000 × $0.005 = $5/month
- Cloud Function invocations: 1000 × $0.0000004 = negligible
- **Total: ~$5/month**

**Option B:**
- BigQuery export: 1 weekly job × 4 = $0.20/month
- Storage: ~10 KB × 52 weeks × $0.026/GB = negligible
- Cloud Function invocations: same as above
- **Total: ~$0.20/month**

**Winner: Option B is 25× cheaper at scale**

### Development Complexity

**Option A (Direct):**
- ⚠️ Must handle BigQuery client auth
- ⚠️ Must write query logic in Node.js
- ⚠️ Must handle query errors, timeouts, retries
- ⚠️ Must test query transformations
- ✅ Single codebase (all Node.js)
- ✅ Easier to iterate on queries during development

**Option B (Bucket):**
- ✅ Simpler Cloud Function (just read file)
- ✅ Python export script is cleaner for data transformations
- ⚠️ Must coordinate two separate deployments
- ⚠️ Must set up Cloud Scheduler
- ⚠️ Two codebases (Python + Node.js)
- ⚠️ More infrastructure to maintain

### Operational Considerations

**Option A:**
- Monitoring: Track BigQuery query latency, errors, quota
- Scaling: Each function invocation queries BigQuery
- Debugging: Check Cloud Function logs + BigQuery job history
- Dependencies: Keep `@google-cloud/bigquery` updated

**Option B:**
- Monitoring: Track export job success, file writes, read latency
- Scaling: Export once, read many times (better for high volume)
- Debugging: Check export logs + Cloud Function logs separately
- Dependencies: Keep Python BigQuery client + Node Storage client updated

---

## Recommendation

### For MVP / Low Volume (< 100 emails/week)
**Choose Option A (Direct BigQuery SDK)**

**Rationale:**
- Simpler to start: one codebase, no scheduling
- Real-time data: always query latest
- Easier development iteration: test queries directly
- Cold start penalty is acceptable for low volume
- $5/month cost is negligible

### For Production / High Volume (> 100 emails/week)
**Choose Option B (Export to Bucket)**

**Rationale:**
- 25× lower cost at scale
- Faster Cloud Function (1-2s vs 3-5s cold start)
- Decoupled pipeline: export failures don't block emails
- Better for batch email generation (e.g., 100 clients at once)
- Cached data: generate variations without re-querying

### Hybrid Approach (Best of Both)
**Start with Option A, migrate to Option B when needed**

1. **Phase 2a:** Implement Option A for immediate functionality
2. **Phase 2b:** Add Option B export script alongside
3. **Phase 2c:** Switch Cloud Function to read from bucket
4. Keep Option A code for ad-hoc queries/debugging

---

## BigQuery Schema Enhancements

### Critical: Add Missing Fields

**1. Add `event_name` column:**
```sql
ALTER TABLE `extrapoint_marts.rpt_revenue_weekly`
ADD COLUMN event_name STRING;

ALTER TABLE `extrapoint_marts.rpt_section_performance`
ADD COLUMN event_name STRING;
```

**2. Add cumulative columns for "all-time" metrics:**
```sql
ALTER TABLE `extrapoint_marts.rpt_revenue_weekly`
ADD COLUMN cumulative_client_revenue FLOAT,
ADD COLUMN cumulative_ep_fees FLOAT,
ADD COLUMN cumulative_tickets_sold INTEGER,
ADD COLUMN cumulative_avg_ticket_price FLOAT;
```

Update logic to calculate running totals when inserting weekly data.

### Optional: Add Pre-Formatted Columns

To reduce transformation logic in code:

```sql
ALTER TABLE `extrapoint_marts.rpt_revenue_weekly`
ADD COLUMN report_date_formatted STRING;  -- "November 14, 2025"

ALTER TABLE `extrapoint_marts.rpt_section_performance`
ADD COLUMN sell_through_formatted STRING;  -- "14.2%"
```

### Recommended: Create Unified View

```sql
CREATE OR REPLACE VIEW `extrapoint_marts.vw_weekly_email_data` AS
SELECT
  r.event_name,
  FORMAT_DATE('%B %e, %Y', DATE(r.order_week_end_date)) as report_date,
  STRUCT(
    STRUCT(
      FORMAT('$%\'.0f', r.client_revenue) as weekly,
      FORMAT('$%\'.0f', r.cumulative_client_revenue) as total
    ) as rb_revenue,
    STRUCT(
      FORMAT('$%\'.0f', r.ep_fees) as weekly,
      FORMAT('$%\'.0f', r.cumulative_ep_fees) as total
    ) as ep_fees,
    STRUCT(
      CAST(r.tickets_sold AS STRING) as weekly,
      CAST(r.cumulative_tickets_sold AS STRING) as total
    ) as tickets_sold,
    STRUCT(
      FORMAT('$%\'.0f', r.average_ticket_price) as weekly,
      FORMAT('$%\'.0f', r.cumulative_avg_ticket_price) as total
    ) as avg_ticket_price
  ) as summary,
  ARRAY_AGG(
    STRUCT(
      s.price_category as category_name,
      s.section as section_name,
      CAST(s.inventory AS STRING) as inventory,
      CAST(s.sold AS STRING) as sold,
      FORMAT('%.1f%%', s.sell_through_pct * 100) as sell_through
    )
  ) as sections
FROM `extrapoint_marts.rpt_revenue_weekly` r
JOIN `extrapoint_marts.rpt_section_performance` s
  ON r.order_week_end_date = s.order_week_end_date
  AND r.event_name = s.event_name
WHERE r.order_week_end_date = @week_end_date
  AND r.event_name = @event_name
GROUP BY r.event_name, r.order_week_end_date, r.client_revenue,
         r.ep_fees, r.tickets_sold, r.average_ticket_price,
         r.cumulative_client_revenue, r.cumulative_ep_fees,
         r.cumulative_tickets_sold, r.cumulative_avg_ticket_price;
```

**Benefits:**
- Single query returns all needed data
- Pre-formatted strings reduce Node.js code
- Type-safe STRUCT matches template expectations
- Less transformation code = fewer bugs

---

## Implementation Phases

### Phase 2a: Direct BigQuery Integration (Recommended Start)

**Week 1-2: Core Integration**
1. Add `@google-cloud/bigquery` dependency
2. Rewrite `scripts/load-data.js` with BigQuery client
3. Implement data transformation functions
4. Add environment variables for project ID, dataset names
5. Test locally with service account credentials

**Week 3: Cloud Function Deployment**
1. Create Cloud Function with HTTP trigger
2. Configure service account with BigQuery read permissions
3. Deploy function with sufficient memory (512 MB recommended)
4. Test with sample requests
5. Add error handling and logging

**Week 4: Schema Updates**
1. Add `event_name` column to BigQuery tables
2. Add cumulative columns for "all-time" metrics
3. Update ETL pipeline to populate new columns
4. Backfill historical data if needed

### Phase 2b: Export Pipeline (Optional Optimization)

**Week 1: Python Export Script**
1. Create `extra_point/scripts/export_weekly_email_data.py`
2. Implement BigQuery → JSON transformation
3. Upload JSON to Cloud Storage bucket
4. Test locally with sample week

**Week 2: Cloud Scheduler**
1. Deploy export script as Cloud Function or Cloud Run job
2. Create Cloud Scheduler job for weekly execution
3. Set up monitoring and alerts
4. Test end-to-end pipeline

**Week 3: Update Email Function**
1. Add `@google-cloud/storage` dependency (remove BigQuery)
2. Update Cloud Function to read from bucket
3. Deploy and test
4. Monitor performance improvements

---

## Testing Strategy

### Unit Tests
- Data transformation functions (currency formatting, grouping)
- Date formatting logic
- Error handling for missing data

### Integration Tests
- BigQuery queries return expected structure
- Bucket reads work with actual GCS
- Maizzle builds successfully with transformed data

### End-to-End Tests
1. **Local Development:**
   ```bash
   # Set test data in BigQuery
   # Run: pnpm dev
   # Verify email renders correctly
   ```

2. **Cloud Function:**
   ```bash
   curl -X POST https://your-function-url \
     -H "Content-Type: application/json" \
     -d '{"eventName": "Rose Bowl 2025", "weekEndDate": "2025-11-14"}' \
     > output.html

   # Open output.html in browser to verify
   ```

3. **Email Client Testing:**
   - Send test emails to Gmail, Outlook, Apple Mail
   - Verify rendering across devices (desktop, mobile)
   - Check accessibility (screen readers, contrast)

---

## Success Metrics

- ✅ Email generates successfully from BigQuery data
- ✅ All template fields populated correctly
- ✅ Currency and percentage formatting accurate
- ✅ Section grouping by category works
- ✅ Cloud Function cold start < 5 seconds (Option A) or < 2 seconds (Option B)
- ✅ Cloud Function warm latency < 800ms (Option A) or < 400ms (Option B)
- ✅ Error rate < 1%
- ✅ Cost per email < $0.01 (Option A) or < $0.001 (Option B)

---

## Next Steps

1. **Decide:** Choose Option A (direct) or Option B (bucket) based on volume expectations
2. **Schema First:** Add `event_name` and cumulative columns to BigQuery
3. **Implement:** Follow Phase 2a implementation plan
4. **Test:** Verify data accuracy with manual spot checks
5. **Deploy:** Start with Cloud Function in development project
6. **Monitor:** Track performance, errors, costs
7. **Optimize:** Consider migration to Option B if volume increases

---

## Open Questions

1. **Event Name Storage:** Where should `event_name` come from?
   - Add to BigQuery tables? (recommended)
   - Pass as parameter to Cloud Function?
   - Lookup table mapping week → event?

2. **All-Time Metrics:** How critical are cumulative "total" values?
   - Add to BigQuery schema? (cleanest)
   - Calculate on-demand with multi-week query? (complex)
   - Drop from template and show weekly-only? (simplest MVP)

3. **Multiple Events:** Will there be multiple events per week?
   - If yes, need event_id or event_name as query parameter
   - If no, can assume single event per week

4. **Historical Reports:** Need to regenerate old reports?
   - If yes, must keep historical JSON files or query historical BigQuery data
   - If no, can overwrite bucket files weekly

5. **Error Handling:** What if BigQuery returns no data?
   - Show "No data available" message in email?
   - Return HTTP 404 from Cloud Function?
   - Send alert to monitoring system?

---

**Document Version:** 1.0
**Created:** 2025-11-21
**Author:** Claude (via DDD Phase 1)
