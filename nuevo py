"""
Agente de monitoreo de tendencias (prototipo del integrador).

Cada ejecución:
  1. Lee fuentes.json. Hay dos tipos de fuente:
       - "busqueda": noticias recientes de Google News para una consulta.
       - "pagina": la sección de noticias o publicaciones de un sitio. El agente toma los
         enlaces de artículos, entra a cada uno y extrae título, fecha y resumen.
  2. Descarta URLs ya vistas (data/estado.json) y notas con más de 30 días.
  3. Clasifica cada nota con un modelo de lenguaje, indicándole las tendencias semilla
     para que use sus nombres cuando corresponda.
  4. Mantiene las tendencias semilla (tendencias_semilla.json): cada una queda como
     confirmada, en observación, o vencida si no tiene notas nuevas en 90 días.
  5. Escribe data/noticias.json (lo que lee el dashboard) y data/registro.csv (la planilla).

Uso:
  export ANTHROPIC_API_KEY=...   # clave propia; nunca en el código
  python agente_tendencias.py

Requisitos: Python 3.9+, solo biblioteca estándar.
"""
import csv, datetime as dt, html, json, os, re, sys, time, urllib.error, urllib.parse, urllib.request
import xml.etree.ElementTree as ET
from email.utils import parsedate_to_datetime

HERE = os.path.dirname(os.path.abspath(__file__))
DATA = os.path.join(HERE, "data")
FUENTES = os.path.join(HERE, "fuentes.json")
SEMILLAS = os.path.join(HERE, "tendencias_semilla.json")
NOTICIAS = os.path.join(DATA, "noticias.json")
REGISTRO = os.path.join(DATA, "registro.csv")
ESTADO = os.path.join(DATA, "estado.json")
MODEL = os.environ.get("ANTHROPIC_MODEL", "claude-haiku-5-5")
MAX_DIAS = 30          # antigüedad máxima de una nota nueva
UMBRAL = 3             # fuentes no-señal para confirmar una tendencia
DIAS_VENCIDA = 90      # sin notas nuevas en este plazo: la tendencia se marca vencida
UA = {"User-Agent": "Mozilla/5.0 (monitor-tendencias/1.0)"}
COLUMNAS = ["fecha_busqueda", "titulo", "fuente", "tipo_fuente", "url", "fecha_publicacion",
            "tendencia", "horizonte", "brecha", "certeza", "estado", "observacion"]

PROMPT = """Sos analista de tendencias tecnológicas en e-commerce. Te paso el título y el resumen de una noticia.
Tendencias ya conocidas: {conocidas}.
Respondé SOLO con un objeto JSON, sin texto antes ni después, con estas claves:
- "es_tendencia": true si describe un cambio de tecnología o de mercado; false si es un anuncio aislado u opinión.
- "tendencia": si la noticia corresponde a una de las conocidas, usá ese nombre exacto. Si no, un nombre corto nuevo (máximo 8 palabras). "" si no aplica.
- "horizonte": "Corto", "Mediano" o "Largo" según cuándo podría volverse habitual; o "inferido" si la nota no lo dice.
- "brecha": "Alta", "Media" o "Sin especificar", respecto del promedio actual de la PyME de e-commerce.
- "certeza": "Alta" si hay dato medido, "Media" si es pronóstico u opinión de un directivo, "Baja" si es especulación.
- "observacion": una frase con la cifra o el hecho clave, sin agregar datos que no estén en el texto.

Título: {titulo}
Resumen: {resumen}
"""


def leer_json(ruta, por_defecto):
    if os.path.exists(ruta):
        with open(ruta, encoding="utf-8") as f:
            return json.load(f)
    return por_defecto


def pedir(url, timeout=20):
    req = urllib.request.Request(url, headers=UA)
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return r.read()


def limpiar(txt):
    return html.unescape(re.sub(r"<[^>]+>", " ", txt or "")).strip()


def recolectar_busqueda(f):
    url = "https://news.google.com/rss/search?" + urllib.parse.urlencode(
        {"q": f["consulta"], "hl": f.get("idioma", "es-419"),
         "gl": f.get("pais", "AR"), "ceid": f.get("ceid", "AR:es-419")})
    raiz = ET.fromstring(pedir(url))
    return [{"titulo": (it.findtext("title") or "").strip(),
             "url": (it.findtext("link") or "").strip(),
             "fecha": (it.findtext("pubDate") or "").strip(),
             "resumen": limpiar(it.findtext("description"))[:1200]}
            for it in raiz.iter("item")]


