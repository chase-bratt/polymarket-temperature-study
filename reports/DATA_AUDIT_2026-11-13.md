# Data audit 2026-11-13

Source: `data/raw/polybot_2026-11-13` -> `data/curated`, `data/ml`.

## Tables

| table | rows | columns | MB |
|---|---:|---:|---:|
| curated/book_full.parquet | 5,613,949 | 12 | 442.8 |
| curated/book_top3.parquet | 3,967,799 | 12 | 98.3 |
| curated/bot_cash.parquet | 1,361 | 4 | 0.0 |
| curated/bot_config_epoch.parquet | 12 | 5 | 0.0 |
| curated/bot_fill.parquet | 918 | 15 | 0.1 |
| curated/bot_order.parquet | 35,993 | 14 | 1.2 |
| curated/bot_position.parquet | 916 | 15 | 0.1 |
| curated/bot_resting_order.parquet | 5,420 | 18 | 0.2 |
| curated/bot_run.parquet | 1,547 | 7 | 0.0 |
| curated/bot_strategy_status.parquet | 1 | 4 | 0.0 |
| curated/bot_trade_review.parquet | 871 | 16 | 0.0 |
| curated/dim_event.parquet | 2,519 | 17 | 0.1 |
| curated/dim_market.parquet | 27,709 | 21 | 3.4 |
| curated/dim_station.parquet | 50 | 10 | 0.0 |
| curated/fc_event_ensemble.parquet | 225,891 | 31 | 19.5 |
| curated/fc_hourly_det.parquet | 8,001,792 | 6 | 8.5 |
| curated/fc_model_run.parquet | 3,030 | 4 | 0.0 |
| curated/fc_regional.parquet | 8,097,433 | 11 | 47.6 |
| curated/fc_vc_daily.parquet | 55,938 | 10 | 0.4 |
| curated/fc_vc_hourly.parquet | 1,343,458 | 9 | 4.7 |
| curated/kalshi_event.parquet | 1,159 | 4 | 0.0 |
| curated/kalshi_market.parquet | 442,665 | 14 | 3.3 |
| curated/label_event.parquet | 2,519 | 22 | 0.0 |
| curated/mkt_follow_snap.parquet | 1,099,241 | 16 | 3.0 |
| curated/mkt_intraday_snap.parquet | 105,050 | 14 | 1.5 |
| curated/mkt_meta.parquet | 2,810,049 | 5 | 22.7 |
| curated/mkt_price_history.parquet | 9,243,402 | 3 | 16.6 |
| curated/mkt_snapshot.parquet | 2,495,262 | 22 | 59.4 |
| curated/mkt_trade.parquet | 1,586,188 | 10 | 84.9 |
| curated/obs_metar.parquet | 48,183 | 28 | 1.3 |
| curated/obs_metar_hist.parquet | 10,767,421 | 23 | 94.8 |
| curated/ops_refresh_log.parquet | 600 | 3 | 0.0 |
| curated/wx_fc_daily.parquet | 375,000 | 7 | 1.0 |
| curated/wx_obs_daily.parquet | 38,433 | 10 | 0.1 |
| curated/wx_obs_daily_hist.parquet | 467,712 | 12 | 1.5 |
| curated/wx_obs_hourly.parquet | 933,527 | 6 | 6.5 |
| ml/dim_date.parquet | 9,817 | 9 | 0.0 |
| ml/market/blend_bucket_scan.parquet | 2,495,262 | 45 | 74.9 |
| ml/market/exec_resting_bid.parquet | 5,420 | 32 | 0.3 |
| ml/splits_v1.parquet | 9,817 | 3 | 0.1 |
| ml/temperature/temp_hist_fc.parquet | 1,990,510 | 52 | 70.2 |
| ml/temperature/temp_label_daily.parquet | 467,730 | 10 | 1.5 |
| ml/temperature/temp_live_ens.parquet | 225,211 | 51 | 24.0 |

## Unique keys

