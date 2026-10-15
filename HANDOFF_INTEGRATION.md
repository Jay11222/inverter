# Integration handoff: PMSM inverter telemetry replay (for Mudhawi)

From: Jayan (data simulation). To: Mudhawi (inference service review, pipeline, deployment, IAIP integration, end-to-end validation, with Riyaz and the Software team).

The replay is a standalone microservice, separate from the Battery/Transformer replay already handed over. It publishes dataset rows as Cumulocity measurements and nothing else: it never calls `/predict`, never builds model requests and never stores predictions. The inference service repository was not modified.

## 1. What is published

One dataset row = one measurement (`type ElectricalAssets_PMSMInverterTelemetry`, fragment `PMSMInverter`), all series at one `time`, plus a `replayTrace` fragment. Values are the exact float64 values of `converted_dataset.csv`.

| Profile | Series | Use |
|---|---|---|
| `sensor` | `Ia_original`, `Ib_original`, `VDC`, `IDC_original`, `T1`, `T2`, `T3`, `VD` + `Ia_adc`, `Ib_adc`, `IDC_adc` | Represents the real device. The upstream feature pipeline must calculate the 10 engineered features from the 10 Hz stream. |
| `sensor_plus_features` | the 18 model features under their model names | Lets the inference path be tested before the upstream pipeline exists. |

Never published: `FDD` (reference label), `Timestamp` (relative index), partitions, predictions. Reference labels are in the run preview and `report.json`, keyed by `observationId` and `rowIndex`, so predictions can be compared afterwards.

Measurement times: 10 Hz spacing exactly (100 ms), UTC, anchored at `ANCHOR_TIME` (default `now`, resolved once per run). The send cadence (default 1 request per second) never changes measurement times, because the rate-of-change and moving-average features depend on the 0.1 s spacing.

## 2. From measurement to a /predict request

**sensor_plus_features:** take each of the 18 features from the series of the same name. If a series is missing from a measurement, it was omitted because its stored value is non-finite (see `replayTrace.omittedSeries`). For `Current_Imbalance`, `Ia_Normalized`, `Ib_Normalized` send `null` (the model imputes the training median). If any other feature is missing the row is not model-ready (only rows 0 to 8 of the file) and must not be sent.

**sensor:** upstream calculates the 10 features from the stream, in time order, per continuous run:

- `Ia_a = 0.074 * (Ia_adc - 512)`, same for `Ib_a`, `IDC_a` (**Arduino calibration; do not use `*_original` here**, the offset and scale differ: errors up to 216.7 W in `Power_AC`).
- `Power_DC = VDC * IDC_a`, `Power_AC = VD * (Ia_a + Ib_a)`, `Current_Imbalance = |Ia_a - Ib_a| / ((Ia_a + Ib_a) / 2)`, `Ia_Normalized = Ia_a / IDC_a`, `Ib_Normalized = Ib_a / IDC_a`, `Temp_Diff_Max = max(T1,T2,T3) - min(T1,T2,T3)`.
- `VDC_RateOfChange = (VDC[t] - VDC[t-1]) / 0.1`, `IDC_RateOfChange = (IDC_a[t] - IDC_a[t-1]) / 0.1`, `VDC_MovingAvg` / `IDC_MovingAvg` = mean of the current and previous 9 samples.
- Zero denominators give infinity; send `null` for the three ratio features. The first sample of a stream has no rate of change and the first 9 no moving average: do not send those.
- `app/feature_engineering.py` in the model repo is the executable reference. It takes all 8 raw ADC channels; with this profile, apply its formulas to the published `VDC`, `VD`, `T1..T3` and the Arduino-calibrated currents. This reproduces the stored features exactly (`FEATURE_REPRODUCIBILITY.md`).
- Each replay run starts a new stream: its first 9 observations are warm-up rows (listed in the preview). Where should the pipeline reset its history: per run, per device, on a time gap? (Open item 1.)

## 3. Samples

`docs/samples/sample_sensor_plus_features.json` and `docs/samples/sample_sensor.json` hold three dry-run measurements each (rows 3449, 5998, 6823 = reference cases ref-10, ref-03, ref-05, all held-out test rows), with the matching `/predict` body, the reference label and the model repo's expected class. Row 3449 shows the null case: `Current_Imbalance` is omitted from the measurement (`omittedSeries: "Current_Imbalance:+inf"`) and is `null` in the body.

Abridged (sensor_plus_features, row 5998, reference label F2, expected class F2):
```json
{"source": {"id": "<existing device id>"}, "time": "2026-10-15T00:00:00.000Z", "type": "ElectricalAssets_PMSMInverterTelemetry",
 "PMSMInverter": {"Ia_original": {"value": ..., "unit": "A"}, "...": "18 series", "Current_Imbalance": {"value": ...}},
 "replayTrace": {"observationId": "pmsm-134d1e0abbcd-r05998", "rowIndex": 5998, "sourceTimestamp": "599.8000000000001",
                 "timeOrigin": "shifted", "profile": "sensor_plus_features", "omittedSeries": "", "reviewFlags": "negative_VDC,negative_VD"}}
```

`tools/reference_case_bodies.py` builds the body for all 11 reference cases. All are valid. They differ from the request bodies in `samples/expected_predictions.json` by 1 to 6 ulps in 2 to 7 features, because the export used pandas' default CSV parser (Data Science item). Compare predicted classes, and probabilities with a tolerance, not byte equality.

## 4. Proposed dry-run selections (pending approval)

| Selection | Rows | Content |
|---|---|---|
| F4 full block | 6516 to 6856 (341) | 55 final-train, 58 final-validation, 68 held-out, 130 CV, 30 purge rows; first 9 rows cross the F3/F4 boundary |
| F0 to F1 transition | 4275 to 4314 (40) | 20 held-out F0 + 20 F1 development rows; the two recordings were made 548 s apart, so this is not a recorded fault onset |
| F2 held-out (optional) | 5885 to 6108 (224) | held-out only |
| F0 held-out with ref-10/11 (optional) | 3436 to 3475 (40) | held-out only; includes the null case |

Replaying non-held-out rows overstates accuracy (the model was trained on block 3 of each class only, 2,064 rows). The preview labels every row.

## 5. What the end-to-end test should check

1. The approved measurements arrive on the existing device with the expected `type`, fragment, series names, units and times (100 ms spacing, no future-time rejection), and no `FDD` or `Timestamp` series.
2. Count delivered = count in the run ledger; no duplicates by `replayTrace.observationId` after retries.
3. Upstream forms one valid `/predict` body per model-ready observation: 18 keys, `null` only for the three ratio features, no NaN or infinity tokens; rows listed as not model-ready are not sent.
4. With `sensor`: upstream features equal the dataset's stored features (compare with the preview's `expected_predict_body_from_dataset`; exact or within 1e-12), including at warm-up and after the history reset.
5. Predictions joined on `observationId` / `rowIndex` with the reference labels in `report.json`: per-class results on held-out rows close to the model's test metrics (macro F1 0.960 on the full held-out set); the 11 reference cases give the expected classes.
6. Ordering and rate: observations processed in time order at the agreed cadence; behaviour when the run is stopped mid-way.

## 6. What was and was not tested here

Tested: 97 automated tests (Python 3.11 and 3.13), the mutation check (all 10 deliberate breakages caught), real-data dry runs with the network disabled, the HTTP service started from a clean environment and called over HTTP, publishing against a local mock Cumulocity (no real tenant).
Not tested: any real tenant, the deployed model or its OpenAPI, the upstream pipeline, the Docker build, packaging. See `OPEN_ITEMS.md`.
