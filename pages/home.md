---
name: Home
assetId: 8b3715f0-a651-4e70-bdd7-9cc15e90828d
type: page
---

# Conversion Funnels

{% range_calendar id="date_filter" default_range="year to date" /%}

```sql funnel_steps
SELECT * FROM (
  SELECT 'Install' as step, 1 as step_order, count(DISTINCT user_pseudo_id) as users
  FROM events_clean WHERE event_name = 'first_open' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Register', 2, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'sign_up' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'KYC Complete', 3, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'verify_liveness_complete' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Checkout', 4, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'begin_checkout' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Purchase', 5, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'purchase' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Repeat Purchase', 6, (
    SELECT count(*) FROM (
      SELECT user_pseudo_id FROM events_clean
      WHERE event_name = 'purchase' AND toDate(event_datetime) {{date_filter.between}}
      GROUP BY user_pseudo_id HAVING count(*) >= 2
    )
  )
) ORDER BY step_order
```

## Overall Conversion Funnel

{% funnel_chart
    data="funnel_steps"
    category="step"
    value="users"
    title="Install → Repeat Purchase"
    show_percent=true
    value_fmt="#,##0"
    order="step_order"
    label_position="outside"
/%}

```sql conversion_rates
WITH params AS (
  SELECT
    {{date_filter.start}} as period_start,
    {{date_filter.end}} as period_end,
    dateDiff('day', {{date_filter.start}}, {{date_filter.end}}) + 1 as period_days
),
current_period AS (
  SELECT
    count(DISTINCT CASE WHEN event_name = 'first_open' THEN user_pseudo_id END) as installs,
    count(DISTINCT CASE WHEN event_name = 'sign_up' THEN user_pseudo_id END) as registrations,
    count(DISTINCT CASE WHEN event_name = 'verify_liveness_complete' THEN user_pseudo_id END) as kyc_complete,
    count(DISTINCT CASE WHEN event_name = 'begin_checkout' THEN user_pseudo_id END) as checkouts,
    count(DISTINCT CASE WHEN event_name = 'purchase' THEN user_pseudo_id END) as purchases,
    (SELECT count(*) FROM (
      SELECT user_pseudo_id FROM events_clean, params p
      WHERE event_name = 'purchase'
        AND toDate(event_datetime) >= p.period_start AND toDate(event_datetime) <= p.period_end
      GROUP BY user_pseudo_id HAVING count(*) >= 2
    )) as repeat_buyers
  FROM events_clean, params p
  WHERE toDate(event_datetime) >= p.period_start AND toDate(event_datetime) <= p.period_end
),
prior_period AS (
  SELECT
    count(DISTINCT CASE WHEN event_name = 'first_open' THEN user_pseudo_id END) as installs,
    count(DISTINCT CASE WHEN event_name = 'sign_up' THEN user_pseudo_id END) as registrations,
    count(DISTINCT CASE WHEN event_name = 'verify_liveness_complete' THEN user_pseudo_id END) as kyc_complete,
    count(DISTINCT CASE WHEN event_name = 'begin_checkout' THEN user_pseudo_id END) as checkouts,
    count(DISTINCT CASE WHEN event_name = 'purchase' THEN user_pseudo_id END) as purchases,
    (SELECT count(*) FROM (
      SELECT user_pseudo_id FROM events_clean, params p
      WHERE event_name = 'purchase'
        AND toDate(event_datetime) >= p.period_start - p.period_days
        AND toDate(event_datetime) < p.period_start
      GROUP BY user_pseudo_id HAVING count(*) >= 2
    )) as repeat_buyers
  FROM events_clean, params p
  WHERE toDate(event_datetime) >= p.period_start - p.period_days
    AND toDate(event_datetime) < p.period_start
)
SELECT
  c.registrations / nullIf(c.installs, 0) as registration_rate,
  pr.registrations / nullIf(pr.installs, 0) as prev_registration_rate,
  c.kyc_complete / nullIf(c.registrations, 0) as kyc_rate,
  pr.kyc_complete / nullIf(pr.registrations, 0) as prev_kyc_rate,
  c.checkouts / nullIf(c.kyc_complete, 0) as checkout_intent_rate,
  pr.checkouts / nullIf(pr.kyc_complete, 0) as prev_checkout_intent_rate,
  c.purchases / nullIf(c.checkouts, 0) as checkout_conversion_rate,
  pr.purchases / nullIf(pr.checkouts, 0) as prev_checkout_conversion_rate,
  c.repeat_buyers / nullIf(c.purchases, 0) as repeat_rate,
  pr.repeat_buyers / nullIf(pr.purchases, 0) as prev_repeat_rate
FROM current_period c, prior_period pr
```

## Step-by-Step Conversion Rates

