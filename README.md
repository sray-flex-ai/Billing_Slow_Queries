SELECT s.request_id, s.api_key, s.model, s.model_group,
       s.prompt_tokens, s.completion_tokens, s.total_tokens, s.call_type,
       CASE
           WHEN (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens') ~ '^[0-9]+$'
               THEN (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens')::bigint
           ELSE 0
       END AS cached_tokens,
       (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens') AS cached_tokens_raw,
       s."startTime" AS start_time,
       s."endTime" AS eligible_at
FROM "LiteLLM_SpendLogs" s
JOIN "LiteLLM_VerificationToken" vt ON vt.token = s.api_key
WHERE vt.key_name = ANY($1::text[])
  AND (s."endTime", s.request_id) > ($3, $4)
  AND s."endTime" < (NOW() AT TIME ZONE 'UTC') - ($5 || ' seconds')::interval
  AND s.request_id IS NOT NULL
  AND (s.total_tokens > 0 AND (COALESCE(s.completion_tokens, 0) > 0 OR s.call_type IN ('aembedding', 'arerank', 'embedding', 'rerank') OR (s.metadata -> 'usage_object' -> 'completion_tokens_details' ->> 'reasoning_tokens') ~ '^[0-9]*[1-9][0-9]*$'))
  AND COALESCE(s.metadata->>'tag', '') <> 'model_benchmark'
ORDER BY s."endTime" ASC, s.request_id ASC
LIMIT $2;

SELECT count(*) > 0
  FROM "LiteLLM_SpendLogs" s
 WHERE s."startTime" >= $1
   AND s."startTime" < $5
   AND s."endTime" < $2
   AND s.request_id IS NOT NULL
   AND (s.total_tokens > 0 AND (COALESCE(s.completion_tokens, 0) > 0 OR s.call_type IN ('aembedding', 'arerank', 'embedding', 'rerank') OR (s.metadata -> 'usage_object' -> 'completion_tokens_details' ->> 'reasoning_tokens') ~ '^[0-9]*[1-9][0-9]*$'))
   AND EXISTS (
       SELECT 1 FROM "LiteLLM_VerificationToken" vt
       WHERE vt.token = s.api_key AND vt.key_name = ANY($4::text[])
   )
   AND NOT EXISTS (SELECT 1 FROM unnest($3::text[]) a(id) WHERE a.id = s.request_id);

SELECT s.request_id, s.api_key, s.model, s.model_group,
       s.prompt_tokens, s.completion_tokens, s.total_tokens, s.call_type,
       CASE
           WHEN (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens') ~ '^[0-9]+$'
               THEN (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens')::bigint
           ELSE 0
       END AS cached_tokens,
       (s.response -> 'usage' -> 'prompt_tokens_details' ->> 'cached_tokens') AS cached_tokens_raw,
       s."startTime" AS start_time
FROM "LiteLLM_SpendLogs" s
JOIN "LiteLLM_VerificationToken" vt ON vt.token = s.api_key
WHERE vt.key_name = ANY($1::text[])
  AND s."startTime" >= $3
  AND s."startTime" < $5
  AND s."endTime" < $6
  AND s.request_id IS NOT NULL
  AND (s.total_tokens > 0 AND (COALESCE(s.completion_tokens, 0) > 0 OR s.call_type IN ('aembedding', 'arerank', 'embedding', 'rerank') OR (s.metadata -> 'usage_object' -> 'completion_tokens_details' ->> 'reasoning_tokens') ~ '^[0-9]*[1-9][0-9]*$'))
  AND COALESCE(s.metadata->>'tag', '') <> 'model_benchmark'
  AND NOT (s.request_id = ANY($4::text[]))
ORDER BY s."startTime" ASC
LIMIT $2;

SELECT uk.key_name
FROM user_api_keys uk
WHERE uk.org_id IS NOT NULL;

SELECT request_id FROM billing_sync_log
WHERE source = 'chat'
AND spend_log_time >= ($1::timestamp AT TIME ZONE 'UTC')
AND spend_log_time < ($2::timestamp AT TIME ZONE 'UTC')
AND request_id NOT LIKE 'gwagg:%';

SELECT MAX(spend_log_time) AT TIME ZONE 'UTC' AS max_billed FROM billing_sync_log
WHERE source = 'chat' AND request_id NOT LIKE 'gwagg:%';

SELECT spend_scan_cursor_time, spend_scan_cursor_request_id FROM billing_runtime_config WHERE id = TRUE;

UPDATE billing_runtime_config
SET spend_scan_cursor_time = $1,
    spend_scan_cursor_request_id = $2
WHERE id = TRUE;

SELECT count(*)
  FROM gw_finalize_manifests m, unnest(m.billing_ids) AS id
 WHERE m.minute >= now() - make_interval(secs => $2)
   AND m.minute <  now() - make_interval(secs => $1)
   AND NOT EXISTS (
       SELECT 1 FROM gateway_usage_events e
        WHERE e.event_day = gateway_uuid7_day(id) AND e.billing_id = id)
   AND NOT EXISTS (SELECT 1 FROM gateway_usage_events_quarantine q WHERE q.billing_id = id);

SELECT
    token,
    LOWER(metadata->>'user_email') AS email,
    NULLIF(metadata->>'user_name', '') AS name,
    key_name,
    team_id,
    created_at,
    updated_at,
    CASE
        WHEN expires IS NULL OR expires >= $1::timestamp THEN 'active'
        ELSE 'revoked'
    END AS status
FROM "LiteLLM_VerificationToken"
WHERE metadata->>'user_email' IS NOT NULL
  AND key_name IS NOT NULL
  AND (updated_at, token) > ($2::timestamp, $3) ORDER BY updated_at, token LIMIT $4;

SELECT
    token,
    LOWER(metadata->>'user_email') AS email,
    NULLIF(metadata->>'user_name', '') AS name,
    key_name,
    team_id,
    created_at,
    updated_at,
    CASE
        WHEN expires IS NULL OR expires >= $1::timestamp THEN 'active'
        ELSE 'revoked'
    END AS status
FROM "LiteLLM_VerificationToken"
WHERE metadata->>'user_email' IS NOT NULL
  AND key_name IS NOT NULL
  AND token > $2 ORDER BY token LIMIT $3;

SELECT
    token,
    LOWER(metadata->>'user_email') AS email,
    NULLIF(metadata->>'user_name', '') AS name,
    key_name,
    team_id,
    created_at,
    updated_at,
    CASE
        WHEN expires IS NULL OR expires >= $1::timestamp THEN 'active'
        ELSE 'revoked'
    END AS status
FROM "LiteLLM_VerificationToken"
WHERE metadata->>'user_email' IS NOT NULL
  AND key_name IS NOT NULL
  AND LOWER(metadata->>'user_email') = ANY($2::text[]) ORDER BY updated_at, token;

SELECT key_name, litellm_updated_at FROM user_api_keys WHERE key_name = ANY($1::text[]);

SELECT watermark, watermark_token, full_cursor FROM user_sync_state;

INSERT INTO user_sync_state (singleton, watermark, watermark_token, full_cursor) VALUES (TRUE, $1, $2, $3)
ON CONFLICT (singleton) DO UPDATE
    SET watermark = EXCLUDED.watermark, watermark_token = EXCLUDED.watermark_token,
        full_cursor = EXCLUDED.full_cursor;