def leer_articulo(url):
    """Título, fecha y resumen de una página de artículo, desde sus metadatos."""
    try:
        doc = pedir(url).decode("utf-8", errors="replace")
    except Exception:
        return None
    def meta(*nombres):
        for n in nombres:
            m = re.search(r'<meta[^>]+(?:property|name)=["\']%s["\'][^>]+content=["\']([^"\']+)' % re.escape(n), doc, re.I)
            if m:
                return html.unescape(m.group(1)).strip()
        return ""
    titulo = meta("og:title") or limpiar((re.search(r"<title[^>]*>(.*?)</title>", doc, re.S | re.I) or [None, ""])[1])
    fecha = meta("article:published_time", "date", "pubdate")
    if not fecha:
        m = re.search(r'<time[^>]+datetime=["\']([^"\']+)', doc, re.I)
        fecha = m.group(1) if m else ""
    resumen = meta("og:description", "description")
    if not resumen:
        resumen = limpiar(re.sub(r"(?is)<(script|style).*?</\1>", " ", doc))[:800]
    return {"titulo": titulo, "url": url, "fecha": fecha, "resumen": resumen[:1200]}


def recolectar_pagina(f):
    """Toma los enlaces de la sección que coinciden con el patrón y lee cada artículo."""
    doc = pedir(f["url"]).decode("utf-8", errors="replace")
    patrones = f.get("patron", ["/blog/"])
    vistos, enlaces = set(), []
    for href in re.findall(r'<a[^>]+href=["\']([^"\']+)["\']', doc, re.I):
        absoluto = urllib.parse.urljoin(f["url"], href).split("#")[0]
        if absoluto in vistos or absoluto.rstrip("/") == f["url"].rstrip("/"):
            continue
        if not any(p in absoluto for p in patrones):
            continue
        vistos.add(absoluto)
        enlaces.append(absoluto)
        if len(enlaces) >= f.get("max", 8):
            break
    arts = [leer_articulo(u) for u in enlaces]
    return [a for a in arts if a and a["titulo"]]


def recolectar(fuentes):
    items = []
    for f in fuentes:
        try:
            if f.get("tipo") == "busqueda":
                crudos = recolectar_busqueda(f)
            elif f.get("tipo") == "pagina":
                crudos = recolectar_pagina(f)
            else:
                print(f"[aviso] tipo de fuente desconocido: {f.get('tipo')}", file=sys.stderr)
                continue
        except Exception as e:
            print(f"[aviso] no se pudo leer '{f['nombre']}': {e}", file=sys.stderr)
            continue
        for c in crudos:
            c["fuente"] = f["nombre"]
            c["tipo"] = f.get("tipo_fuente", "Especializada")
            items.append(c)
    return items


def fecha_dt(txt):
    """Convierte fechas RFC 2822 (Google News) o ISO (páginas). None si no se puede."""
    if not txt:
        return None
    try:
        return parsedate_to_datetime(txt)
    except Exception:
        pass
    try:
        return dt.datetime.fromisoformat(txt.replace("Z", "+00:00"))
    except Exception:
        return None


def es_reciente(txt):
    d = fecha_dt(txt)
    if d is None:
        return True  # sin fecha: se deja pasar y se revisa a mano
    if d.tzinfo is None:
        d = d.replace(tzinfo=dt.timezone.utc)
    return (dt.datetime.now(dt.timezone.utc) - d).days <= MAX_DIAS


def clasificar(item, conocidas, reintentos=3):
    clave = os.environ.get("ANTHROPIC_API_KEY")
    if not clave:
        raise SystemExit("Falta ANTHROPIC_API_KEY en el entorno.")
    cuerpo = {"model": MODEL, "max_tokens": 400,
              "messages": [{"role": "user", "content": PROMPT.format(
                  conocidas="; ".join(conocidas), titulo=item["titulo"], resumen=item["resumen"])}]}
    for intento in range(reintentos):
        req = urllib.request.Request("https://api.anthropic.com/v1/messages",
                                     data=json.dumps(cuerpo).encode("utf-8"),
                                     headers={"content-type": "application/json",
                                              "x-api-key": clave,
                                              "anthropic-version": "2023-06-01"})
        try:
            with urllib.request.urlopen(req, timeout=60) as r:
                texto = json.loads(r.read())["content"][0]["text"]
            return json.loads(texto[texto.index("{"): texto.rindex("}") + 1])
        except urllib.error.HTTPError as e:
            if e.code in (429, 500, 502, 503, 529) and intento < reintentos - 1:
                time.sleep(2 ** intento * 5)
                continue
            detalle = e.read().decode("utf-8", errors="replace")[:300]
            print(f"[aviso] error {e.code} en '{item['titulo'][:60]}': {detalle}", file=sys.stderr)
            return None
        except Exception as e:
            print(f"[aviso] clasificación fallida en '{item['titulo'][:60]}': {e}", file=sys.stderr)
            return None


def fila_semilla(s, hoy):
    return {"fecha_busqueda": hoy, "titulo": s["tendencia"], "fuente": s["fuente"],
            "tipo_fuente": "Analista", "url": s.get("url", ""), "fecha_publicacion": "",
            "tendencia": s["tendencia"], "horizonte": s["horizonte"], "brecha": s["brecha"],
            "certeza": "Media", "estado": "", "observacion": s.get("observacion", ""), "origen": "semilla"}


