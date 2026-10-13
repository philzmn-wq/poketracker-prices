#!/usr/bin/env python3
"""PokeTracker Preis-Archiv: sammelt einmal täglich die Preise aller Pokémon-Sealed-Produkte.

- EU: Cardmarket (öffentliche Preisliste, alle Sprachen gemischt) – Preis-Trend, günstigstes Angebot, 7-Tage-Schnitt
- US: TCGplayer (über tcgcsv.com, englische Produkte) – Marktpreis, günstigstes Angebot

Ablage (für die App, über raw.githubusercontent.com abrufbar):
  data/eu/NN.json      {"<cardmarketId>":  [[JJJJMMTT, trend, low, avg7, avg30, avg1, avg], ...]}   NN = Id % 100
  data/us/NN.json      {"<tcgplayerId>":   [[JJJJMMTT, market, low], ...]}       Preise in US-Dollar
  data/map.json        {"<tcgplayerId>": <cardmarketId>}   gleiche Produkte über den Namen verbunden
  data/latest.json     Stand, Wechselkurs, Anzahl
Nur Standardbibliothek – läuft auf GitHub Actions ohne Installation.
"""
import json, os, re, sys, time, unicodedata, urllib.request
from datetime import datetime, timezone

DATA = os.path.join(os.path.dirname(os.path.abspath(__file__)), '..', 'data')
TODAY = int(datetime.now(timezone.utc).strftime('%Y%m%d'))
UA = {'User-Agent': 'PokeTracker-Preisarchiv/1.0 (privates Sammler-Projekt)'}


def get(url, tries=4):
    for i in range(tries):
        try:
            with urllib.request.urlopen(urllib.request.Request(url, headers=UA), timeout=90) as r:
                return json.load(r)
        except Exception as e:  # kurze Pause und erneut versuchen
            if i == tries - 1:
                raise
            time.sleep(2 + i * 3)


def norm(name):
    s = unicodedata.normalize('NFKD', name).encode('ascii', 'ignore').decode().lower()
    s = s.replace('&', ' and ')
    s = re.sub(r'[^a-z0-9]+', ' ', s)
    return ' '.join(s.split())


KIND = {'case', 'display', 'box', 'tin', 'bundle', 'booster', 'blister', 'collection', 'pack', 'sleeved', 'mini', 'premium', 'elite', 'trainer', 'build', 'battle', 'half', 'deck', 'combo', '18', '36', '10', '6', '3pack', '2pack'}
DROP = {'exclusive', 'the', 'pokemon', 'tcg', 'and'}


def key(name):
    """Vergleichsschlüssel: Wörter ohne Reihenfolge, TCGplayer- und Cardmarket-Schreibweisen angeglichen."""
    s = norm(name)
    s = s.replace('half booster box', 'booster box 18 boosters')
    s = re.sub(r'\bbooster pack\b', 'booster', s)
    s = re.sub(r'\b(\d+) pack\b', r'\1pack', s)            # "3 Pack Blister" == "3-Pack Blister"
    words = [w for w in s.split() if w not in DROP]
    if 'case' in words:                                      # "Booster Box Case" == "6 Booster Box Case"
        words = [w for w in words if not w.isdigit()]
    return frozenset(words)


def match_cm(name, cm_keys, by_first):
    k = key(name)
    if k in cm_keys:
        return cm_keys[k]
    # Ähnlichster Name mit denselben ersten zwei Wörtern (meist der Set-Name), nur wenn eindeutig und sehr ähnlich.
    first = ' '.join(norm(name).split()[:2])
    best, best_score, second = None, 0, 0
    for ck, cid in by_first.get(first, []):
        if (k ^ ck) & KIND:                                  # Produktart muss gleich sein (Case ≠ Box, Display ≠ Tin …)
            continue
        score = len(k & ck) / max(1, len(k | ck))
        if score > best_score:
            best, best_score, second = cid, score, best_score
        elif score > second:
            second = score
    return best if best_score >= 0.8 and best_score - second >= 0.15 else None


def load(path):
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        return {}


def save(path, obj):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    tmp = path + '.tmp'
    with open(tmp, 'w') as f:
        json.dump(obj, f, separators=(',', ':'), sort_keys=True)
    os.replace(tmp, path)


