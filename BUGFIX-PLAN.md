# Bugfix & Förbättringsplan – Meta Analytics

> Läs denna plan NOGGRANT innan du gör något. Bekräfta att du förstått varje fas innan implementation.

## Identifierade problem

### P1 (Kritisk): Ingen plattformsseparation i UI
Appen blandar FB- och IG-data utan möjlighet att filtrera. Konton med samma namn (t.ex. "Barnradion Sveriges Radio") visas dubbelt utan indikation på vilken plattform de tillhör.

### P2 (Kritisk): Extrem lagg vid interaktion
Flera sekunders fördröjning vid klick på radioknappar/checkboxar. Orsak: dyra beräkningar körs synkront i `useEffect` utan memoization.

### P3 (Medel): Engagemangsmått blandas utan förklaring
FB-engagemang (interaktioner + klick) och IG-engagemang (interaktioner + saves + follows) visas i samma kolumn utan att användaren vet vilken formel som gäller.

### P4 (Medel): Plattformsbadge saknas i selektorer och trendanalys
Kontodropdowns och trendanalysens legend visar bara kontonamn, inte plattform.

---

## Fas 1: Plattformsfilter (löser P1)

### Vad som ska byggas
Lägg till ett **globalt plattformsfilter** i `MainView.jsx` (INTE separata flikar – det blir lättare att underhålla och möjliggör framtida korsplattformsvyer).

```
[Alla] [Facebook] [Instagram]    ← Segmented control / toggle-grupp högst upp
```

### Implementation

1. **Ny state i MainView:**
```javascript
const [platformFilter, setPlatformFilter] = useState('all'); // 'all' | 'facebook' | 'instagram'
```

2. **Filtrera data INNAN den skickas till child components:**
```javascript
const filteredData = useMemo(() => {
  if (!data || platformFilter === 'all') return data;
  return data.filter(post => post._platform === platformFilter);
}, [data, platformFilter]);
```

3. **Skicka `filteredData` (inte `data`) till AccountView, PostView, PostTypeView, TrendAnalysisView.**

4. **Rendera filtret som en segmented control ovanför flikarna:**
   - Visa antal inlägg per plattform i knapparna: `Facebook (2 450)` / `Instagram (1 230)`
   - Om bara en plattform finns i datan, visa inte filtret alls

5. **VIKTIGT: Gör INTE separata kodvägar per plattform i vykomponenterna.** Filtreringen sker centralt i MainView. Vyerna tar emot redan filtrerad data.

### Kontrollera efter implementation
- [ ] Kan man filtrera på bara FB? Bara IG? Båda?
- [ ] Visar filtret korrekt antal per plattform?
- [ ] Döljs filtret om det bara finns en plattform?

---

## Fas 2: Performance (löser P2)

### Princip: FLYTTA alla tunga beräkningar till `useMemo`, ta bort `useEffect` för beräkningar.

### AccountView.jsx – mest kritisk

**Problem:** `summarizeByAccount()` körs i en `useEffect` och sätter state med `setSummaryData()`. Det orsakar:
1. En render med gammal data
2. Beräkning
3. setState → ny render med ny data

**Fix:** Gör om till `useMemo`:
```javascript
// ERSÄTT useEffect + setSummaryData med:
const summaryData = useMemo(() => {
  if (!data || !selectedFields || selectedFields.length === 0) return [];
  return summarizeByAccount(data, selectedFields);
}, [data, selectedFields]);

const totalSummary = useMemo(() => {
  // ... beräkna totaler från summaryData
}, [summaryData, selectedFields, data]);
```

Ta bort `isLoading`-state – med `useMemo` behövs den inte.

### MainView.jsx – selectedFields-loop

**Problem:** `useEffect` på rad ~155 kör `setSelectedFields(prev => prev.filter(...))` vid varje `activeView`-byte, även om inget ändras. Det triggar en onödig re-render.

**Fix:**
```javascript
useEffect(() => {
  const availableFields = Object.keys(getAvailableFields());
  setSelectedFields(prev => {
    const filtered = prev.filter(field => availableFields.includes(field));
    // Undvik re-render om inget ändras
    if (filtered.length === prev.length && filtered.every((f, i) => f === prev[i])) {
      return prev; // Samma referens = ingen re-render
    }
    return filtered;
  });
}, [activeView]);
```

### PostTypeView.jsx

**Problem:** `aggregateByPostType()` körs i `useEffect`. Samma fix som AccountView – byt till `useMemo`.

### TrendAnalysisView.jsx

Redan `useMemo` – bra. Men kontrollera att `chartLines` inte omberäknas i onödan.

### Kontrollera efter implementation
- [ ] Klicka på radioknapp i Trendanalys – reagerar den omedelbart?
- [ ] Klicka på/av checkboxar i "Välj värden" – uppdateras tabellen utan lagg?
- [ ] Ladda 5000+ rader och testa ovanstående igen

---

## Fas 3: Plattformsbadge överallt (löser P4)

### Var badges ska läggas till

1. **Kontodropdowns** (PostView, PostTypeView): Visa `Kontonamn [FB]` eller `Kontonamn [IG]` i Select-options.

2. **TrendAnalysisView kontolista** (checkboxarna): Lägg till en liten badge efter kontonamnet.

3. **TrendAnalysisView legend** (under grafen): Visa `[FB]` eller `[IG]` efter kontonamnet.

4. **AccountView tabell**: Plattformsbadge visas redan via `PlatformBadge`-komponenten – kontrollera att den alltid syns, inte bara vid `hasMixedPlatforms`.

### Befintlig PlatformBadge-komponent

Det finns redan en `PlatformBadge` i AccountView.jsx och PostView.jsx. **Extrahera den till en delad komponent:**

```
src/renderer/components/ui/PlatformBadge.jsx
```

Använd den konsekvent i alla vyer.

### Kontrollera efter implementation
- [ ] Ser man plattform i varje kontodropdown?
- [ ] Visar trendanalysens legend plattform per konto?
- [ ] Om samma kontonamn finns på båda plattformarna – är de tydligt separerade?

---

## Fas 4: Engagemangsförklaring (löser P3)

### Vad som ska göras

1. **Lägg till en info-ikon (ℹ) bredvid "Engagemang"-kolumnrubriken** i AccountView och PostView som visar en tooltip:
   - Om filtret = Facebook: "Engagemang = reaktioner + kommentarer + delningar + klick"
   - Om filtret = Instagram: "Engagemang = gilla + kommentarer + delningar + sparade + följare"
   - Om filtret = Alla: "Engagemanget beräknas olika per plattform. FB: inkl. klick. IG: inkl. sparade & följare."

2. **`ENGAGEMENT_INFO` och `INTERACTIONS_INFO`** finns redan i `columnConfig.js` – använd dem.

3. **I PostView per-rad:** När plattformsfiltret = "Alla", visa en liten fotnot-ikon på engagemangscellen om formeln skiljer sig.

### Kontrollera efter implementation
- [ ] Syns info-ikonen bredvid "Engagemang" i kolumnrubriken?
- [ ] Visar tooltipens text rätt beroende på plattformsfilter?

---

## Ordning och regler

1. **Implementera i ordning: Fas 1 → 2 → 3 → 4**
2. **Testa efter varje fas** – bekräfta med mig innan nästa
3. **Ändra INTE datamodellen, storageService eller webDataProcessor** – problemen är i UI-lagret
4. **Extrahera PlatformBadge till delad komponent i Fas 3, inte innan**
5. **Skriv inga nya tester** – fokusera på att fixa buggarna
6. **Bryt inte befintlig export-funktionalitet** (CSV/Excel)
