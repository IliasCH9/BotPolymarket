# Bot Polymarket CLOB — README Spec “Performance First”

> **Rôle visé**: bot quant/HFT-style orienté exécution, latence et robustesse, **sans promesse de gains garantis**.  
> **Disclaimer**: la performance réelle dépend fortement de la liquidité, de la concurrence, des frais, de la volatilité, de la qualité des signaux et de la discipline d’exécution.

## Références officielles Polymarket (obligatoires)

- CLOB intro: https://docs.polymarket.com/developers/CLOB/introduction
- Auth: https://docs.polymarket.com/developers/CLOB/authentication
- WebSocket overview: https://docs.polymarket.com/developers/CLOB/websocket/wss-overview
- Create order: https://docs.polymarket.com/developers/CLOB/orders/create-order
- Rate limits: https://docs.polymarket.com/api-reference/rate-limits
- Market makers intro: https://docs.polymarket.com/developers/market-makers/introduction

---

## 1) Stratégies “edge candidates” classées

### Candidat #1 (priorité haute): **Market Making agressif multi-marchés**

**Hypothèse d’edge**
- Exécuter plus vite et plus proprement que la moyenne sur un univers de marchés sélectionnés.
- Capturer le spread net de fees via:
  - meilleure latence décision/exécution,
  - meilleur positionnement au top-of-book,
  - contrôle d’inventaire dynamique.

**Pré-requis data/infra**
- Flux WebSocket CLOB stable (book/trades).
- Maintien d’un L2 local (best bid/ask + profondeur).
- Engine de requote faible latence et anti-churn.
- Instrumentation de latence tick→ack.

**Failure modes**
- Adverse selection (fills juste avant move défavorable).
- Churn excessif (annulations/replace trop fréquents).
- Mauvaise sélection de marchés (spread “piège”, profondeur faible).
- Saturation rate limits.

**Mesure de succès**
- Spread capture > 0 net des frais.
- Fill ratio stable, cancel rate maîtrisé.
- Time-at-top-of-book élevé.
- PnL net positif sur plusieurs régimes de volatilité.

**Concurrence attendue**: **élevée** (stratégie la plus “crowded”).

---

### Candidat #2: **Micro-alpha orderbook imbalance / flow**

**Hypothèse d’edge**
- Les signaux microstructure (imbalance, agressor flow, micro-momentum court) améliorent le skew et le spread adaptatif.
- Permet de réduire adverse selection et d’augmenter la qualité de fills.

**Pré-requis data/infra**
- Historique tick-level book + trades + propres fills.
- Calcul en temps réel:
  - Imbalance L1/L2,
  - Signed trade flow,
  - Volatilité court terme (EWMA).

**Failure modes**
- Signal decay rapide (alpha disparaît).
- Sur-optimisation (overfit backtest).
- Fausse confiance en périodes de news/shock.

**Mesure de succès**
- Hausse du spread capture vs baseline MM.
- Baisse de l’adverse selection à horizon Δ (ex: +3s, +10s).
- Amélioration PnL/turnover à risque d’inventaire constant.

**Concurrence attendue**: **moyenne à élevée**.

---

### Candidat #3: **Arbitrage / parité (si applicable)**

**Hypothèse d’edge**
- Exploiter incohérences nettes de frais/slippage:
  - Parité YES/NO,
  - Incohérences inter-marchés corrélés.

**Pré-requis data/infra**
- Mapping fiable des marchés corrélés.
- Exécution rapide multi-legs, gestion du leg risk.
- Vérification coûts totaux (fees + slippage + latence).

**Failure modes**
- Leg non rempli (risk d’exécution partielle).
- Corrélations qui cassent en stress.
- Coûts cachés > edge théorique.

**Mesure de succès**
- Taux de complétion de paniers > seuil cible.
- PnL net par opportunité positif et stable.
- Drawdown maîtrisé en épisodes de dislocation.

**Concurrence attendue**: **moyenne**, mais edge intermittent.

---

## 2) Sélection de marchés (Market Scorer)

### 2.1 Features (définition exacte)

Pour chaque marché `m` sur fenêtre glissante `W` (ex: 15 min):