def r2(v):
    return None if v is None else round(float(v), 2)


def append(shards, region, pid, point):
    nn = '%02d' % (int(pid) % 100)
    key = (region, nn)
    if key not in shards:
        shards[key] = load(os.path.join(DATA, region, nn + '.json'))
    series = shards[key].setdefault(str(pid), [])
    if series and series[-1][0] == TODAY:
        series[-1] = point          # am selben Tag erneut gelaufen: überschreiben
    else:
        series.append(point)


def main():
    shards = {}

    # ---- EU: Cardmarket ----
    products = get('https://downloads.s3.cardmarket.com/productCatalog/productList/products_nonsingles_6.json')['products']
    guide = {g['idProduct']: g for g in get('https://downloads.s3.cardmarket.com/productCatalog/priceGuide/price_guide_6.json')['priceGuides']}
    cm_by_name = {}
    by_first = {}
    eu_count = 0
    for p in products:
        g = guide.get(p['idProduct'])
        if not g:
            continue
        trend, low, avg7 = r2(g.get('trend')), r2(g.get('low')), r2(g.get('avg7'))
        if trend is None and low is None:
            continue
        append(shards, 'eu', p['idProduct'], [TODAY, trend, low, avg7, r2(g.get('avg30')), r2(g.get('avg1')), r2(g.get('avg'))])
        eu_count += 1
        cm_by_name.setdefault(key(p['name']), p['idProduct'])
        by_first.setdefault(' '.join(norm(p['name']).split()[:2]), []).append((key(p['name']), p['idProduct']))

    # ---- US: TCGplayer über tcgcsv (eine Abfrage je Set-Gruppe und Datei) ----
    groups = get('https://tcgcsv.com/tcgplayer/3/groups')['results']
    mapping = load(os.path.join(DATA, 'map.json'))
    us_count = 0
    for i, grp in enumerate(groups):
        gid = grp['groupId']
        try:
            prods = get(f'https://tcgcsv.com/tcgplayer/3/{gid}/products')['results']
            prices = get(f'https://tcgcsv.com/tcgplayer/3/{gid}/prices')['results']
        except Exception as e:
            print('Gruppe', gid, 'übersprungen:', e, file=sys.stderr)
            continue
        sealed = {}
        for p in prods:
            if any(e.get('name') == 'Number' for e in (p.get('extendedData') or [])) or p['name'].startswith('Code Card'):
                continue
            sealed[p['productId']] = p['name']
        best = {}
        for pr in prices:
            pid = pr['productId']
            if pid not in sealed:
                continue
            market = pr.get('marketPrice') if pr.get('marketPrice') is not None else pr.get('midPrice')
            if market is None:
                continue
            if pid not in best or market > best[pid][0]:
                best[pid] = (market, pr.get('lowPrice'))
        for pid, (market, low) in best.items():
            append(shards, 'us', pid, [TODAY, r2(market), r2(low)])
            us_count += 1
            cm = match_cm(sealed[pid], cm_by_name, by_first)
            if cm:
                mapping[str(pid)] = cm
        time.sleep(0.15)           # tcgcsv schonen
        if i % 25 == 0:
            print(f'{i}/{len(groups)} Gruppen …', flush=True)

    for (region, nn), obj in shards.items():
        save(os.path.join(DATA, region, nn + '.json'), obj)
    save(os.path.join(DATA, 'map.json'), mapping)
    try:
        fx = get('https://api.frankfurter.dev/v1/latest?from=USD&to=EUR')['rates']['EUR']
    except Exception:
        fx = load(os.path.join(DATA, 'latest.json')).get('usdToEur', 0.9)
    latest = load(os.path.join(DATA, 'latest.json'))
    latest.update({'date': TODAY, 'usdToEur': fx, 'eu': eu_count, 'us': us_count, 'mapped': len(mapping)})
    latest.setdefault('since', TODAY)
    save(os.path.join(DATA, 'latest.json'), latest)
    print(f'Fertig: {eu_count} EU-Preise, {us_count} US-Preise, {len(mapping)} verbundene Produkte.')


if __name__ == '__main__':
    main()
