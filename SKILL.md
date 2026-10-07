---
name: global-ai-act-transparency
description: Setzt projektübergreifend Dokumentations- und Transparenzpflichten bei maschinellem Lernen und Scoring-Algorithmen durch. Use when creating, changing, or refactoring scoring logic, risk classifications, ML model pipelines, or automated decision algorithms in any project.
triggers: Bei der Erstellung, Änderung oder dem Refactoring von Scoring-Logik, Risiko-Klassifizierungen, ML-Modell-Pipelines oder Entscheidungs-Algorithmen in jedem Projekt.
---

# AI Act & Algorithmische Transparenz

## Grundsatz

Jede automatisierte Entscheidung und jeder berechnete Score muss erklärbar, dokumentiert und anhand der Logs nachvollziehbar sein (EU AI Act, DSGVO Art. 22). Diese Regeln gelten in jedem Projekt und ohne Ausnahme, auch beim Refactoring bestehender Logik.

## Pflichtregeln

1. **Strukturierter Docstring ist Pflicht.** Jede Funktion, die automatisierte Entscheidungen trifft oder Scores berechnet, bekommt einen strukturierten Docstring. Das gilt auch für Hilfsfunktionen, die Gewichte, Schwellenwerte oder Klassen festlegen. Fehlt der Docstring in bestehendem Code, den ich ändere, ergänze ich ihn.

2. **Der Docstring enthält beide Tags:**
   - `@compliance-rationale`: Warum wird dieser Wert so berechnet? Nenne fachliche Begründung, Gewichte und Schwellenwerte.
   - `@bias-check`: Welche Annahmen wurden getroffen, die zu einer Verzerrung führen könnten? Dazu gehören Proxy-Variablen (z. B. PLZ, Alter, Name, Sprache), Trainingsdaten und Schwellenwerte.

   Ein leerer Tag, „n/a“ oder „keine“ ohne Begründung zählt nicht.

3. **Inputs werden deterministisch nachvollziehbar geloggt.** Pro Entscheidung logge ich alle Inputs, die Modell- bzw. Regelversion, Gewichte und Schwellenwerte, das Ergebnis und eine Korrelations-ID, sodass die Entscheidung reproduzierbar ist. Zufallsquellen bekommen einen festen, geloggten Seed. Personenbezogene Inputs werden dabei nur maskiert oder pseudonymisiert geloggt (**REQUIRED:** global-dsgvo-pii-guard).

4. **Blackbox-Warnung im Chat.** Wenn eine Logik eine Entscheidung trifft, die sich nicht durch eine einfache Variablen-Gewichtung erklären lässt, warne ich dich proaktiv im Chat. Das betrifft z. B. neuronale Netze, Gradient Boosting, Ensembles, LLM-basierte Entscheidungen und nicht dokumentierte Drittanbieter-Scores. Die Warnung nennt die Funktion, warum sie nicht erklärbar ist und mögliche Gegenmaßnahmen (z. B. SHAP/LIME, ein interpretierbares Surrogat-Modell, Human-in-the-Loop).

## Beispiel

```python
def credit_risk_score(applicant: Applicant, model_version: str = "rules-v3") -> float:
    """Berechnet den Kreditrisiko-Score (0–100).

    @compliance-rationale: Gewichtete Summe aus Zahlungshistorie (0.5),
        Einkommens-/Ausgabenquote (0.3) und Beschäftigungsdauer (0.2).
        Schwelle 60 = manuelle Prüfung, gemäß Kreditrichtlinie KR-2026.
    @bias-check: Beschäftigungsdauer benachteiligt Berufseinsteiger und
        Personen mit Erwerbsunterbrechungen (z. B. Elternzeit). PLZ wird
        bewusst nicht genutzt (Proxy für Herkunft).
    """
    score = 0.5 * applicant.payment_history + 0.3 * applicant.income_ratio + 0.2 * applicant.employment_years_norm
    # Deterministisches Audit-Log; PII nur maskiert (DSGVO-Compliance)
    audit_log.info("credit_risk_score", extra={
        "correlation_id": applicant.request_id, "model_version": model_version,
        "inputs": {"payment_history": applicant.payment_history, "income_ratio": applicant.income_ratio,
                   "employment_years_norm": applicant.employment_years_norm},
        "weights": [0.5, 0.3, 0.2], "result": score,
    })
    return score
```

## Red Flags: STOP

- Score-, Ranking- oder Klassifizierungsfunktion ohne `@compliance-rationale` oder `@bias-check`
- „Ist nur ein Refactoring“ oder „die Logik ändert sich nicht“
- Magic Numbers als Gewichte oder Schwellenwerte ohne Begründung
- `model.predict(...)` ohne Logging von Inputs und Modellversion
- Ein ML-Modell ersetzt eine Regel-Logik, ohne Blackbox-Warnung im Chat

**In all diesen Fällen: Docstring ergänzen, Audit-Logging einbauen und Blackbox-Logik im Chat melden.**
