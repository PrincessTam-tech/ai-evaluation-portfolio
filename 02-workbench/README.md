
Evaluation pipeline for a fictional fintech support assistant (NovaPay).

Layers:
  1. Component  - intent recognition (predicted vs labelled intent)
  2. Trajectory - retrieval (retrieved KB articles vs labelled sources of truth)
  3. End-to-end - output scoring against required facts and the word register

Then: regression check (v1 vs v2), agreement with human labels (Cohen's kappa),
and a small reward model trained on human preferences (Bradley-Terry).

All data is synthetic. Run:  python eval_pipeline.py
"""
import json
import math
import re

DATA = json.load(open("data.json", encoding="utf-8"))
FACTS = {f["code"]: f for f in DATA["dictionary"]["facts"]}
REGISTER = {r["tag"]: r for r in DATA["register"]}
RANK = {"Fail": 0, "Partial": 1, "Pass": 2}


def find(patterns, text):
    """Return the first matching snippet for any pattern, or None."""
    for p in patterns:
        m = re.search(p, text, flags=re.IGNORECASE)
        if m:
            return m.group(0)
    return None


def evaluate(case, text, predicted_intent, retrieved):
    # Layer 3: required facts
    found = {code: bool(find(FACTS[code]["patterns"], text)) for code in case["facts"]}
    coverage = sum(found.values()) / len(found)

    # Layer 3: failure tags from the word register, plus case-specific traps
    tags = []
    for tag, rule in REGISTER.items():
        hit = find(rule["patterns"], text)
        if hit:
            tags.append({"tag": tag, "severity": rule["severity"], "evidence": hit})
    for trap in case["traps"]:
        hit = find([trap["pattern"]], text)
        if hit:
            tags.append({"tag": trap["tag"], "severity": REGISTER[trap["tag"]]["severity"], "evidence": hit})

    # Score
    if any(t["severity"] == "high" for t in tags) or coverage < 0.5:
        score = "Fail"
    elif tags or coverage < 1:
        score = "Partial"
    else:
        score = "Pass"

    # Layers 1 and 2
    intent_ok = predicted_intent == case["intent"]
    retrieval = len(set(retrieved) & set(case["kb"])) / len(case["kb"])

    missing = [c for c, ok in found.items() if not ok]
    return {"score": score, "coverage": coverage, "tags": tags, "missing": missing,
            "intent_ok": intent_ok, "retrieval": retrieval}


def explain(r):
    parts = [f"{t['tag']} ({t['severity']}): \"{t['evidence']}\"" for t in r["tags"]]
    if r["missing"]:
        parts.append("missing " + ", ".join(r["missing"]))
    if not r["intent_ok"]:
        parts.append("intent misclassified")
    if r["retrieval"] < 1:
        parts.append(f"retrieval {r['retrieval']:.0%} of labelled sources")
    return "; ".join(parts) or "All required facts present; no failure tags"


def next_action(r):
    if r["score"] == "Pass" and r["intent_ok"] and r["retrieval"] == 1:
        return "Keep as a passing anchor in the golden set"
    order = {"high": 0, "medium": 1}
    for t in sorted(r["tags"], key=lambda t: order[t["severity"]]):
        return REGISTER[t["tag"]]["action"]
    if not r["intent_ok"]:
        return "Add labelled examples of this intent to the classifier training set"
    if r["retrieval"] < 1:
        return "Check retrieval: a labelled source was not retrieved"
    return "Add the missing facts to the answer template or KB article"


def regression(a, b):
    if RANK[b["score"]] < RANK[a["score"]]:
        return f"REGRESSION: {a['score']} -> {b['score']}"
    if RANK[b["score"]] > RANK[a["score"]]:
        return f"Improved: {a['score']} -> {b['score']}"
    return f"No change: {a['score']}"


def cohen_kappa(x, y):
    labels = ["Pass", "Partial", "Fail"]
    n = len(x)
    po = sum(a == b for a, b in zip(x, y)) / n
    pe = sum((x.count(l) / n) * (y.count(l) / n) for l in labels)
    return 1.0 if pe == 1 else (po - pe) / (1 - pe)


# ---------- Reward model (Bradley-Terry on labelled features) ----------
FEATURES = list(REGISTER) + ["coverage"]


def featurise(r):
    v = [sum(t["tag"] == tag for t in r["tags"]) for tag in REGISTER]
    return v + [r["coverage"]]


def train_reward_model(pairs, epochs=300, lr=0.5, l2=0.01):
    """pairs: list of (features_chosen, features_rejected). Learns w so that
    reward(chosen) > reward(rejected): loss = -log sigmoid(w.(fc - fr))."""
    w = [0.0] * len(FEATURES)
    history = []
    for _ in range(epochs):
        grad = [0.0] * len(w)
        loss = 0.0
        for fc, fr in pairs:
            d = [a - b for a, b in zip(fc, fr)]
            z = sum(wi * di for wi, di in zip(w, d))
            p = 1 / (1 + math.exp(-z))
            loss += -math.log(max(p, 1e-12))
            for i, di in enumerate(d):
                grad[i] += -(1 - p) * di
        w = [wi - lr * (g / len(pairs) + l2 * wi) for wi, g in zip(w, grad)]
        history.append(loss / len(pairs))
    return w, history


if __name__ == "__main__":
    auto, human, pairs = [], [], []
    print(f"{'Case':5} {'v1':8} {'v2':8} Regression")
    for c in DATA["cases"]:
        res = {v: evaluate(c, o["text"], o["predicted_intent"], o["retrieved"]) for v, o in c["outputs"].items()}
        print(f"{c['id']:5} {res['v1']['score']:8} {res['v2']['score']:8} {regression(res['v1'], res['v2'])}")
        for v in ("v1", "v2"):
            auto.append(res[v]["score"])
            human.append(c["human"][v])
        chosen, rejected = ("v1", "v2") if RANK[c["human"]["v1"]] > RANK[c["human"]["v2"]] else ("v2", "v1")
        pairs.append((featurise(res[chosen]), featurise(res[rejected])))

    for v, idx in (("v1", 0), ("v2", 1)):
        scores = auto[idx::2]
        print(f"{v} pass rate: {scores.count('Pass')}/{len(scores)}")
    agree = sum(a == h for a, h in zip(auto, human)) / len(auto)
    print(f"Judge vs human agreement: {agree:.0%}, kappa {cohen_kappa(auto, human):.2f}")

    w, hist = train_reward_model(pairs)
    print(f"Reward model loss: {hist[0]:.3f} -> {hist[-1]:.3f}")
    for f, wi in sorted(zip(FEATURES, w), key=lambda x: x[1]):
        print(f"  {f:24} {wi:+.2f}")