| table | key | duplicates | |
|---|---|---:|---|
| dim_event | event_id | 0 | PASS |
| dim_market | market_id | 0 | PASS |
| dim_station | icao | 0 | PASS |
| mkt_snapshot | scan_utc, market_id | 0 | PASS |
| mkt_trade | trade_hash | 0 | PASS |
| bot_order | order_id | 0 | PASS |
| bot_fill | fill_id | 0 | PASS |
| bot_resting_order | resting_id | 0 | PASS |
| bot_position | market_id, side, strategy | 0 | PASS |
| obs_metar | icao, obs_utc, raw | 0 | PASS |
| wx_obs_hourly | icao, obs_utc | 0 | PASS |
| wx_obs_daily | icao, date_local | 0 | PASS |
| wx_fc_daily | icao, target_date_local, model, lead_days | 0 | PASS |
| fc_event_ensemble | issue_utc, event_id | 0 | PASS |
| fc_hourly_det | icao, issue_utc, valid_utc, model | 0 | PASS |
| fc_regional | icao, issue_utc, valid_utc, model | 0 | PASS |
| label_event | event_id | 0 | PASS |
| mkt_price_history | market_id, price_utc | 0 | PASS |
| book_full | snap_utc, market_id, outcome_side | 0 | PASS |

## Integrity, coverage and leakage

| check | value | |
|---|---:|---|
| resolved events with != 1 YES bucket | 0 | PASS |
| METAR-rule, non-proxy events where METAR label disagrees with resolution | 0 | PASS |
| METAR-rule, non-proxy events checked | 1,829 | info |
| proxy / noaa / unknown-rule disagreements (expected: HK Observatory, Taipei, early rules) | 41 | info |
| observed temperatures outside -60..60 C | 0 | PASS |
| forecast temperatures outside -60..60 C | 0 | PASS |
| snapshot quotes outside 0..1 | 0 | PASS |
| crossed books in snapshots (yes_bid > yes_ask) | 0 | info |
| trades whose market is not in dim_market | 0 | info |
| trade prints inside the documented 11-02 22:10Z..11-04 03:31Z outage (backfilled by the v2 API?) | 47,285 | info |
| METAR reports whose obs time differs from the nominal hour | 13,009 | info |
| METAR reports without T-group (whole-degree only) | 40,322 | info |
| history vs live daily max differ by > 0.5 C on overlapping days | 3 | info |
| snapshot scan gaps > 2 h (count) | 3 | info |
| trade-print gaps > 3 h (count; includes the 11-02..11-04 API outage) | 0 | info |
| ensemble rows with 0 members (quota fallback; excluded from temp_live_ens) | 680 | info |
| LEAK temp_live_ens: rows issued after the target day ended | 0 | PASS |
| LEAK temp_hist_fc: persistence day not before the issue day | 0 | PASS |
| blend_bucket_scan rows scanned after the target day ended (same-day/settling scans) | 0 | info |
| blend_bucket_scan p_mkt_norm sums off by > 1e-6 | 0 | PASS |
| US (F) snapshots with fc_mean_mkt < 0 (unit double-conversion guard) | 0 | PASS |

## Null rates of model inputs and labels

| table.column | non-null | of |
|---|---:|---:|
| temperature/temp_hist_fc.fc_mean_c | 1,990,510 | 1,990,510 |
| temperature/temp_hist_fc.obs_lag_c | 1,933,438 | 1,990,510 |
| temperature/temp_hist_fc.err_mean_30d_c | 1,942,770 | 1,990,510 |
| temperature/temp_hist_fc.y_c | 1,982,170 | 1,990,510 |
| temperature/temp_live_ens.ens_mean_c | 225,211 | 225,211 |
| temperature/temp_live_ens.reg_tmax_c | 220,384 | 225,211 |
| temperature/temp_live_ens.reg_cloud_mean_pct | 220,384 | 225,211 |
| temperature/temp_live_ens.obs_last_temp_c | 225,189 | 225,211 |
| temperature/temp_live_ens.y_c | 198,978 | 225,211 |
| market/blend_bucket_scan.p_mkt_norm | 2,491,210 | 2,495,262 |
| market/blend_bucket_scan.bot_p_model | 2,495,262 | 2,495,262 |
| market/blend_bucket_scan.y_win | 2,294,963 | 2,495,262 |
| market/exec_resting_bid.yes_bid | 5,420 | 5,420 |
| market/exec_resting_bid.y_side_won | 4,901 | 5,420 |

**Result: all hard checks PASS**
