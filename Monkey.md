# Tampermonkey in Firefox installieren und Userscript einfügen

Diese kurze Anleitung zeigt, wie du Tampermonkey in Firefox installierst und ein eigenes Userscript einfügst.

## 1. Tampermonkey in Firefox installieren

1. Öffne Firefox mit Einstellungen und gehe auf Erweiterungen und Themes
2. Suche nach **Tampermonkey**.
3. Klicke auf **Zu Firefox hinzufügen**.
4. Bestätige die Installation mit **Hinzufügen**.
5. Nach der Installation erscheint das Tampermonkey-Symbol oben rechts in Firefox.

## 2. Neues Userscript in Tampermonkey einfügen

1. Klicke oben rechts in Firefox auf das **Tampermonkey-Symbol**.
2. Wähle **Dashboard**.
3. Klicke auf **Neues Script erstellen** oder **Add a new script**.
4. Lösche den vorhandenen Beispielcode im Editor.
5. Kopiere das folgende Userscript vollständig in den Editor.
6. Klicke auf **Datei > Speichern** oder drücke **Strg + S**.
7. Prüfe im Dashboard, ob das Script aktiviert ist.

## 3. Userscript:

```javascript

// ==UserScript==
// @name         Lagerverwaltung → Sciformation: Bestandsverkauf eintragen
// @namespace    lagerverwaltung
// @version      0.4.0
// @description  Holt die naechste zu uebertragende Ausgabe aus der Lagerverwaltung und fuellt sie halb-automatisch ins Sciformation-Verkaufsformular. Login + Bestaetigung macht der Nutzer manuell.
// @author       Lagerverwaltung
// @match https://fb08-ctest.org.chemie.uni-giessen.de/*
// @connect fb08oc-ggb.org.chemie.uni-giessen.de
// @grant        GM_xmlhttpRequest
// @grant        GM_addStyle
// @run-at       document-idle
// ==/UserScript==

/*
 * ────────────────────────────────────────────────────────────────────────────
 * KONFIGURATION — vor dem ersten Einsatz anpassen
 * ────────────────────────────────────────────────────────────────────────────
 *
 *  1. @match  oben: exakte URL der Sciformation-Verkaufseingabeseite
 *             (mit abschliessendem * fuer Query-/Hash-Anhaengsel).
 *  2. @connect oben: Host der Lagerverwaltung OHNE Schema, z.B.
 *             lager.chemie.uni-giessen.de  (erlaubt GM_xmlhttpRequest dorthin).
 *  3. LAGER_BASE unten: vollstaendige erreichbare App-URL (anderer PC im Netz!),
 *             idealerweise https. GM_xmlhttpRequest umgeht CORS und Mixed-Content.
 *
 * fuelleFormular() ist an die Sciformation-"Bestandsverkauf"-Maske angepasst
 * (Stand 2026-05-25). Bei Sciformation-Updates ggf. die Selektoren dort prüfen.
 */
(function () {
  'use strict';

const LAGER_BASE = 'https://fb08oc-ggb.org.chemie.uni-giessen.de'; // <-- anpassen (z.B. https://lager.intern.uni-giessen.de)

  // Selektor von Sciformations eigenem Speichern-/Speichern-und-beenden-Button.
  // Sciformation-Toolbar: <a class="z-toolbarbutton" title="Speichern"> bzw.
  // title="Speichern & Beenden". Leer lassen ('') deaktiviert die
  // Auto-Markierung — dann nur der manuelle Button.
  const SPEICHERN_SELECTOR = 'a.z-toolbarbutton[title="Speichern"], a.z-toolbarbutton[title="Speichern & Beenden"]';

  // Aktueller, im Panel geladener Verkauf.
  let aktuell = null;

  // Dokument, in dem das Verkaufsformular liegt. Sciformation rendert die Maske
  // in einem (gleich-origin) iframe → dort hineingreifen statt ins Top-Dokument.
  // Wird zu Beginn von fuelleFormular() gesetzt.
  let formDoc = document;

  function getFormDoc() {
    const iframes = Array.from(document.querySelectorAll('iframe'));
    // iframe bevorzugen, dessen Dokument die Verkaufsmaske enthält.
    for (const f of iframes) {
      let d = null;
      try { d = f.contentDocument; } catch (e) { d = null; } // cross-origin → null
      if (d && d.querySelector(
        'input.z-combobox-input, input.z-combobox-input-full, a.z-toolbarbutton')) {
        return d;
      }
    }
    // Sonst: erstes zugaengliches iframe-Dokument, zuletzt Top-Dokument.
    for (const f of iframes) {
      try { if (f.contentDocument) return f.contentDocument; } catch (e) { /* cross-origin */ }
    }
    return document;
  }

  // ── HTTP gegen die Lagerverwaltung-API (CORS-/Mixed-Content-frei) ──────────
  function apiRequest(method, pfad, body) {
    return new Promise((resolve, reject) => {
      GM_xmlhttpRequest({
        method,
        url: LAGER_BASE + pfad,
        headers: body ? { 'Content-Type': 'application/json' } : {},
        data: body ? JSON.stringify(body) : undefined,
        onload: (r) => {
          if (r.status === 204 || !r.responseText) return resolve(null);
          if (r.status < 200 || r.status >= 300) {
            return reject(new Error(`HTTP ${r.status}: ${r.responseText.slice(0, 200)}`));
          }
          try {
            resolve(JSON.parse(r.responseText));
          } catch (e) {
            reject(new Error('Antwort ist kein JSON: ' + e.message));
          }
        },
        onerror: () => reject(new Error('Netzwerkfehler — ist ' + LAGER_BASE + ' erreichbar (@connect, VPN/Firewall)?')),
        ontimeout: () => reject(new Error('Timeout bei ' + pfad)),
      });
    });
  }

  // ── UI: schwebendes Panel ───────────────────────────────────────────────────
  GM_addStyle(`
    #lv-sf-panel { position: fixed; right: 16px; bottom: 16px; z-index: 999999;
      width: 320px; font: 13px/1.4 system-ui, sans-serif; color: #1a1a1a;
      background: #fff; border: 1px solid #c4c4c4; border-radius: 8px;
      box-shadow: 0 4px 16px rgba(0,0,0,.18); overflow: hidden; }
    #lv-sf-panel header { background: #1565c0; color: #fff; padding: 8px 12px;
      font-weight: 600; display: flex; justify-content: space-between; align-items: center; }
    #lv-sf-panel .body { padding: 10px 12px; max-height: 50vh; overflow: auto; }
    #lv-sf-panel button { font: inherit; cursor: pointer; border-radius: 5px;
      border: 1px solid #1565c0; background: #1565c0; color: #fff; padding: 6px 10px; }
    #lv-sf-panel button.sec { background: #fff; color: #1565c0; }
    #lv-sf-panel button:disabled { opacity: .5; cursor: default; }
    #lv-sf-panel .row { display: flex; gap: 6px; margin-top: 8px; flex-wrap: wrap; }
    #lv-sf-panel .muted { color: #666; }
    #lv-sf-panel .pos { margin: 2px 0; }
    #lv-sf-panel code { background: #f0f0f0; padding: 0 3px; border-radius: 3px; }
    #lv-sf-status { margin-top: 8px; min-height: 1.2em; }
    #lv-sf-status.err { color: #c62828; }
    #lv-sf-min { background: transparent; border: none; color: #fff; padding: 0 4px; }
  `);

  const panel = document.createElement('div');
  panel.id = 'lv-sf-panel';
  panel.innerHTML = `
    <header>
      <span>Sciformation-Übertragung</span>
      <button id="lv-sf-min" title="Ein-/ausklappen">–</button>
    </header>
    <div class="body">
      <div class="row">
        <button id="lv-sf-load">Nächsten Verkauf laden</button>
      </div>
      <div id="lv-sf-content" class="muted" style="margin-top:8px;">Noch nichts geladen.</div>
      <div class="row" id="lv-sf-actions" style="display:none;">
        <button id="lv-sf-fill">In Formular eintragen</button>
        <button id="lv-sf-savesync">Speichern + übertragen</button>
        <button id="lv-sf-done" class="sec">Nur als übertragen markieren</button>
      </div>
      <div id="lv-sf-status"></div>
    </div>`;
  document.body.appendChild(panel);

  const $ = (id) => panel.querySelector(id);
  const setStatus = (txt, err) => {
    const el = $('#lv-sf-status');
    el.textContent = txt || '';
    el.className = err ? 'err' : '';
  };

  $('#lv-sf-min').addEventListener('click', () => {
    const b = panel.querySelector('.body');
    b.style.display = b.style.display === 'none' ? '' : 'none';
  });

  // ── Verkauf laden ─────────────────────────────────────────────────────────
  $('#lv-sf-load').addEventListener('click', async () => {
    setStatus('Lade…');
    try {
      const daten = await apiRequest('GET', '/api/sciformation/naechste');
      aktuell = daten;
      if (!daten) {
        $('#lv-sf-content').innerHTML = '<span class="muted">Keine offenen Verkäufe.</span>';
        $('#lv-sf-actions').style.display = 'none';
        setStatus('');
        return;
      }
      renderVerkauf(daten);
      $('#lv-sf-actions').style.display = '';
      setStatus('');
    } catch (e) {
      setStatus(e.message, true);
    }
  });

  function renderVerkauf(d) {
    const k = d.kunde || {};
    const posHtml = (d.positionen || [])
      .map((p) => `<div class="pos">• <code>${p.artikelnummer ?? '?'}</code> ${escapeHtml(p.bezeichnung ?? '')} <b>×${p.menge}</b></div>`)
      .join('');
    $('#lv-sf-content').innerHTML = `
      <div><b>Ausgabe #${d.id}</b> · ${new Date(d.datum).toLocaleDateString('de-DE')}
        ${d.vorgemerkt ? '<span style="color:#1565c0;">★ vorgemerkt</span>' : ''}
        <span class="muted">(noch ${d.offen_gesamt} offen)</span></div>
      <div style="margin-top:4px;"><b>Kunde:</b> ${escapeHtml(k.name ?? '—')}
        ${k.arbeitsgruppe ? `<span class="muted">(${escapeHtml(k.arbeitsgruppe)})</span>` : ''}</div>
      <div class="muted" style="font-size:12px;">
        Chipkarte: ${escapeHtml(k.chipkarte ?? '—')} · ID2: ${escapeHtml(k.mitarbeiter_id_2 ?? '—')}
        · User: ${escapeHtml(k.benutzername ?? '—')}</div>
      <div style="margin-top:6px;">${posHtml || '<span class="muted">keine Positionen</span>'}</div>`;
  }

  // ── Ins Formular eintragen ──────────────────────────────────────────────────
  $('#lv-sf-fill').addEventListener('click', async () => {
    if (!aktuell) return;
    setStatus('Trage ein…');
    try {
      await fuelleFormular(aktuell);
      // An Sciformations Speichern-Button binden: nach erfolgreichem Speichern
      // wird die Ausgabe automatisch als uebertragen markiert.
      bindSpeichernButton(aktuell.id);
      setStatus('Eingetragen — bitte prüfen und in Sciformation speichern.');
    } catch (e) {
      setStatus(e.message, true);
    }
  });

  // ── Als übertragen markieren (Fallback-Button) ──────────────────────────────
  $('#lv-sf-done').addEventListener('click', () => {
    if (aktuell) markiereSynced(aktuell.id);
  });

  // ── Speichern + übertragen in einem Schritt ─────────────────────────────────
  // Klickt Sciformations Speichern-Button und markiert die Ausgabe nach
  // erfolgreichem Speichern automatisch als uebertragen.
  $('#lv-sf-savesync').addEventListener('click', async () => {
    if (!aktuell) return;
    formDoc = getFormDoc();
    const btn = formDoc.querySelector('a.z-toolbarbutton[title="Speichern"]')
      || formDoc.querySelector('a.z-toolbarbutton[title="Speichern & Beenden"]');
    if (!btn) {
      setStatus('Speichern-Button in Sciformation nicht gefunden.', true);
      return;
    }
    setStatus('Speichere in Sciformation…');
    simulateClick(btn);
    await speichernUndVerbuchen(aktuell.id);
  });

  // Wartet auf den Speicher-Erfolg und markiert dann. Gegen Doppelausfuehrung
  // gesichert — der an den Speichern-Button gebundene Listener (bindSpeichern-
  // Button) und dieser Button teilen sich die Logik.
  let verbuchungLaeuft = false;
  async function speichernUndVerbuchen(ausgabeId) {
    if (verbuchungLaeuft) return;
    verbuchungLaeuft = true;
    try {
      const ok = await warteAufSpeicherErfolg();
      if (ok) {
        await markiereSynced(ausgabeId);
      } else {
        setStatus('Speichern nicht bestätigt — Ausgabe NICHT markiert. Bitte prüfen.', true);
      }
    } catch (e) {
      setStatus('Speicher-Erkennung fehlgeschlagen: ' + e.message, true);
    } finally {
      verbuchungLaeuft = false;
    }
  }

  // Setzt sciformation_synced_at im Backend und laedt den naechsten Verkauf.
  async function markiereSynced(id, sciformationId) {
    setStatus('Markiere…');
    try {
      const body = sciformationId ? { sciformation_id: sciformationId } : {};
      await apiRequest('POST', `/api/sciformation/${id}/synced`, body);
      setStatus(`Ausgabe #${id} als übertragen markiert.`);
      $('#lv-sf-load').click(); // naechsten Verkauf nachladen
    } catch (e) {
      setStatus(e.message, true);
    }
  }

  /*
   * Haengt sich an Sciformations eigenen Speichern-Button. Wenn der Nutzer
   * speichert UND der Speichervorgang erfolgreich war, wird die Ausgabe
   * automatisch markiert. Wichtig: NICHT schon beim blossen Klick markieren —
   * sonst wuerde ein von Sciformation wegen Validierung abgelehnter Verkauf
   * faelschlich als uebertragen gelten. Deshalb wartet die Markierung auf das
   * Erfolgssignal (warteAufSpeicherErfolg).
   *
   * Beim DOM-Erfassen zu ermitteln: SPEICHERN_SELECTOR (oben) und woran man den
   * Erfolg erkennt (Erfolgsmeldung/Toast, Formular leert sich, Navigation).
   */
  function bindSpeichernButton(ausgabeId) {
    if (!SPEICHERN_SELECTOR) return; // Auto-Markierung nicht konfiguriert
    const buttons = getFormDoc().querySelectorAll(SPEICHERN_SELECTOR);
    if (!buttons.length) {
      setStatus('Hinweis: Speichern-Button nicht gefunden — bitte nach dem Speichern „Als übertragen markieren" klicken.');
      return;
    }
    buttons.forEach((btn) => {
      // Einmal-Listener pro Eintrag-Vorgang (verhindert Doppel-Bindung).
      const handler = () => {
        buttons.forEach((b) => b.removeEventListener('click', handler));
        speichernUndVerbuchen(ausgabeId);
      };
      btn.addEventListener('click', handler);
    });
  }

  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  }

  /*
   * Formular-Befüllung — angepasst an Sciformation "Bestandsverkauf" (ZK),
   * erfasst 2026-05-25. `daten` = Antwort von /api/sciformation/naechste:
   *   { id, datum, kunde: { name, nachname, chipkarte, mitarbeiter_id_2, … },
   *     positionen: [ { artikelnummer, bezeichnung, menge, einzelpreis } ] }
   *
   * Annahmen (bei Sciformation-Updates ggf. anpassen):
   *  - Kundenfeld: Combobox placeholder "Suchen". Chipkartennummer eintippen;
   *    ZK setzt den Kunden (Fallback: ersten Vorschlag klicken).
   *  - Artikelzeile: Combobox placeholder "Suchbegriff/Barcode" (Klasse
   *    z-combobox-input-full). Art.-Nr. tippen → Vorschlag, dessen Text
   *    "[<Art.-Nr.>]" enthält, anklicken.
   *  - Menge: Textbox title "Artikelzahl" in derselben Zeile (tr.z-row).
   *  - Neue Zeile: Button (z-button) mit Bild consumable.png und Text "+".
   *  - ZK reagiert nicht auf reines el.value=x → setNativeValue + Events feuern.
   *  - ZK-IDs (vKYE…, hCpD…) wechseln bei jedem Laden → NUR über Klassen/
   *    Attribute/Text selektieren, nie über IDs.
   */
  async function fuelleFormular(daten) {
    console.log('[LV->SF] fuelleFormular - Daten:', daten);
    formDoc = getFormDoc();
    console.log('[LV->SF] Formular-Dokument:', formDoc === document ? 'Top-Fenster' : 'iframe');
    if (daten.kunde) {
      setStatus('Setze Kunde…');
      await setzeKunde(daten.kunde);
    }
    const positionen = daten.positionen || [];
    for (let i = 0; i < positionen.length; i++) {
      setStatus(`Trage Artikel ${i + 1}/${positionen.length} ein…`);
      await setzeArtikelPosition(positionen[i], i);
    }
  }

  async function setzeKunde(k) {
    let inp = formDoc.querySelector('input.z-combobox-input[placeholder="Suchen"]');
    if (!inp) {
      // Fallback: sichtbares Combobox-Input (kein Artikelfeld), dessen
      // placeholder "such" enthält.
      inp = Array.from(formDoc.querySelectorAll('input.z-combobox-input')).find(
        (e) => !e.classList.contains('z-combobox-input-full')
          && /such/i.test(e.placeholder || '') && isVisible(e)) || null;
    }
    if (!inp) {
      const phs = Array.from(formDoc.querySelectorAll(
        'input.z-combobox-input, input.z-combobox-input-full'))
        .map((e) => JSON.stringify(e.placeholder || '')).join(', ') || '(keine)';
      const frames = document.querySelectorAll('iframe').length;
      throw new Error('Kundenfeld nicht gefunden. Vorhandene Combobox-Placeholder: ['
        + phs + ']' + (frames ? ' · ' + frames + ' iframe(s) auf der Seite' : ''));
    }
    const wert = k.chipkarte || k.mitarbeiter_id_2 || k.nachname || k.name;
    if (!wert) throw new Error('Kunde hat keinen Suchwert (Chipkarte/Name).');
    inp.focus();
    setNativeValue(inp, String(wert));
    fireInput(inp);
    // Chipkarte ist eindeutig: auf Vorschläge warten und ersten Treffer klicken.
    // (Setzt ZK den Kunden schon ohne Auswahl, erscheinen keine Items — auch ok.)
    const combo = inp.closest('.z-combobox');
    const hatItems = await waitFor(
      () => combo.querySelectorAll('li.z-comboitem').length > 0, 3000, 150);
    if (hatItems) {
      simulateClick(combo.querySelector('li.z-comboitem'));
      await delay(300);
    }
  }

  function artikelInputs() {
    return Array.from(formDoc.querySelectorAll(
      'input.z-combobox-input-full[placeholder="Suchbegriff/Barcode"]')).filter(isVisible);
  }
  function leereArtikelZeileInput() {
    return artikelInputs().find((el) => !el.value.trim()) || null;
  }
  function neueArtikelZeileButton() {
    return Array.from(formDoc.querySelectorAll('button.z-button')).find(
      (b) => isVisible(b) && b.querySelector('img[src*="consumable.png"]') && b.textContent.trim() === '+');
  }

  async function setzeArtikelPosition(p, index) {
    // Leere Artikelzeile sicherstellen (erste ist schon da; weitere per "+").
    let inp = leereArtikelZeileInput();
    if (!inp) {
      const btn = neueArtikelZeileButton();
      if (!btn) throw new Error('Button „neue Artikelzeile" (+) nicht gefunden.');
      simulateClick(btn);
      await waitFor(() => leereArtikelZeileInput(), 4000, 150);
      inp = leereArtikelZeileInput();
    }
    if (!inp) throw new Error('Keine leere Artikelzeile für Position ' + (index + 1) + '.');

    const artnr = String(p.artikelnummer == null ? '' : p.artikelnummer).trim();
    if (!artnr) throw new Error('Position ' + (index + 1) + ' hat keine Artikelnummer.');

    // Zeile (tr) VOR der Auswahl merken — die Mengen-Textbox liegt darin.
    const row = inp.closest('tr');

    inp.focus();
    setNativeValue(inp, artnr);
    fireInput(inp);

    // Vorschlagsliste finden: ZK verschiebt das Popup teils aus dem Combobox-
    // Element heraus → über die per aria-owns/aria-controls verknüpfte Popup-ID
    // im ganzen Formular-Dokument suchen (Fallback: sichtbare comboitems).
    const span = inp.closest('.z-combobox');
    const ppId = (span && span.getAttribute('aria-owns')) || inp.getAttribute('aria-controls');
    const findeItems = () => {
      const pp = ppId ? formDoc.getElementById(ppId) : (span && span.querySelector('.z-combobox-popup'));
      let lis = pp ? Array.from(pp.querySelectorAll('li.z-comboitem')) : [];
      if (!lis.length) lis = Array.from(formDoc.querySelectorAll('li.z-comboitem')).filter(isVisible);
      return lis;
    };
    await waitFor(() => findeItems().length > 0, 6000, 150);
    const items = findeItems();
    if (!items.length) {
      throw new Error('Keine Vorschläge für Art.-Nr. ' + artnr + ' erschienen.');
    }
    // Auswahl: bevorzugt der Vorschlag mit "[<Art.-Nr.>]" im Text, sonst der
    // oberste Eintrag (so wie man es per Hand macht).
    const ziel = items.find((li) => li.textContent.includes('[' + artnr + ']')) || items[0];
    console.log('[LV->SF] Artikel', artnr, '→', ziel.textContent.trim().replace(/\s+/g, ' '));
    simulateClick(ziel);
    await delay(400);

    // Menge in derselben Zeile setzen.
    const qty = row && row.querySelector('input.z-textbox[title="Artikelzahl"]');
    if (qty && p.menge != null) setTextbox(qty, p.menge);
  }

  /*
   * Erfolgserkennung nach dem Speichern. Heuristik: kurz warten und prüfen, ob
   * Sciformation eine Fehler-/Validierungsmeldung zeigt. Keine Meldung →
   * Erfolg (true). Bei „Speichern & Beenden" verschwindet die Maske ohne
   * Vollnavigation (ZK-AJAX), das Skript läuft weiter — der /synced-Aufruf
   * kommt durch. Der Nutzer hat den Eintrag vorher visuell geprüft.
   */
  async function warteAufSpeicherErfolg() {
    await delay(1500);
    const sel = '.z-errorbox, .z-errbox, .z-messagebox-window, .z-notification-error';
    // Fehlermeldungen koennen im iframe ODER im Top-Fenster erscheinen.
    const boxen = [...getFormDoc().querySelectorAll(sel), ...document.querySelectorAll(sel)];
    const fehler = boxen.some(isVisible);
    return !fehler;
  }

  // ── Low-Level-Helfer (ZK- und iframe-tauglich) ──────────────────────────────
  // Elemente koennen aus dem iframe-Realm stammen → Konstruktoren/Prototypen
  // aus DEREN Fenster nehmen, sonst wirft Firefox "Illegal invocation".
  function winOf(el) {
    return (el.ownerDocument && el.ownerDocument.defaultView) || window;
  }
  function setNativeValue(el, value) {
    const win = winOf(el);
    const proto = el.tagName === 'TEXTAREA' ? win.HTMLTextAreaElement.prototype : win.HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, String(value));
  }
  function fireInput(el) {
    const win = winOf(el);
    el.dispatchEvent(new win.Event('input', { bubbles: true }));
    el.dispatchEvent(new win.KeyboardEvent('keyup', { bubbles: true }));
  }
  function setTextbox(el, value) {
    const win = winOf(el);
    el.focus();
    setNativeValue(el, value);
    el.dispatchEvent(new win.Event('input', { bubbles: true }));
    el.dispatchEvent(new win.Event('change', { bubbles: true }));
    el.blur(); // ZK-Textbox sendet onChange beim Blur
  }
  function simulateClick(el) {
    if (!el) return;
    const win = winOf(el);
    ['pointerdown', 'mousedown', 'mouseup', 'click'].forEach((type) =>
      el.dispatchEvent(new win.MouseEvent(type, { bubbles: true, cancelable: true, view: win })));
  }
  function isVisible(el) {
    return !!(el && (el.offsetWidth || el.offsetHeight || el.getClientRects().length));
  }
  function waitFor(fn, timeout = 5000, interval = 150) {
    return new Promise((resolve) => {
      const start = Date.now();
      (function tick() {
        let r = false;
        try { r = fn(); } catch (e) { r = false; }
        if (r) return resolve(r);
        if (Date.now() - start >= timeout) return resolve(false);
        setTimeout(tick, interval);
      })();
    });
  }
  function delay(ms) {
    return new Promise((r) => setTimeout(r, ms));
  }
})();


```
