# Weather API for Kazakhstan (Almaty / Astana) — 2026-11-08

Research agent output. VERIFIED = source read or live-probed today; ASSUMED = inference.

## 1. Open-Meteo

**Model coverage (VERIFIED by live probe, Almaty 43.238/76.945):**
- `icon_eu` → `{"error":true,"reason":"No data is available for this location"}`. ICON-EU domain is 25.45°W–63.98°E (VERIFIED https://www.dwd.de/EN/ourservices/nwp_forecast_data/nwp_forecast_data.html); Almaty ~77°E, Astana ~71.5°E are outside.
- `icon_global` (13 km), `icon_seamless`, `ecmwf_ifs025`, `ecmwf_ifs` (IFS HRES 9 km), `gfs_seamless`, `gem_seamless`, `meteofrance_seamless`, `ukmo_seamless` (10 km), `jma_seamless`, `best_match` all return data for Almaty.
- **best_match** outside regional coverage = ICON-global/GFS/IFS blend only (VERIFIED https://open-meteo.com/en/docs). For Almaty its grid point equals the `ecmwf_ifs` point, so IFS HRES likely dominates (ASSUMED).
- `dwd_icon_global/eu/d2` IDs marked "Discontinued 2026-11-30"; use `icon_seamless`/native IDs (VERIFIED https://open-meteo.com/en/docs/dwd-api).
- Previous-runs API (`_previous_day0..7`) and Ensemble API exist (VERIFIED https://open-meteo.com/en/docs/previous-runs-api).
- Free tier: 600/min, 5 000/h, 10 000/day, no key, non-commercial, CC-BY 4.0 (VERIFIED https://open-meteo.com/en/pricing).
- No Central Asia accuracy evaluations found (ASSUMED none public).

## 2. Yandex Weather API (Meteum), 2026

- Plans (VERIFIED https://yandex.ru/dev/weather/doc/en/concepts/pricing): "Test" free **7 days only**, one-time; "Optimal" 54 000 RUB/month; per-request from 0.024 RUB. Legacy v1 "on your site" 50 req/day informer effectively deprecated (ASSUMED).
- ToS (VERIFIED https://yandex.ru/legal/pogoda_b2b_offer/): offer for Russian-currency-law "Residents", "only for professional or business activity", weekly data refresh, deletion on termination, mandatory branding. Poor fit for a private individual in Kazakhstan (ASSUMED).
- Accuracy: Meteum 2.0 claims (self-reported) 40 % short-term error cut; weaker in sparsely populated areas (VERIFIED https://yandex.ru/pogoda/technology , https://habr.com/ru/articles/724686/). No independent comparison vs ECMWF for Almaty found.

## 3. Other candidates

| Provider | Free quota | Key/card | Notes |
|---|---|---|---|
| Kazhydromet | no API | — | HTML 7-day city pages (VERIFIED https://www.kazhydromet.kz/ru/weather/in_city_7_days/6/149 ); monthly observation DB |
| AccuWeather | 14-day trial only | key, then paid | no permanent free tier (VERIFIED https://developer.accuweather.com/faq ) |
| **Visual Crossing** | 30 000 records/month (~1 000/day), 15-day | key, no card mentioned (VERIFIED https://www.visualcrossing.com/weather-data-editions/ ) | good JSON |
| Weatherbit | 50/day, daily only | key | too thin |
| Tomorrow.io | 500/day, 25/h | key | field-counted |
| Meteoblue | 1-year free API, non-commercial | key | Almaty only global models (VERIFIED multimodel page) |
| WeatherAPI.com | 100 000/month, 3-day | key, no card | short horizon |
| OpenWeatherMap | One Call 3.0 1 000/day | **card required** | fails no-card criterion |

## 4. Independent benchmarks

None for Almaty/Astana (ForecastWatch regions don't name Central Asia; ForecastAdvisor coverage untested). Nearest academic: WRF Almaty winter-inversion paper, no inter-model comparison. ASSUMED: Almaty foothill inversions and Astana steppe winters are where 9–13 km global models err most (T2m under inversions, snow/rain phase).

## Recommendation

**Primary: Open-Meteo with explicit models**, one request: `models=ecmwf_ifs,ukmo_seamless,icon_seamless,gfs_seamless`; present `ecmwf_ifs` and use the spread as a confidence hint. No key, no card, 10 000/day, previous-runs endpoint for self-verification.
**Fallback: Visual Crossing** (key, ~1 000 records/day, 15-day JSON). Yandex only with a business entity and payment. Skip AccuWeather, Weatherbit, OWM.

## Two-week self-verification plan

1. Hourly: `https://aviationweather.gov/api/data/metar?ids=UAAA,UACC&format=json` (VERIFIED working) → store temp, dewpoint, wind, wx.
2. Every 6 h: Open-Meteo hourly T2m/precip/wind for UAAA 43.35/77.027 and UACC 51.024/71.468 for each model + best_match, and Visual Crossing.
3. Daily: `_previous_day1`/`_previous_day2` for 24/48 h lead scoring.
4. Per-model MAE/bias T2m, precip hit rate ≥0.1 mm, wind MAE; split Almaty nights vs days.
5. After 14 days pick lowest T2m MAE per city.

## Sources

open-meteo.com docs/ecmwf-api/dwd-api/previous-runs-api/pricing; dwd.de NWP data page; yandex.ru weather pricing, index-v3, pogoda_b2b_offer, pogoda/technology; habr 724686; kazhydromet.kz; developer.accuweather.com/faq; visualcrossing.com editions; weatherbit.io/pricing; tomorrow.io rate limits; meteoblue free API + Almaty multimodel; weatherapi.com/pricing; openweathermap.org/one-call-transfer; forecastadvisor.com; forecastwatch.com 2024 report; github open-meteo issue 851; aviationweather.gov/data/api.