- `vol_usd_W`: volume USD sur W.
- `trades_per_min_W`: fréquence de trades.
- `depth_xbps_W`: profondeur agrégée à ±X bps du mid (ex: X=20).
- `spread_bps_W`: spread moyen en bps.
- `mid_vol_W`: volatilité du mid (écart-type des returns 1s annualisé localement).
- `mid_stability_W`: 1 / (1 + nombre de jumps > J bps).
- `competition_proxy_W`: proxy compétition = ratio (quote updates / fills) + churn observé du top.
- `edge_proxy_W`: estimateur interne = spread capture attendu - adverse selection attendu - fee burden.

### 2.2 Score

Normaliser chaque feature en z-score robuste (median/IQR), clip [-3, +3].

```text
Score(m) =
  + 0.22 * Z(vol_usd_W)
  + 0.18 * Z(trades_per_min_W)
  + 0.20 * Z(depth_xbps_W)
  - 0.10 * Z(spread_bps_W trop large instable)
  - 0.12 * Z(mid_vol_W)
  + 0.08 * Z(mid_stability_W)
  - 0.10 * Z(competition_proxy_W)
  + 0.14 * Z(edge_proxy_W)
```

> Les poids sont un point de départ. À recalibrer via A/B + walk-forward.

### 2.3 Filtres d’exclusion

Exclure marchés si:
- `depth_xbps_W < depth_min` (ex: < 2 000 USD à ±20 bps).
- Prix collés aux bornes: `best_ask >= 0.99` ou `best_bid <= 0.01` (MM pur peu attractif).
- `trades_per_min_W < 0.5` (liquidité insuffisante).
- Événement imminent high-news-risk (si stratégie MM pure non directionnelle).
- Rate-limit budget insuffisant pour maintenir qualité de quote.

### 2.4 Rotation Top N + cooldown

- Recompute score toutes les `K=5` minutes.
- Maintenir `Top N` actifs (ex: N=20).
- **Cooldown**:
  - sortie d’un marché → lock 15 min avant réentrée,
  - entrée nouvelle → warm-up 2 min en taille réduite.
- Hysteresis:
  - entrer si rank <= N,
  - sortir seulement si rank > N + H (ex: H=5) pour éviter le flip-flop.

---

## 3) Architecture basse latence

### Stack recommandée

- **Option 1 (performance max)**: Rust.
- **Option 2 (productivité/perf équilibrée)**: TypeScript Node.js.
- Python acceptable pour R&D/backtest, moins optimale pour le path critique.

### 3.1 Market Data (WS)

- Connexion WebSocket CLOB unique + multiplex channels.
- Gestion reconnect avec backoff exponentiel (jitter).
- Après reconnect:
  1. récupérer snapshot book,
  2. rejouer updates bufferisés si possible,
  3. resynchroniser seq/orderbook.
- Maintenir L2 local: best bid/ask, depth tiers, microprice.

### 3.2 Strategy Engine

- Tick handler évènementiel (book/trade/fill).
- Calcul par tick:
  - `mid`,
  - `spread_target`,
  - `skew`,
  - `size`.
- SLA interne: décision < 5–20 ms (p95 selon charge).

### 3.3 Execution Engine

- Queue ordres priorisée (cancel > replace > new selon contexte).
- Cancel/replace intelligent (éviter churn inutile).
- Idempotence stricte:
  - `client_order_id` unique,
  - table de correspondance local->exchange,
  - re-check `open orders` après timeout ACK.
- Throttle strict conforme rate limits officiels (pas de contournement).

### 3.4 State Store

- In-memory pour hot path.
- Persistance légère (SQLite/Redis):
  - open orders,
  - positions,
  - inventory,
  - derniers mids,
  - dernier état machine.
- Recovery boot: restore + reconcile avec exchange.

### 3.5 Observability

- Logs JSON structurés (trace_id, order_id, market_id, latency buckets).
- Metrics Prometheus:
  - latence p50/p95/p99,
  - fill/cancel,
  - drawdown,
  - time-at-top.
- Dashboards + alerting (disconnect storm, error burst, stuck orders).

---

## 4) Pricing & exécution (agressif mais cohérent)

### 4.1 Pricing

- `mid = (best_bid + best_ask)/2`
- `spread_target = base_spread * vol_adj * flow_adj * competition_adj`

Règles:
- Serrer spread si profondeur forte + faible vol + adverse selection basse.
- Élargir spread si vol monte, flow agressif unilatéral, ou shock.

### 4.2 Inventory skew

```text
skew = k_inv * inv_norm + k_flow * signed_flow + k_vol * vol_regime
```

