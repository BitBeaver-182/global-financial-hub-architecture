# Currency Model

Support arbitrary user-defined currencies.

A currency has stable ID, code, name, symbol, decimal scale, status, and optional ISO metadata.

Every monetary fact preserves native amount and currency. Reporting currency is derived.

FX observations identify base, quote, rate, effective date/time, source, and quality/status.

Missing FX is explicit unavailable/null; never rate 1.

A real currency exchange is an economic event and is distinct from display conversion.

Historical FX selection must eventually be an explicit policy: transaction date, valuation date, period average, period end, or another documented rule. Do not mix policies silently.