{% big_value
    data="conversion_rates"
    value="registration_rate"
    title="Registration Rate"
    fmt="pct1"
    info="Install → Register. Distinct users with 'sign_up' event / distinct users with 'first_open' event."
    comparison={
        compare_vs="target"
        target="prev_registration_rate"
        text="vs prior period"
    }
/%}

{% big_value
    data="conversion_rates"
    value="kyc_rate"
    title="KYC Conversion Rate"
    fmt="pct1"
    info="Register → KYC Complete. Distinct users with 'verify_liveness_complete' event / distinct users with 'sign_up' event."
    comparison={
        compare_vs="target"
        target="prev_kyc_rate"
        text="vs prior period"
    }
/%}

{% big_value
    data="conversion_rates"
    value="checkout_intent_rate"
    title="Checkout Intent Rate"
    fmt="pct1"
    info="KYC Complete → Checkout. Distinct users with 'begin_checkout' event / distinct users with 'verify_liveness_complete' event."
    comparison={
        compare_vs="target"
        target="prev_checkout_intent_rate"
        text="vs prior period"
    }
/%}

{% big_value
    data="conversion_rates"
    value="checkout_conversion_rate"
    title="Checkout Conversion Rate"
    fmt="pct1"
    info="Checkout → Purchase. Distinct users with 'purchase' event / distinct users with 'begin_checkout' event."
    comparison={
        compare_vs="target"
        target="prev_checkout_conversion_rate"
        text="vs prior period"
    }
/%}

{% big_value
    data="conversion_rates"
    value="repeat_rate"
    title="Repeat Investment Rate"
    fmt="pct1"
    info="First Purchase → Repeat. Users with 2+ 'purchase' events / all users with at least 1 'purchase' event."
    comparison={
        compare_vs="target"
        target="prev_repeat_rate"
        text="vs prior period"
    }
/%}

```sql kyc_funnel
SELECT * FROM (
  SELECT 'Click Verify Identity' as step, 1 as step_order, count(DISTINCT user_pseudo_id) as users
  FROM events_clean WHERE event_name = 'click_verify_identity' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Verify Email', 2, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'verify_email' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Start Liveness', 3, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'start_verify_liveness' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Verify Identity', 4, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'verify_identity' AND toDate(event_datetime) {{date_filter.between}}
  UNION ALL
  SELECT 'Liveness Complete', 5, count(DISTINCT user_pseudo_id)
  FROM events_clean WHERE event_name = 'verify_liveness_complete' AND toDate(event_datetime) {{date_filter.between}}
) ORDER BY step_order
```

```sql kyc_rates
WITH params AS (
  SELECT
    {{date_filter.start}} as period_start,
    {{date_filter.end}} as period_end,
    dateDiff('day', {{date_filter.start}}, {{date_filter.end}}) + 1 as period_days
),
current_period AS (
  SELECT
    count(DISTINCT CASE WHEN event_name = 'click_verify_identity' THEN user_pseudo_id END) as click_verify,
    count(DISTINCT CASE WHEN event_name = 'verify_liveness_complete' THEN user_pseudo_id END) as liveness_complete
  FROM events_clean, params p
  WHERE toDate(event_datetime) >= p.period_start AND toDate(event_datetime) <= p.period_end
    AND event_name IN ('click_verify_identity', 'verify_liveness_complete')
),
prior_period AS (
  SELECT
    count(DISTINCT CASE WHEN event_name = 'click_verify_identity' THEN user_pseudo_id END) as click_verify,
    count(DISTINCT CASE WHEN event_name = 'verify_liveness_complete' THEN user_pseudo_id END) as liveness_complete
  FROM events_clean, params p
  WHERE toDate(event_datetime) >= p.period_start - p.period_days AND toDate(event_datetime) < p.period_start
    AND event_name IN ('click_verify_identity', 'verify_liveness_complete')
)
SELECT
  c.liveness_complete / nullIf(c.click_verify, 0) as kyc_completion_rate,
  pr.liveness_complete / nullIf(pr.click_verify, 0) as prev_kyc_completion_rate
FROM current_period c, prior_period pr
```

## KYC Verification Funnel

{% big_value
    data="kyc_rates"
    value="kyc_completion_rate"
    title="KYC Completion Rate"
    fmt="pct1"
    info="Click Verify Identity → Liveness Complete. Distinct users with 'verify_liveness_complete' event / distinct users with 'click_verify_identity' event."
    comparison={
        compare_vs="target"
        target="prev_kyc_completion_rate"
        text="vs prior period"
    }
/%}

{% funnel_chart
    data="kyc_funnel"
    category="step"
    value="users"
    title="KYC Verification Steps"
    show_percent=true
    value_fmt="#,##0"
    order="step_order"
    label_position="outside"
/%}