- Si inventaire long: bid moins agressif, ask plus agressif (déstockage).
- Si inventaire short: inverse.
- Hard limits d’inventaire par marché + global.

### 4.3 Execution rules

Placer ordre seulement si:
- tick size valide,
- prix/size dans bornes,
- pas de doublon (`anti-dup fingerprint: market+side+price+size`).

Cancel/replace si:
- `|mid_new - mid_last_quote| > repricing_threshold`, ou
- quote non compétitive (perte top-of-book > timeout), ou
- signal de risque microstructure défavorable.

Limiter cancels/min:
- budget dynamique par marché,
- priorité aux quotes avec expectancy la plus forte.

---

## 5) Pipeline d’expérimentation & optimisation

### 5.1 Data collection

- Capturer en continu:
  - snapshots + deltas book,
  - trades stream,
  - ordres envoyés/acks/rejets,
  - fills + positions.
- Horodatage monotonic + exchange timestamp.

### 5.2 Replay/backtest microstructure

- Rejouer flux tick-level.
- Simuler fills probabilistes selon position dans queue + pression de flux.
- Mesurer adverse selection:

```text
AS(Δ) = sign(side) * (mid_{t+Δ} - fill_price)
```

(positif favorable, négatif défavorable, selon convention choisie).

### 5.3 A/B tests

Paramètres testés:
- `spread_target`,
- refresh/requote interval,
- size ladder,
- gains de skew,
- seuils Market Scorer.

### 5.4 Optimisation

- Grid search initiale, puis Bayesian optimization.
- Walk-forward (train/validation/test temporel).
- Garde-fous anti-overfit:
  - pénalité complexité,
  - stabilité cross-régimes,
  - stress windows (news/volatile).

### 5.5 Critères Go prod

- Robustesse sur régimes calme/volatile/news.
- PnL net et drawdown dans seuils cibles.
- KPI exécution stables (latence, erreurs, fills).

---

## 6) Métriques obligatoires

### Per-market

- Spread capture (bps)
- Fill ratio
- Adverse selection: `mid_{t+Δ} - fill_price` (signé)
- Time-at-top-of-book
- Cancel rate

### Global

- PnL net (après fees)
- Sharpe
- Max drawdown
- Turnover
- Fee ratio
- Latence p50/p95/p99 (tick→order ack)
- Uptime
- WS disconnect count
- API error rate

---

## 7) State machine détaillée

```text
INIT -> SYNC -> QUOTE -> REQUOTE -> PAUSE -> RECOVER
```

### INIT
- Charger config, secrets, limites, paramètres marchés.
- Initialiser store local + métriques.
- Transition vers SYNC si checks techniques OK.

### SYNC
- Ouvrir WS + subscriptions.
- Récupérer snapshot books.
- Reconcile open orders/positions via REST.
- Transition vers QUOTE quand sync complète et fraîche.

### QUOTE
- Calculer prix/size/skew.
- Poster quotes initiales si critères validés.
- Transition vers REQUOTE sur nouveaux ticks pertinents.
- Transition vers PAUSE sur anomalie technique (WS down, erreurs API répétées).

### REQUOTE
- Évaluer nécessité cancel/replace.
- Appliquer throttles + anti-dup + idempotence.
- Retour QUOTE si quotes alignées.
- PAUSE si breach de garde-fous techniques.

### PAUSE
- Stopper nouvelles quotes.
- Maintenir cancel safety si nécessaire.
- Démarrer diagnostics / timers.
- Transition RECOVER dès que préconditions techniques reviennent.

### RECOVER
- Reconnect + resync snapshot.
- Reconcile ordres orphelins.
- Vérifier cohérence inventaire/état.
- Retour SYNC puis QUOTE.

---

## 8) Appels API / WS à utiliser (selon docs Polymarket)

> Utiliser strictement les endpoints et schémas du portail docs ci-dessus. Les noms ci-dessous reflètent la structure CLOB officielle; vérifier versions/champs exacts avant implémentation.

### REST CLOB

- **Authentication** (signature + headers API): voir guide `CLOB/authentication`.
- **Create order**: endpoint “Create Order” (POST) du CLOB.
- **Cancel order(s)**: endpoint(s) CLOB de cancel.
- **Get open orders**: endpoint de listing ordres ouverts.
- **Get markets / books / prices**: endpoints de consultation marchés et carnets.
- **Rate limits**: appliquer plafonds documentés `api-reference/rate-limits`.