def estado_tendencia(filas_t, hoy):
    """Estado de una tendencia a partir de sus filas del agente."""
    agente = [r for r in filas_t if r["origen"] == "agente"]
    no_senal = {r["url"] for r in agente if r["tipo_fuente"] != "Señal"}
    if len(no_senal) >= UMBRAL:
        return "Confirmada"
    if agente:
        ultima = max(dt.date.fromisoformat(r["fecha_busqueda"]) for r in agente)
        if (hoy - ultima).days > DIAS_VENCIDA:
            return "Vencida"
        return "En observación"
    return "Semilla sin novedades"


def main():
    os.makedirs(DATA, exist_ok=True)
    hoy = dt.date.today()
    hoy_txt = hoy.isoformat()
    fuentes = leer_json(FUENTES, [])
    semillas = leer_json(SEMILLAS, [])
    if not fuentes:
        raise SystemExit("fuentes.json está vacío o no existe.")
    estado = leer_json(ESTADO, {"urls_vistas": []})
    vistas = set(estado["urls_vistas"])
    filas = leer_json(NOTICIAS, {"filas": []})["filas"]
    if not any(r["origen"] == "semilla" for r in filas):
        filas.extend(fila_semilla(s, hoy_txt) for s in semillas)

    conocidas = sorted({r["tendencia"] for r in filas if r["tendencia"]})
    nuevos = [i for i in recolectar(fuentes)
              if i["url"] and i["url"] not in vistas and es_reciente(i["fecha"])]
    print(f"Notas nuevas a clasificar: {len(nuevos)}")

    nuevas = 0
    fallos = 0
    en_esta_corrida = set()
    for item in nuevos:
        if item["url"] in vistas or item["url"] in en_esta_corrida:
            continue  # la misma nota puede venir de dos fuentes en una ejecución
        en_esta_corrida.add(item["url"])
        c = clasificar(item, conocidas)
        if c is None:
            # Falló la API: la nota NO se marca como vista, así se reintenta en la próxima ejecución
            fallos += 1
            continue
        vistas.add(item["url"])
        if not c.get("es_tendencia") or not c.get("tendencia"):
            continue
        filas.append({
            "fecha_busqueda": hoy_txt, "titulo": item["titulo"], "fuente": item["fuente"],
            "tipo_fuente": item["tipo"], "url": item["url"], "fecha_publicacion": item["fecha"],
            "tendencia": c["tendencia"], "horizonte": c.get("horizonte", "inferido"),
            "brecha": c.get("brecha", "Sin especificar"), "certeza": c.get("certeza", "Baja"),
            "estado": "", "observacion": c.get("observacion", ""), "origen": "agente",
        })
        if c["tendencia"] not in conocidas:
            conocidas.append(c["tendencia"])
        nuevas += 1

    # Estado por tendencia, copiado a cada fila para que el dashboard lo muestre
    por_t = {}
    for r in filas:
        if r["tendencia"]:
            por_t.setdefault(r["tendencia"], []).append(r)
    resumen = []
    for t, fs in por_t.items():
        est = estado_tendencia(fs, hoy)
        for r in fs:
            r["estado"] = est
        agente = [r for r in fs if r["origen"] == "agente"]
        resumen.append({
            "tendencia": t, "estado": est,
            "horizonte": max(set(r["horizonte"] for r in fs), key=lambda h: sum(1 for r in fs if r["horizonte"] == h)),
            "brecha": max(set(r["brecha"] for r in fs), key=lambda b: sum(1 for r in fs if r["brecha"] == b)),
            "fuentes_no_senal": len({r["url"] for r in agente if r["tipo_fuente"] != "Señal"}),
            "ultima_nota": max((r["fecha_busqueda"] for r in agente), default=""),
        })

    with open(REGISTRO, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=COLUMNAS, extrasaction="ignore")
        w.writeheader()
        w.writerows(filas)
    with open(NOTICIAS, "w", encoding="utf-8") as f:
        json.dump({"actualizado": dt.datetime.now().isoformat(timespec="minutes"),
                   "tendencias": resumen, "filas": filas}, f, ensure_ascii=False, indent=2)
    with open(ESTADO, "w", encoding="utf-8") as f:
        json.dump({"urls_vistas": sorted(vistas)}, f, ensure_ascii=False)

    conf = sum(1 for t in resumen if t["estado"] == "Confirmada")
    print(f"Notas nuevas como tendencia: {nuevas} · Tendencias: {len(resumen)} · "
          f"Confirmadas: {conf} · Vencidas: {sum(1 for t in resumen if t['estado'] == 'Vencida')}")
    if nuevos and fallos == len(nuevos):
        sys.exit("ERROR: ninguna nota pudo clasificarse. Revisar los avisos de la API arriba.")


if __name__ == "__main__":
    main()
