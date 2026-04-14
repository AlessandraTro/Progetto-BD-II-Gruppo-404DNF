# Progetto-BD-II-Gruppo-404DNF
Motore di ricerca intelligente per cataloghi e-commerce con autocomplete, ranking e filtri dinamici


## Avvio del progetto

---

## Requisiti

Prima di iniziare, assicurarsi di avere:

* Docker Desktop installato e avviato
* Git installato
* Python 3 installato
* `curl` disponibile da terminale

---

# 1. Preparazione del dataset

Prima di utilizzare OpenSearch è necessario generare il dataset nel formato corretto.

Il processo avviene in più step sequenziali.

---

## Step 0: Pulizia preliminare del dataset principale

Questo step serve a ripulire il dataset di partenza prima delle trasformazioni successive.

### macOS / Linux

```bash
python3 AggiustaDataset.py
```

### Windows PowerShell

```powershell
python AggiustaDataset.py
```

Output:

```text
D1_corretto.txt
```

---

## Step 1: Pulizia del catalogo

### macOS / Linux

```bash
python3 pulizia_catalogo.py
```

### Windows PowerShell

```powershell
python pulizia_catalogo.py
```

Output:

```text
catalogo_pulito.json
```

---

## Step 2: Arricchimento multilingua

### macOS / Linux

```bash
python3 dataset_inglese.py
```

### Windows PowerShell

```powershell
python dataset_inglese.py
```

Output:

```text
catalogo_multilingua.json
```

---

## Step 3: Conversione in NDJSON

### macOS / Linux

```bash
python3 conversione.py
```

### Windows PowerShell

```powershell
python conversione.py
```

Output finale:

```text
catalogo_multilingua.ndjson
```

---

## Pipeline completa dataset

```text
dataset principale
        ↓
AggiustaDataset.py
        ↓
D1_corretto.txt
        ↓
pulizia_catalogo.py
        ↓
catalogo_pulito.json
        ↓
dataset_inglese.py
        ↓
catalogo_multilingua.json
        ↓
conversione.py
        ↓
catalogo_multilingua.ndjson
```

---

# 2. Clonare la repository

### macOS / Linux

```bash
git clone <URL_REPOSITORY>
cd Progetto-BD-II-Gruppo-404DNF
```

### Windows PowerShell

```powershell
git clone <URL_REPOSITORY>
cd Progetto-BD-II-Gruppo-404DNF
```

---

# 3. Avviare Docker

### macOS / Linux

```bash
docker compose up -d --build
docker ps
```

### Windows PowerShell

```powershell
docker compose up -d --build
docker ps
```

Verificare che OpenSearch sia attivo sulla porta `9200`.

---

# 4. Preparare il dataset

Spostare il file:

```text
catalogo_multilingua.ndjson
```

nella cartella del progetto clonato.

---

# 5. Creare i chunk

## macOS / Linux

```bash
mkdir -p bulk_chunks
split -l 20000 -d -a 3 catalogo_multilingua.ndjson bulk_chunks/chunk_
for f in bulk_chunks/chunk_*; do mv "$f" "$f.ndjson"; done
```

Verifica:

```bash
ls bulk_chunks
```

---

## Windows PowerShell

```powershell
python -c "import os; input_file='catalogo_multilingua.ndjson'; output_dir='bulk_chunks'; chunk_size=20000; os.makedirs(output_dir, exist_ok=True); f=open(input_file,'r',encoding='utf-8'); chunk=[]; index=0
for i,line in enumerate(f,1):
    chunk.append(line)
    if i % chunk_size == 0:
        open(f'{output_dir}/chunk_{index:03}.ndjson','w',encoding='utf-8').writelines(chunk)
        chunk=[]
        index+=1
if chunk:
    open(f'{output_dir}/chunk_{index:03}.ndjson','w',encoding='utf-8').writelines(chunk)
f.close()"
```

Verifica:

```powershell
Get-ChildItem .\bulk_chunks
```

---

# 6. Creare l’indice OpenSearch

## macOS / Linux

```bash
curl -X PUT "http://localhost:9200/catalogo_prodotti" \
  -H "Content-Type: application/json" \
  --data-binary "@mapping.json"
```

## Windows PowerShell

```powershell
curl.exe -X PUT "http://localhost:9200/catalogo_prodotti" -H "Content-Type: application/json" --data-binary "@mapping.json"
```

---

# 7. Caricare i dati

## macOS / Linux

```bash
for f in bulk_chunks/*.ndjson; do
  echo "Carico $f"
  curl -X POST "http://localhost:9200/_bulk?refresh=true" \
    -H "Content-Type: application/x-ndjson" \
    --data-binary @"$f"
  echo ""
done
```

## Windows PowerShell

```powershell
Get-ChildItem .\bulk_chunks\*.ndjson | ForEach-Object {
    Write-Host "Importo $($_.Name)..."
    curl.exe -s -H "Content-Type: application/x-ndjson" -X POST "http://localhost:9200/_bulk?refresh=true" --data-binary "@$($_.FullName)"
    Write-Host ""
}
```

---

# 8. Verifica caricamento

## macOS / Linux

```bash
curl "http://localhost:9200/catalogo_prodotti/_count"
```

## Windows PowerShell

```powershell
curl.exe "http://localhost:9200/catalogo_prodotti/_count"
```

Valore atteso:

```text
1121455
```

---

# 9. Test della ricerca

## macOS / Linux

```bash
curl -X POST "http://localhost:9200/catalogo_prodotti/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 5,
    "query": {
      "multi_match": {
        "query": "cavo prolunga nero",
        "fields": [
          "titolo_interno^5",
          "titolo_seo^4",
          "descrizione_seo^3",
          "descrizione_completa^2",
          "tag^2",
          "brand^2",
          "categoria",
          "ean"
        ],
        "fuzziness": "AUTO"
      }
    }
  }'
```

---

## Windows PowerShell

Creare `query.json`:

```json
{
  "size": 5,
  "query": {
    "multi_match": {
      "query": "cavo prolunga nero",
      "fields": [
        "titolo_interno^5",
        "titolo_seo^4",
        "descrizione_seo^3",
        "descrizione_completa^2",
        "tag^2",
        "brand^2",
        "categoria",
        "ean"
      ],
      "fuzziness": "AUTO"
    }
  }
}
```

Eseguire:

```powershell
curl.exe -X POST "http://localhost:9200/catalogo_prodotti/_search" -H "Content-Type: application/json" --data-binary "@query.json"
```

---

# 10. Reset completo (se necessario)

Eliminare indice:

## macOS / Linux

```bash
curl -X DELETE "http://localhost:9200/catalogo_prodotti"
```

## Windows PowerShell

```powershell
curl.exe -X DELETE "http://localhost:9200/catalogo_prodotti"
```

Ritornare al punto 5

---

# 11. Avviare il Back-end

Avviare il progetto backend dall’IDE tramite **Run**.

---

# 12. Avviare il Front-end

## macOS / Linux

```bash
cd front-end
python3 -m http.server 5500
```

## Windows PowerShell

```powershell
cd .\front-end
python -m http.server 5500
```

---

# 13. Aprire il progetto

Aprire nel browser:

```text
http://localhost:5500
```

---

# Ordine corretto

1. Pulizia dataset iniziale
2. Pulizia catalogo
3. Arricchimento multilingua
4. Conversione NDJSON
5. Avvio Docker
6. Creazione chunk
7. Creazione indice
8. Upload dati
9. Verifica `_count`
10. Test ricerca
11. Avvio backend
12. Avvio frontend