### WebSocket CLOB

- Connexion au WS CLOB officiel (`wss-overview`).
- Souscriptions principales:
  - market/orderbook updates,
  - trades,
  - user channel (ordres/fills) si disponible dans votre setup auth.
- Gérer heartbeat/ping, reconnexion, resync snapshot.

---

## 9) Plan d’exécution ordres (idempotence incluse)

1. Générer décision quote (`price,size,side`).
2. Valider contraintes (tick, min size, inventory, budget rate-limit).
3. Vérifier duplicata (fingerprint actif).
4. Envoyer ordre (client_order_id unique).
5. Attendre ACK jusqu’à `ack_timeout_ms`.
6. Si timeout:
   - query open orders,
   - si ordre présent → marquer confirmé,
   - sinon retry borné avec même intention idempotente.
7. Sur fill partiel/total:
   - MAJ inventaire,
   - recalcul skew,
   - potentielle requote opposée.
8. Sur cancel/replace:
   - séquencer pour éviter cross état incohérent.

---

## 10) Paramètres par défaut (orientés performance, agressifs mais raisonnables)

| Paramètre | Défaut v1 | Commentaire |
|---|---:|---|
| `top_n_markets` | 20 | Univers actif simultané |
| `score_recompute_sec` | 300 | Rotation toutes 5 min |
| `market_cooldown_sec` | 900 | Anti-churn de sélection |
| `quote_refresh_ms` | 250 | Réactivité élevée |
| `repricing_threshold_bps` | 2.0 | Requote si mid bouge significativement |
| `base_spread_bps` | 12 | Point de départ MM agressif |
| `min_spread_bps` | 6 | Ne pas sous-coter le risque |
| `max_spread_bps` | 40 | Protection régime stress |
| `order_size_usd` | 50 | Taille initiale prudente |
| `max_inventory_usd_per_market` | 400 | Cap inventaire marché |
| `max_inventory_usd_global` | 3000 | Cap inventaire total |
| `max_cancels_per_min_per_market` | 30 | Protection rate-limit/churn |
| `ack_timeout_ms` | 800 | Timeout ACK réaliste |
| `ws_reconnect_backoff_ms` | 250→8000 | Exponentiel + jitter |
| `pause_on_api_error_rate` | >5%/1min | Tech safety |
| `pause_on_ws_disconnects` | >3/5min | Tech safety |

> Ces valeurs sont des points de départ; elles doivent être recalibrées par marché et par régime.

---

## 11) Go-live checklist “performance”

### Avant réel

- WS stable 24h en continu.
- Resync snapshot validée (pas de divergence book locale).
- Idempotence validée (timeouts ACK + retry/reconcile).
- Recovery crash testée (redémarrage + état cohérent).
- Dashboards + alertes opérationnels.
- Respect démontré des rate limits officiels.

### Ramp-up réel

1. Démarrer 1 marché (taille minimale).
2. Passer à 5 marchés si KPI stables.
3. Passer à 20 marchés avec montée progressive de taille.
4. Activer stop auto sur anomalies techniques:
   - spike erreurs API,
   - instabilité WS,
   - divergence état ordres.

---

## 12) Roadmap v1 / v2 / v3

### v1 — MM agressif (baseline production)
- Market Scorer + MM multi-marchés + inventory skew simple.
- Exécution idempotente + observabilité complète.
- Objectif: qualité exécution + robustesse prod.

### v2 — + flow signals
- Ajout imbalance/flow/micro-momentum.
- Spread/skew adaptatifs par régime.
- Objectif: réduire adverse selection, améliorer spread capture.

### v3 — + arbitrage/parité
- Détection incohérences YES/NO et corrélations.
- Moteur d’exécution multi-legs + contrôle leg risk.
- Objectif: sources d’edge complémentaires non corrélées au MM pur.

---

## 13) Cadre de performance (rappel)

Mesurer en continu:
- PnL net, Sharpe, max drawdown, turnover,
- Fill ratio, adverse selection,
- Latence tick→decision→send→ack,
- Quote quality (time-at-top, spread capture, slippage),
- Robustesse prod (uptime, recovery, erreurs).

**Principe final**: priorité à la qualité d’exécution, à la sélection de marchés et à la discipline d’itération empirique. Aucun setup ne garantit un profit constant.
