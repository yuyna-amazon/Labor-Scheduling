// ==UserScript==
// @name         Labor Scheduling
// @namespace    https://github.com/yuyna-amazon/labor-scheduling
// @version      2.3
// @description  scheduling.amazon.com (opportunity-admin) を画面分割し、右側にシフト別の出勤者集約パネルを表示する
// @author       yuyna
// @icon         https://www.google.com/s2/favicons?sz=64&domain=amazon.com
// @match        https://scheduling.amazon.com/*
// @updateURL    https://raw.githubusercontent.com/yuyna-amazon/labor-scheduling/main/LaborScheduling.user.js
// @downloadURL  https://raw.githubusercontent.com/yuyna-amazon/labor-scheduling/main/LaborScheduling.user.js
// @run-at       document-idle
// @grant        GM_setClipboard
// @grant        unsafeWindow
// ==/UserScript==

(function () {
    'use strict';

    /* ======================================================
       定数 / 設定
    ====================================================== */

    const PANEL_ID = 'lsp-panel';
    const STYLE_ID = 'lsp-style';
    const REOPEN_ID = 'lsp-reopen';
    const LS_KEY = 'lsp:settings:v2';

    const ROUTE_KEYWORD = 'opportunity-admin';

    const WIDTH_MIN = 340;
    const WIDTH_DEFAULT = 520;

    const DEFAULTS = {
        width: WIDTH_DEFAULT,
        open: true,
        containFixed: false,
        allRoutes: false,
        mig23: false,        // 既存設定の containFixed を一度だけ無効化する移行フラグ
        autoRefresh: true,
        tab: 'shift',
        dateKey: 'ALL',      // 'ALL' or 'YYYY-MM-DD'
        shiftKey: 'range',   // 'range' = 開始-終了 / 'start' = 開始のみ
        excludeCancelled: true
    };

    let S = loadSettings();

    // === 状態 ===
    let DATA = null;       // { source, days: [...], employeeSource, sample }
    let lastSignature = '';
    let observer = null;
    let refreshTimer = null;
    let resizing = false;
    let expanded = new Set();   // 出勤者タブで開いているシフト

    /* ======================================================
       共通ユーティリティ
    ====================================================== */

    function loadSettings() {
        try {
            const raw = localStorage.getItem(LS_KEY);
            if (!raw) return { ...DEFAULTS, mig23: true };
            const s = { ...DEFAULTS, ...(JSON.parse(raw) || {}) };
            // v2.3: transform 方式はモーダルの位置を壊すため既存設定でも一度無効化する
            if (!s.mig23) { s.containFixed = false; s.mig23 = true; }
            return s;
        } catch (e) {
            return { ...DEFAULTS, mig23: true };
        }
    }

    function saveSettings() {
        try { localStorage.setItem(LS_KEY, JSON.stringify(S)); } catch (e) { /* noop */ }
    }

    function esc(v) {
        return String(v == null ? '' : v).replace(/[&<>"']/g, c => ({
            '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
        })[c]);
    }

    function clampWidth(w) {
        const max = Math.max(WIDTH_MIN, window.innerWidth - 340);
        return Math.min(Math.max(Math.round(w) || WIDTH_DEFAULT, WIDTH_MIN), max);
    }

    function fmtNum(n) {
        if (n == null || !Number.isFinite(n)) return '-';
        return (Math.round(n * 10) / 10).toLocaleString('ja-JP');
    }

    function pct(a, b) {
        if (!b) return null;
        return a / b * 100;
    }

    function nowLabel() {
        const d = new Date();
        const p = n => String(n).padStart(2, '0');
        return `${p(d.getHours())}:${p(d.getMinutes())}:${p(d.getSeconds())}`;
    }

    function copyText(text) {
        try {
            if (typeof GM_setClipboard === 'function') { GM_setClipboard(text, 'text'); return true; }
        } catch (e) { /* fallthrough */ }
        try {
            const ta = document.createElement('textarea');
            ta.value = text;
            ta.style.cssText = 'position:fixed;left:-9999px';
            document.documentElement.appendChild(ta);
            ta.select();
            document.execCommand('copy');
            ta.remove();
            return true;
        } catch (e) { return false; }
    }

    function flash(btn, label) {
        const old = btn.textContent;
        btn.textContent = label || 'コピー済';
        setTimeout(() => { btn.textContent = old; }, 1200);
    }

    function isTargetRoute() {
        if (S.allRoutes) return true;
        return (location.hash || '').includes(ROUTE_KEYWORD);
    }

    function currentSiteId() {
        const m = (location.href || '').match(/siteId=([A-Za-z0-9_-]+)/);
        return m ? m[1] : '-';
    }

    // ページ側の window（Tampermonkey サンドボックス回避）
    function pageWindow() {
        try {
            if (typeof unsafeWindow !== 'undefined' && unsafeWindow) return unsafeWindow;
        } catch (e) { /* noop */ }
        return window;
    }

    function getAngular() {
        try {
            const w = pageWindow();
            if (w.angular && typeof w.angular.element === 'function') return w.angular;
        } catch (e) { /* noop */ }
        try {
            if (window.angular && typeof window.angular.element === 'function') return window.angular;
        } catch (e) { /* noop */ }
        return null;
    }

    // 時刻 "HH:MM" → 分。ソート用
    function timeToMin(t) {
        const m = /^(\d{1,2}):(\d{2})$/.exec(String(t || '').trim());
        if (!m) return 99999;
        return parseInt(m[1], 10) * 60 + parseInt(m[2], 10);
    }

    /* ======================================================
       スタイル
    ====================================================== */

    function injectStyle() {
        if (document.getElementById(STYLE_ID)) return;
        const st = document.createElement('style');
        st.id = STYLE_ID;
        st.textContent = `
:root { --lsp-w: ${WIDTH_DEFAULT}px; }

/* --- 画面分割：本体側を右パネル分だけ狭める --- */
html.lsp-split > body {
    width: calc(100% - var(--lsp-w)) !important;
    min-width: 0 !important;
    max-width: none !important;
    box-sizing: border-box !important;
    overflow-x: auto !important;
}
/* 実験オプション。position:fixed の基準が body になるため、モーダルが
   画面外に出る副作用がある。既定は無効。モーダル表示中は自動解除する。 */
html.lsp-split.lsp-contain:not(.lsp-modal) > body {
    transform: translateZ(0) !important;
    will-change: transform;
}

/* --- Bootstrap / angular-ui-bootstrap の固定要素を分割幅に収める --- */
html.lsp-split .modal,
html.lsp-split .modal-backdrop,
html.lsp-split [uib-modal-window],
html.lsp-split [uib-modal-backdrop] {
    right: var(--lsp-w) !important;
    left: 0 !important;
    width: auto !important;
}
html.lsp-split .navbar-fixed-top,
html.lsp-split .navbar-fixed-bottom {
    right: var(--lsp-w) !important;
    left: 0 !important;
    width: auto !important;
}
/* モーダルは本体側の最前面に出す（パネルより下、ページ内では最上位） */
html.lsp-split .modal, html.lsp-split [uib-modal-window] { z-index: 2147482000 !important; }
html.lsp-split .modal-backdrop, html.lsp-split [uib-modal-backdrop] { z-index: 2147481900 !important; }

/* --- パネル --- */
#${PANEL_ID} {
    position: fixed; top: 0; right: 0; bottom: 0;
    width: var(--lsp-w); z-index: 2147483000;
    display: flex; flex-direction: column;
    background: #fff; border-left: 1px solid #d5dbdb;
    box-shadow: -2px 0 8px rgba(0,0,0,.08);
    font-family: "Segoe UI", "Meiryo", system-ui, sans-serif;
    font-size: 12px; color: #16191f; box-sizing: border-box;
}
#${PANEL_ID} *, #${PANEL_ID} *::before, #${PANEL_ID} *::after { box-sizing: border-box; }

#${PANEL_ID} .lsp-grip {
    position: absolute; top: 0; left: -3px; bottom: 0; width: 7px;
    cursor: col-resize; z-index: 5;
}
#${PANEL_ID} .lsp-grip:hover { background: rgba(0,115,187,.25); }
html.lsp-resizing, html.lsp-resizing * { cursor: col-resize !important; user-select: none !important; }

#${PANEL_ID} .lsp-head {
    display: flex; align-items: center; gap: 6px;
    padding: 8px 10px; background: #232f3e; color: #fff; flex: 0 0 auto;
}
#${PANEL_ID} .lsp-title { font-weight: 700; font-size: 13px; white-space: nowrap; }
#${PANEL_ID} .lsp-badge {
    background: #ff9900; color: #16191f; font-weight: 700;
    border-radius: 9px; padding: 1px 8px; font-size: 11px; white-space: nowrap;
}
#${PANEL_ID} .lsp-head .lsp-spacer { flex: 1 1 auto; }
#${PANEL_ID} .lsp-ico {
    border: 0; background: rgba(255,255,255,.12); color: #fff;
    width: 26px; height: 24px; border-radius: 4px; cursor: pointer;
    font-size: 13px; line-height: 1; display: inline-flex; align-items: center; justify-content: center;
}
#${PANEL_ID} .lsp-ico:hover { background: rgba(255,255,255,.28); }

#${PANEL_ID} .lsp-tabs {
    display: flex; gap: 2px; padding: 6px 8px 0;
    background: #f2f3f3; border-bottom: 1px solid #d5dbdb; flex: 0 0 auto;
}
#${PANEL_ID} .lsp-tab {
    border: 1px solid transparent; border-bottom: 0; background: transparent;
    padding: 5px 10px; border-radius: 6px 6px 0 0; cursor: pointer;
    font-size: 12px; color: #545b64; white-space: nowrap;
}
#${PANEL_ID} .lsp-tab:hover { color: #16191f; background: #e9ebed; }
#${PANEL_ID} .lsp-tab.on {
    background: #fff; border-color: #d5dbdb; color: #0073bb;
    font-weight: 700; margin-bottom: -1px;
}

#${PANEL_ID} .lsp-body { flex: 1 1 auto; overflow: auto; padding: 10px; }
#${PANEL_ID} .lsp-foot {
    flex: 0 0 auto; padding: 5px 10px; border-top: 1px solid #eaeded;
    background: #fafafa; color: #687078; font-size: 11px;
}

#${PANEL_ID} .lsp-cards { display: grid; grid-template-columns: repeat(4, 1fr); gap: 6px; margin-bottom: 10px; }
#${PANEL_ID} .lsp-card { background: #f7f8f8; border: 1px solid #eaeded; border-radius: 6px; padding: 6px 8px; }
#${PANEL_ID} .lsp-card .k { font-size: 10px; color: #687078; white-space: nowrap; }
#${PANEL_ID} .lsp-card .v { font-size: 17px; font-weight: 700; }
#${PANEL_ID} .lsp-card.warn .v { color: #d13212; }

#${PANEL_ID} .lsp-row { display: flex; gap: 6px; align-items: center; margin-bottom: 8px; flex-wrap: wrap; }
#${PANEL_ID} .lsp-lb { font-size: 11px; color: #545b64; }
#${PANEL_ID} select, #${PANEL_ID} input[type="text"] {
    font: inherit; padding: 3px 6px; border: 1px solid #aab7b8;
    border-radius: 4px; background: #fff; color: #16191f; max-width: 100%;
}
#${PANEL_ID} .lsp-btn {
    font: inherit; padding: 3px 9px; border: 1px solid #aab7b8; border-radius: 4px;
    background: #fff; cursor: pointer; color: #16191f; white-space: nowrap;
}
#${PANEL_ID} .lsp-btn:hover { background: #f2f3f3; }
#${PANEL_ID} .lsp-btn.primary { background: #0073bb; border-color: #0073bb; color: #fff; }
#${PANEL_ID} .lsp-btn.primary:hover { background: #005d99; }

#${PANEL_ID} h4 {
    margin: 14px 0 6px; font-size: 12px; color: #16191f;
    border-bottom: 1px solid #eaeded; padding-bottom: 3px;
}
#${PANEL_ID} h4:first-child { margin-top: 0; }

#${PANEL_ID} table.lsp-t { border-collapse: collapse; width: 100%; font-size: 11px; }
#${PANEL_ID} table.lsp-t th, #${PANEL_ID} table.lsp-t td {
    border: 1px solid #eaeded; padding: 3px 6px; text-align: left; vertical-align: top;
}
#${PANEL_ID} table.lsp-t th {
    background: #f2f3f3; position: sticky; top: 0; z-index: 1; font-weight: 700; white-space: nowrap;
}
#${PANEL_ID} table.lsp-t td.num { text-align: right; font-variant-numeric: tabular-nums; white-space: nowrap; }
#${PANEL_ID} table.lsp-t td.nw { white-space: nowrap; }
#${PANEL_ID} table.lsp-t tbody tr:nth-child(even) { background: #fcfcfc; }
#${PANEL_ID} table.lsp-t tbody tr:hover { background: #f0f7fb; }
#${PANEL_ID} table.lsp-t tfoot td { background: #f7f8f8; font-weight: 700; }
#${PANEL_ID} table.lsp-t td.short { color: #d13212; font-weight: 700; }
#${PANEL_ID} table.lsp-t td.full { color: #1d8102; }

#${PANEL_ID} .lsp-bar { position: relative; background: #eaeded; border-radius: 3px; height: 12px; min-width: 46px; }
#${PANEL_ID} .lsp-bar > i { position: absolute; inset: 0 auto 0 0; background: #0073bb; border-radius: 3px; }
#${PANEL_ID} .lsp-bar.ok > i { background: #1d8102; }
#${PANEL_ID} .lsp-bar.low > i { background: #d13212; }

#${PANEL_ID} .lsp-note { color: #687078; font-size: 11px; line-height: 1.6; }
#${PANEL_ID} .lsp-warn {
    background: #fffaf0; border: 1px solid #ffe0a3; border-radius: 4px;
    padding: 8px; color: #6b4b00; line-height: 1.6;
}
#${PANEL_ID} .lsp-err {
    background: #fdf3f1; border: 1px solid #f5c4bb; border-radius: 4px;
    padding: 8px; color: #8b2013; line-height: 1.6;
}
#${PANEL_ID} pre.lsp-pre {
    background: #16191f; color: #d5f5d5; padding: 8px; border-radius: 4px;
    font-size: 10px; line-height: 1.5; max-height: 46vh; overflow: auto;
    white-space: pre; margin: 0;
}
#${PANEL_ID} .lsp-chk { display: flex; align-items: center; gap: 6px; margin: 7px 0; font-size: 12px; }

/* 出勤者タブ */
#${PANEL_ID} .lsp-shift {
    border: 1px solid #eaeded; border-radius: 5px; margin-bottom: 6px; overflow: hidden;
}
#${PANEL_ID} .lsp-shift > .hd {
    display: flex; align-items: center; gap: 8px; padding: 5px 8px;
    background: #f7f8f8; cursor: pointer; user-select: none;
}
#${PANEL_ID} .lsp-shift > .hd:hover { background: #f0f7fb; }
#${PANEL_ID} .lsp-shift > .hd .t { font-weight: 700; font-variant-numeric: tabular-nums; }
#${PANEL_ID} .lsp-shift > .hd .c { color: #545b64; font-size: 11px; }
#${PANEL_ID} .lsp-shift > .hd .sp { flex: 1 1 auto; }
#${PANEL_ID} .lsp-shift > .bd { padding: 6px 8px; border-top: 1px solid #eaeded; }
#${PANEL_ID} .lsp-emps { display: flex; flex-wrap: wrap; gap: 4px; }
#${PANEL_ID} .lsp-emp {
    background: #eff7fd; border: 1px solid #cfe6f7; border-radius: 10px;
    padding: 1px 8px; font-size: 11px; white-space: nowrap;
}
#${PANEL_ID} .lsp-emp.dup { background: #fdf3f1; border-color: #f5c4bb; color: #8b2013; }
#${PANEL_ID} table.lsp-t tbody tr.dup { background: #fdf3f1; }
#${PANEL_ID} table.lsp-t tbody tr.dup:hover { background: #fbe8e4; }
#${PANEL_ID} .lsp-sub { color: #687078; font-size: 10px; margin: 4px 0 2px; }
#${PANEL_ID} .lsp-actions {
    margin-top: 14px; padding-top: 10px; border-top: 1px solid #d5dbdb;
}
#${PANEL_ID} .lsp-actions .lsp-row:last-of-type { margin-bottom: 4px; }

#${REOPEN_ID} {
    position: fixed; top: 50%; right: 0; transform: translateY(-50%);
    z-index: 2147483000; background: #232f3e; color: #fff; border: 0;
    border-radius: 6px 0 0 6px; padding: 10px 6px; cursor: pointer;
    font-family: "Segoe UI", "Meiryo", sans-serif; font-size: 11px;
    writing-mode: vertical-rl; letter-spacing: 2px;
    box-shadow: -2px 0 6px rgba(0,0,0,.2);
}
#${REOPEN_ID}:hover { background: #37475a; }
`;
        (document.head || document.documentElement).appendChild(st);
    }

    /* ======================================================
       データ抽出
       ・表示値（人員数 / 開始 / 終了 / ワークグループ）は DOM から
       ・承認済み従業員は AngularJS スコープ（singleOpportunity.acceptedEmployees）から
    ====================================================== */

    function txt(el) {
        if (!el) return '';
        return (el.innerText || el.textContent || '').replace(/\s+/g, ' ').trim();
    }

    // <br> 区切りを行として取り出す
    function txtLines(el) {
        if (!el) return [];
        return (el.innerText || el.textContent || '')
            .split(/\r?\n/)
            .map(s => s.trim())
            .filter(Boolean);
    }

    function nextTd(td) {
        let n = td ? td.nextElementSibling : null;
        while (n && n.tagName !== 'TD') n = n.nextElementSibling;
        return n;
    }

    /* --- 従業員オブジェクトから 名前 / ログイン / 従業員ID を推定 --- */

    // 優先順のキー候補（大文字小文字は無視して完全一致で探す）
    const NAME_KEYS = ['employeeName', 'name', 'fullName', 'displayName', 'associateName',
                       'personName', 'employeeFullName', 'associateFullName'];
    const LOGIN_KEYS = ['employeeLogin', 'login', 'alias', 'loginId', 'userName', 'username',
                        'user', 'ldap', 'ldapId'];
    const ID_KEYS = ['employeeId', 'badgeId', 'personId', 'associateId', 'employeeNumber',
                     'badgeNumber', 'employeeGuid', 'personnelId', 'id'];

    // 完全一致で見つからない場合の部分一致フォールバック
    const NAME_RE = [/name$/i];
    const LOGIN_RE = [/login/i, /alias/i];
    const ID_RE = [/(employee|person|associate|badge|personnel).*(id|number)$/i, /^id$/i];

    // ネストを1段だけ展開してスカラー値のみ集める
    function flattenScalars(obj, depth) {
        const out = {};
        if (!obj || typeof obj !== 'object') return out;
        Object.keys(obj).forEach(k => {
            const v = obj[k];
            if (v == null) return;
            if (typeof v === 'string' || typeof v === 'number') {
                if (!(k in out)) out[k] = String(v).trim();
            } else if (typeof v === 'object' && !Array.isArray(v) && (depth || 0) < 1) {
                const inner = flattenScalars(v, (depth || 0) + 1);
                Object.keys(inner).forEach(ik => { if (!(ik in out)) out[ik] = inner[ik]; });
            }
        });
        return out;
    }

    function pickField(flat, exactKeys, regexes, excludeRe) {
        const keys = Object.keys(flat);
        for (const want of exactKeys) {
            const hit = keys.find(k => k.toLowerCase() === want.toLowerCase() && flat[k]);
            if (hit) return flat[hit];
        }
        for (const re of regexes) {
            const hit = keys.find(k => flat[k] && re.test(k) && !(excludeRe && excludeRe.test(k)));
            if (hit) return flat[hit];
        }
        return '';
    }

    function mapEmployee(e) {
        const rec = { name: '', login: '', id: '', label: '', key: '', raw: e };

        if (e == null) return rec;
        if (typeof e === 'string' || typeof e === 'number') {
            rec.login = String(e);
            rec.label = rec.login;
            rec.key = rec.login;
            return rec;
        }

        const flat = flattenScalars(e, 0);

        rec.login = pickField(flat, LOGIN_KEYS, LOGIN_RE, null);
        rec.id = pickField(flat, ID_KEYS, ID_RE, /login|alias/i);
        // 「〜Name」でも login / user 系のキーは名前として扱わない
        rec.name = pickField(flat, NAME_KEYS, NAME_RE, /login|alias|user|file|group|site|type|status/i);

        if (!rec.name) {
            const first = pickField(flat, ['firstName', 'givenName'], [], null);
            const last = pickField(flat, ['lastName', 'familyName', 'surname'], [], null);
            if (first || last) rec.name = [first, last].filter(Boolean).join(' ');
        }

        // 名前がログインと同一なら名前扱いしない
        if (rec.name && rec.login && rec.name.toLowerCase() === rec.login.toLowerCase()) rec.name = '';
        // IDがログインと同一なら重複表示しない
        if (rec.id && rec.login && rec.id.toLowerCase() === rec.login.toLowerCase()) rec.id = '';

        rec.key = rec.id || rec.login || rec.name || '';
        rec.label = rec.name && rec.login ? `${rec.name} (${rec.login})`
                  : (rec.name || rec.login || rec.id || '(不明)');
        return rec;
    }

    function parseRow(tr, ng, stat) {
        const tds = Array.from(tr.children).filter(el => el.tagName === 'TD');
        if (!tds.length) return null;

        // 人員数セル（ng-click=getHeadcountDetailInfo / class="pointer"）を基準にする
        let hcTd = tr.querySelector('td.pointer[ng-click]') || tr.querySelector('td.pointer');
        if (!hcTd) hcTd = tds.find(td => /^\d+\s*\/\s*\d+$/.test(txt(td)));
        if (!hcTd) return null;

        const hcIdx = tds.indexOf(hcTd);
        const hcText = txt(hcTd);
        const m = /(\d+)\s*\/\s*(\d+)/.exec(hcText);
        const accepted = m ? parseInt(m[1], 10) : null;
        const headcount = m ? parseInt(m[2], 10) : null;

        // 人員数の直後 4 列 = 機会開始 / 機会終了 / サインアップ開始 / サインアップ終了
        const start = txt(tds[hcIdx + 1]);
        const end = txt(tds[hcIdx + 2]);
        const signupStart = txt(tds[hcIdx + 3]);
        const signupEnd = txt(tds[hcIdx + 4]);

        // ワークグループ列（workgroups の ng-repeat が入っている td）
        const wgSpan = tr.querySelector('td span[ng-repeat*="workgroups"]');
        const wgTd = wgSpan ? wgSpan.closest('td') : null;
        const workgroups = wgTd ? txtLines(wgTd) : [];
        const type = wgTd ? txt(nextTd(wgTd)) : '';

        const cancelled = !!tr.querySelector('td.cancelled-opportunity');
        const full = !!tr.querySelector('.glyphicon-warning-sign');

        // Angular スコープから承認済み従業員
        let employees = null;
        let rawKeys = null;
        let empKeys = null;
        if (ng) {
            stat.tried++;
            try {
                const sc = ng.element(tr).scope();
                const o = sc && sc.singleOpportunity;
                if (o) {
                    stat.hit++;
                    rawKeys = Object.keys(o);
                    if (Array.isArray(o.acceptedEmployees)) {
                        employees = o.acceptedEmployees.map(mapEmployee);
                        const first = o.acceptedEmployees[0];
                        if (first && typeof first === 'object') empKeys = Object.keys(first);
                    }
                }
            } catch (e) { /* noop */ }
        }

        return {
            id: tr.getAttribute('opportunity-id') || '',
            workgroups,
            type,
            accepted,
            headcount,
            start,
            end,
            signupStart,
            signupEnd,
            cancelled,
            full,
            employees,     // null = 取得不可 / [] = 0名
            rawKeys,
            empKeys
        };
    }

    function extractData() {
        const containers = Array.from(document.querySelectorAll('div.date_container'));
        if (!containers.length) return null;

        const ng = getAngular();
        const stat = { tried: 0, hit: 0 };
        const days = [];
        let sampleOppKeys = null;
        let sampleEmpKeys = null;

        containers.forEach(c => {
            const h = c.querySelector('h5');
            const label = txt(h);
            const dm = /(\d{4}-\d{2}-\d{2})/.exec(label);
            const dateKey = dm ? dm[1] : (label || '不明');

            const rows = Array.from(c.querySelectorAll('tr[opportunity-id]'));
            const opportunities = [];
            rows.forEach(tr => {
                const o = parseRow(tr, ng, stat);
                if (!o) return;
                if (!sampleOppKeys && o.rawKeys) sampleOppKeys = o.rawKeys;
                if (!sampleEmpKeys && o.empKeys) sampleEmpKeys = o.empKeys;
                opportunities.push(o);
            });

            if (opportunities.length) days.push({ dateKey, label, opportunities });
        });

        if (!days.length) return null;

        // 従業員名が1件でも取れたか
        const anyEmployees = days.some(d => d.opportunities.some(o => Array.isArray(o.employees) && o.employees.length));
        const anyEmployeeArray = days.some(d => d.opportunities.some(o => Array.isArray(o.employees)));

        return {
            days,
            angular: !!ng,
            scopeHit: stat.hit,
            scopeTried: stat.tried,
            employeeSource: anyEmployeeArray ? 'angular' : 'none',
            hasEmployeeNames: anyEmployees,
            sampleOppKeys,
            sampleEmpKeys
        };
    }

    function signatureOf(d) {
        if (!d) return 'none';
        const parts = [d.employeeSource];
        d.days.forEach(day => {
            parts.push(day.dateKey + ':' + day.opportunities.length);
            day.opportunities.forEach(o => {
                parts.push(`${o.start}-${o.end}|${o.accepted}/${o.headcount}|${Array.isArray(o.employees) ? o.employees.length : 'x'}`);
            });
        });
        return parts.join(';');
    }

    /* ======================================================
       従業員名簿ディレクトリ
       行スコープには従業員ログインしか無いため、詳細モーダルの表から
       「従業員名 / 従業員ログイン / 従業員ID」を回収してログインで突き合わせる
    ====================================================== */

    const DIR_KEY = 'lsp:dir:v1';
    let DIR = new Map();       // loginLower -> { login, name, id }
    let harvesting = false;
    let harvestAbort = false;

    // モーダル表のヘッダー判定
    const HDR_LOGIN = /従業員ログイン|ログイン|\blogin\b|alias/i;
    const HDR_ID = /従業員\s*id|従業員番号|バッジ|badge|employee\s*id|associate\s*id|^id$/i;
    const HDR_NAME = /従業員名|氏名|名前|employee\s*name|associate\s*name|^name$/i;

    function loadDir() {
        try {
            const raw = localStorage.getItem(DIR_KEY);
            if (!raw) return;
            const obj = JSON.parse(raw) || {};
            Object.keys(obj).forEach(k => DIR.set(k, obj[k]));
        } catch (e) { /* noop */ }
    }

    function saveDir() {
        try {
            const obj = {};
            DIR.forEach((v, k) => { obj[k] = v; });
            localStorage.setItem(DIR_KEY, JSON.stringify(obj));
        } catch (e) { /* noop */ }
    }

    function enrichEmployee(e) {
        if (!e) return e;
        const login = e.login || '';
        const hit = login ? DIR.get(login.toLowerCase()) : null;
        if (!hit) return e;
        const name = e.name || hit.name || '';
        const id = e.id || hit.id || '';
        return {
            ...e, name, id,
            key: id || login || name,
            label: name && login ? `${name} (${login})` : (name || login || id || '(不明)')
        };
    }

    function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

    async function waitFor(fn, timeout, interval) {
        const t0 = Date.now();
        for (;;) {
            let v = null;
            try { v = fn(); } catch (e) { v = null; }
            if (v) return v;
            if (Date.now() - t0 >= timeout) return null;
            await sleep(interval || 120);
        }
    }

    function visibleModal() {
        const sel = '[uib-modal-window], .modal.in, .modal[style*="display: block"], .modal-dialog, .modal';
        return Array.from(document.querySelectorAll(sel)).find(el => {
            if (el.closest('#' + PANEL_ID)) return false;
            const r = el.getBoundingClientRect();
            return r.width > 120 && r.height > 60;
        }) || null;
    }

    // 従業員名簿らしい表を探す（クラス名に依存せずヘッダー文言で判定）
    function findEmployeeTables(root) {
        return Array.from((root || document).querySelectorAll('table')).filter(t => {
            if (t.closest('#' + PANEL_ID)) return false;
            if (t.closest('div.date_container')) return false;
            const texts = Array.from(t.querySelectorAll('th, thead td')).map(txt);
            if (!texts.length) return false;
            const n = [HDR_LOGIN, HDR_ID, HDR_NAME].filter(re => texts.some(x => re.test(x))).length;
            return n >= 2 && t.querySelectorAll('tbody tr').length > 0;
        });
    }

    function headerRowOf(t) {
        if (t.tHead && t.tHead.rows.length) return t.tHead.rows[t.tHead.rows.length - 1];
        for (const r of t.rows) {
            if (r.cells.length && Array.from(r.cells).some(c => c.tagName === 'TH')) return r;
        }
        return null;
    }

    // 表を名簿ディレクトリに取り込む
    function scrapeDirectory(t) {
        const hr = headerRowOf(t);
        if (!hr) return { added: 0, rows: 0, noLogin: true };

        const heads = Array.from(hr.cells).map(txt);
        let iLogin = -1, iId = -1, iName = -1;
        heads.forEach((h, i) => {
            if (iLogin < 0 && HDR_LOGIN.test(h)) { iLogin = i; return; }
            if (iId < 0 && HDR_ID.test(h)) { iId = i; return; }
            if (iName < 0 && HDR_NAME.test(h)) { iName = i; return; }
        });

        if (iLogin < 0) return { added: 0, rows: 0, noLogin: true };

        const bodyRows = t.tBodies.length
            ? Array.from(t.tBodies).flatMap(b => Array.from(b.rows))
            : Array.from(t.rows).filter(r => r !== hr);

        let added = 0, rows = 0;
        bodyRows.forEach(r => {
            const cells = Array.from(r.cells).map(txt);
            const login = (cells[iLogin] || '').trim();
            if (!login) return;
            rows++;
            const name = iName >= 0 ? (cells[iName] || '').trim() : '';
            const id = iId >= 0 ? (cells[iId] || '').trim() : '';
            if (!name && !id) return;
            const k = login.toLowerCase();
            const prev = DIR.get(k) || { login, name: '', id: '' };
            const next = { login, name: name || prev.name, id: id || prev.id };
            if (!prev || prev.name !== next.name || prev.id !== next.id) added++;
            DIR.set(k, next);
        });

        return { added, rows, noLogin: false };
    }

    // 開いているモーダルを閉じる（破壊的なボタンは押さない）
    async function closeModal() {
        const m = visibleModal();
        if (!m) return true;

        let btn = m.querySelector('.modal-header .close, button.close, [ng-click*="dismiss"], [ng-click*="cancel"]');
        if (!btn) {
            btn = Array.from(m.querySelectorAll('button, a.btn')).find(b =>
                /^(閉じる|閉|close|キャンセル|cancel|ok|戻る)$/i.test(txt(b)));
        }
        if (btn) btn.click();

        let gone = await waitFor(() => !visibleModal(), 1500, 100);
        if (!gone) {
            ['keydown', 'keyup'].forEach(type => {
                document.dispatchEvent(new KeyboardEvent(type, {
                    key: 'Escape', code: 'Escape', keyCode: 27, which: 27, bubbles: true
                }));
            });
            gone = await waitFor(() => !visibleModal(), 1500, 100);
        }
        return !!gone;
    }

    function harvestStatus(text) {
        const p = document.getElementById(PANEL_ID);
        const el = p ? p.querySelector('[data-hstat]') : null;
        if (el) el.textContent = text;
        setFoot(text);
    }

    // 対象の date_container（日付フィルタを尊重）
    function targetContainers() {
        const all = Array.from(document.querySelectorAll('div.date_container'));
        if (S.dateKey === 'ALL') return all;
        return all.filter(c => {
            const label = txt(c.querySelector('h5'));
            return label.indexOf(S.dateKey) >= 0;
        });
    }

    async function harvest(targets, label) {
        if (harvesting) return;
        if (!targets.length) { harvestStatus('対象がありません'); return; }

        harvesting = true;
        harvestAbort = false;
        let ok = 0, failed = 0, noLogin = 0;

        try {
            for (let i = 0; i < targets.length; i++) {
                if (harvestAbort) break;
                harvestStatus(`${label} ${i + 1}/${targets.length} … 名簿 ${DIR.size}件`);

                try { targets[i].click(); } catch (e) { /* noop */ }

                const table = await waitFor(() => {
                    const m = visibleModal();
                    if (!m) return null;
                    return findEmployeeTables(m)[0] || null;
                }, 7000, 150);

                if (table) {
                    const r = scrapeDirectory(table);
                    if (r.noLogin) noLogin++; else ok++;
                } else {
                    failed++;
                }

                await closeModal();
                await sleep(200);
            }
        } finally {
            harvesting = false;
            saveDir();
            lastSignature = '';
            refresh(true);
            const msg = [`${label} ${harvestAbort ? '中断' : '完了'}`,
                         `名簿 ${DIR.size}件`,
                         failed ? `表を検出できず ${failed}` : '',
                         noLogin ? `ログイン列なし ${noLogin}` : ''].filter(Boolean).join(' / ');
            harvestStatus(msg);
        }
    }

    function harvestByOpportunity() {
        const targets = [];
        targetContainers().forEach(c => {
            c.querySelectorAll('tr[opportunity-id]').forEach(tr => {
                if (S.excludeCancelled && tr.querySelector('td.cancelled-opportunity')) return;
                const td = tr.querySelector('td.pointer[ng-click]') || tr.querySelector('td.pointer');
                if (!td) return;
                const m = /(\d+)\s*\/\s*(\d+)/.exec(txt(td));
                if (m && parseInt(m[1], 10) === 0) return;   // 承認済み0名はスキップ
                targets.push(td);
            });
        });
        harvest(targets, '機会ごとに取得');
    }

    function harvestByDaySummary() {
        const targets = [];
        targetContainers().forEach(c => {
            const a = Array.from(c.querySelectorAll('a, button'))
                .find(el => /getAcceptedSummary/.test(el.getAttribute('ng-click') || ''));
            if (a) targets.push(a);
        });
        harvest(targets, '日別サマリーから取得');
    }

    /* ======================================================
       集計
    ====================================================== */

    function selectedDays() {
        if (!DATA) return [];
        if (S.dateKey === 'ALL') return DATA.days;
        return DATA.days.filter(d => d.dateKey === S.dateKey);
    }

    function targetOpportunities() {
        const out = [];
        selectedDays().forEach(d => {
            d.opportunities.forEach(o => {
                if (S.excludeCancelled && o.cancelled) return;
                out.push({
                    ...o,
                    dateKey: d.dateKey,
                    dateLabel: d.label,
                    employees: Array.isArray(o.employees) ? o.employees.map(enrichEmployee) : o.employees
                });
            });
        });
        return out;
    }

    function shiftKeyOf(o) {
        return S.shiftKey === 'start' ? (o.start || '-') : `${o.start || '-'} - ${o.end || '-'}`;
    }

    // シフト単位に集約
    function aggregateShifts(opps) {
        const map = new Map();

        opps.forEach(o => {
            const key = shiftKeyOf(o);
            if (!map.has(key)) {
                map.set(key, {
                    key,
                    start: o.start,
                    end: o.end,
                    count: 0,
                    accepted: 0,
                    headcount: 0,
                    employees: [],
                    people: new Set(),
                    workgroups: new Set(),
                    dates: new Set(),
                    employeesKnown: true
                });
            }
            const g = map.get(key);
            g.count++;
            if (Number.isFinite(o.accepted)) g.accepted += o.accepted;
            if (Number.isFinite(o.headcount)) g.headcount += o.headcount;
            o.workgroups.forEach(w => g.workgroups.add(w));
            g.dates.add(o.dateKey);

            if (Array.isArray(o.employees)) {
                o.employees.forEach(e => {
                    g.employees.push({ ...e, dateKey: o.dateKey, workgroups: o.workgroups });
                    if (e.key) g.people.add(e.key);
                });
            } else {
                g.employeesKnown = false;
            }
        });

        const groups = Array.from(map.values());
        groups.sort((a, b) => {
            const d = timeToMin(a.start) - timeToMin(b.start);
            if (d !== 0) return d;
            return timeToMin(a.end) - timeToMin(b.end);
        });
        return groups;
    }

    // 同一日で複数シフトに登録されている人（重複）
    function duplicatePeople(opps) {
        const perDate = new Map();   // dateKey -> Map(key -> { emp, shifts:Set })
        opps.forEach(o => {
            if (!Array.isArray(o.employees)) return;
            if (!perDate.has(o.dateKey)) perDate.set(o.dateKey, new Map());
            const m = perDate.get(o.dateKey);
            o.employees.forEach(e => {
                if (!e.key) return;
                if (!m.has(e.key)) m.set(e.key, { emp: e, shifts: new Set() });
                m.get(e.key).shifts.add(shiftKeyOf(o));
            });
        });

        const dups = [];
        perDate.forEach((m, dateKey) => {
            m.forEach((v, key) => {
                if (v.shifts.size >= 2) {
                    dups.push({
                        dateKey, key,
                        name: v.emp.name, login: v.emp.login, id: v.emp.id,
                        shifts: Array.from(v.shifts)
                    });
                }
            });
        });
        return dups;
    }

    /* ======================================================
       パネル外枠
    ====================================================== */

    function ensurePanel() {
        let p = document.getElementById(PANEL_ID);
        if (p) return p;

        p = document.createElement('div');
        p.id = PANEL_ID;
        p.innerHTML = `
<div class="lsp-grip" title="ドラッグで幅変更 / ダブルクリックで既定幅"></div>
<div class="lsp-head">
    <span class="lsp-title">シフト集約</span>
    <span class="lsp-badge" data-site>-</span>
    <span class="lsp-spacer"></span>
    <button class="lsp-ico" data-act="refresh" title="再集計">&#8635;</button>
    <button class="lsp-ico" data-act="close" title="閉じる">&#10142;</button>
</div>
<div class="lsp-tabs">
    <button class="lsp-tab" data-tab="shift">シフト別</button>
    <button class="lsp-tab" data-tab="emp">出勤者</button>
    <button class="lsp-tab" data-tab="list">機会一覧</button>
    <button class="lsp-tab" data-tab="probe">調査</button>
    <button class="lsp-tab" data-tab="conf">設定</button>
</div>
<div class="lsp-body"></div>
<div class="lsp-foot"><span data-foot>-</span></div>`;

        document.documentElement.appendChild(p);

        p.querySelector('[data-act="refresh"]').addEventListener('click', () => refresh(true));
        p.querySelector('[data-act="close"]').addEventListener('click', () => setOpen(false));
        p.querySelectorAll('.lsp-tab').forEach(b => {
            b.addEventListener('click', () => {
                S.tab = b.dataset.tab;
                saveSettings();
                renderTabs();
                renderBody();
            });
        });

        setupResize(p);
        return p;
    }

    function setupResize(panel) {
        const grip = panel.querySelector('.lsp-grip');
        const onMove = e => {
            if (!resizing) return;
            S.width = clampWidth(window.innerWidth - e.clientX);
            applyWidth();
        };
        const onUp = () => {
            if (!resizing) return;
            resizing = false;
            document.documentElement.classList.remove('lsp-resizing');
            window.removeEventListener('pointermove', onMove);
            window.removeEventListener('pointerup', onUp);
            saveSettings();
        };
        grip.addEventListener('pointerdown', e => {
            e.preventDefault();
            resizing = true;
            document.documentElement.classList.add('lsp-resizing');
            window.addEventListener('pointermove', onMove);
            window.addEventListener('pointerup', onUp);
        });
        grip.addEventListener('dblclick', () => {
            S.width = WIDTH_DEFAULT;
            applyWidth();
            saveSettings();
        });
    }

    function ensureReopenButton() {
        let b = document.getElementById(REOPEN_ID);
        if (b) return b;
        b = document.createElement('button');
        b.id = REOPEN_ID;
        b.type = 'button';
        b.textContent = 'シフト集約';
        b.title = 'パネルを開く (Alt+P)';
        b.addEventListener('click', () => setOpen(true));
        document.documentElement.appendChild(b);
        return b;
    }

    function applyWidth() {
        S.width = clampWidth(S.width);
        document.documentElement.style.setProperty('--lsp-w', S.width + 'px');
    }

    function applySplit() {
        const html = document.documentElement;
        const on = isTargetRoute() && S.open;
        html.classList.toggle('lsp-split', on);
        html.classList.toggle('lsp-contain', on && S.containFixed);
        syncModalState();
    }

    // モーダル表示中は transform を外す（fixed の基準が body になるのを防ぐ）
    function syncModalState() {
        const open = !!visibleModal();
        document.documentElement.classList.toggle('lsp-modal', open);
    }

    function setOpen(open) {
        S.open = !!open;
        saveSettings();
        update();
    }

    function renderTabs() {
        const p = document.getElementById(PANEL_ID);
        if (!p) return;
        p.querySelectorAll('.lsp-tab').forEach(b => b.classList.toggle('on', b.dataset.tab === S.tab));
        const site = p.querySelector('[data-site]');
        if (site) site.textContent = currentSiteId();
    }

    function setFoot(text) {
        const p = document.getElementById(PANEL_ID);
        if (!p) return;
        const f = p.querySelector('[data-foot]');
        if (f) f.textContent = text;
    }

    function renderBody() {
        const p = document.getElementById(PANEL_ID);
        if (!p) return;
        const body = p.querySelector('.lsp-body');
        if (!body) return;
        const top = body.scrollTop;

        if (S.tab === 'conf') renderConf(body);
        else if (S.tab === 'probe') renderProbe(body);
        else if (S.tab === 'list') renderList(body);
        else if (S.tab === 'emp') renderEmp(body);
        else renderShift(body);

        body.scrollTop = top;
    }

    /* ======================================================
       共通パーツ
    ====================================================== */

    function noDataHTML() {
        return `
<div class="lsp-warn">
  一覧（<code>div.date_container</code>）が見つかりませんでした。<br>
  日付を指定して検索し、一覧が表示された状態で「&#8635;」を押してください。
</div>`;
    }

    // 日付・シフト基準・除外設定の共通コントロール
    function controlsHTML() {
        const days = DATA ? DATA.days : [];
        return `
<div class="lsp-row">
  <label class="lsp-lb">日付</label>
  <select data-date>
    <option value="ALL"${S.dateKey === 'ALL' ? ' selected' : ''}>全日付 (${days.length})</option>
    ${days.map(d => `<option value="${esc(d.dateKey)}"${S.dateKey === d.dateKey ? ' selected' : ''}>${esc(d.label || d.dateKey)}</option>`).join('')}
  </select>
  <label class="lsp-lb">シフト基準</label>
  <select data-shiftkey>
    <option value="range"${S.shiftKey === 'range' ? ' selected' : ''}>開始-終了</option>
    <option value="start"${S.shiftKey === 'start' ? ' selected' : ''}>開始のみ</option>
  </select>
</div>
<label class="lsp-chk"><input type="checkbox" data-excl${S.excludeCancelled ? ' checked' : ''}>取消・非表示の機会を除外</label>`;
    }

    function bindControls(body) {
        const d = body.querySelector('[data-date]');
        if (d) d.addEventListener('change', () => { S.dateKey = d.value; saveSettings(); renderBody(); });
        const sk = body.querySelector('[data-shiftkey]');
        if (sk) sk.addEventListener('change', () => { S.shiftKey = sk.value; saveSettings(); renderBody(); });
        const ex = body.querySelector('[data-excl]');
        if (ex) ex.addEventListener('change', () => { S.excludeCancelled = ex.checked; saveSettings(); renderBody(); });
    }

    function barHTML(a, b) {
        const r = pct(a, b);
        if (r == null) return '';
        const cls = r >= 100 ? 'ok' : (r < 50 ? 'low' : '');
        return `<div class="lsp-bar ${cls}" title="${r.toFixed(0)}%"><i style="width:${Math.min(100, r).toFixed(0)}%"></i></div>`;
    }

    function wgShort(set) {
        const arr = Array.from(set);
        if (!arr.length) return '-';
        if (arr.length <= 2) return arr.join(' / ');
        return `${arr.slice(0, 2).join(' / ')} +${arr.length - 2}`;
    }

    /* ======================================================
       シフト別タブ
    ====================================================== */

    function renderShift(body) {
        if (!DATA) { body.innerHTML = noDataHTML(); return; }

        const opps = targetOpportunities();
        const groups = aggregateShifts(opps);

        const tAcc = groups.reduce((s, g) => s + g.accepted, 0);
        const tHc = groups.reduce((s, g) => s + g.headcount, 0);
        const tShort = Math.max(0, tHc - tAcc);
        const rate = pct(tAcc, tHc);

        body.innerHTML = `
${controlsHTML()}

<div class="lsp-cards">
  <div class="lsp-card"><div class="k">シフト数</div><div class="v">${groups.length}</div></div>
  <div class="lsp-card"><div class="k">出勤者(承認済み)</div><div class="v">${fmtNum(tAcc)}</div></div>
  <div class="lsp-card"><div class="k">定員</div><div class="v">${fmtNum(tHc)}</div></div>
  <div class="lsp-card${tShort > 0 ? ' warn' : ''}"><div class="k">不足</div><div class="v">${fmtNum(tShort)}</div></div>
</div>

<div class="lsp-row">
  <span class="lsp-lb">充足率 ${rate == null ? '-' : rate.toFixed(1) + '%'}</span>
  <span style="flex:1 1 60px">${barHTML(tAcc, tHc)}</span>
  <button class="lsp-btn" data-act="copyshift">TSVコピー</button>
</div>

<table class="lsp-t">
  <thead><tr>
    <th>シフト</th><th>機会</th><th>出勤者</th><th>定員</th><th>不足</th><th>充足</th><th>ワークグループ</th>
  </tr></thead>
  <tbody>
    ${groups.map(g => {
            const short = Math.max(0, g.headcount - g.accepted);
            return `<tr>
      <td class="nw"><b>${esc(g.key)}</b>${S.dateKey === 'ALL' && g.dates.size > 1 ? `<div class="lsp-sub">${g.dates.size} 日分</div>` : ''}</td>
      <td class="num">${g.count}</td>
      <td class="num">${fmtNum(g.accepted)}${g.people.size && g.people.size !== g.accepted ? `<div class="lsp-sub">実${g.people.size}名</div>` : ''}</td>
      <td class="num">${fmtNum(g.headcount)}</td>
      <td class="num ${short > 0 ? 'short' : 'full'}">${short}</td>
      <td>${barHTML(g.accepted, g.headcount)}</td>
      <td>${esc(wgShort(g.workgroups))}</td>
    </tr>`;
        }).join('')}
  </tbody>
  <tfoot><tr>
    <td>合計</td>
    <td class="num">${opps.length}</td>
    <td class="num">${fmtNum(tAcc)}</td>
    <td class="num">${fmtNum(tHc)}</td>
    <td class="num">${tShort}</td>
    <td>${rate == null ? '-' : rate.toFixed(0) + '%'}</td>
    <td></td>
  </tr></tfoot>
</table>`;

        bindControls(body);
        body.querySelector('[data-act="copyshift"]').addEventListener('click', e => {
            const head = ['シフト', '機会数', '出勤者', '定員', '不足', '充足率', 'ワークグループ'].join('\t');
            const lines = groups.map(g => {
                const short = Math.max(0, g.headcount - g.accepted);
                const r = pct(g.accepted, g.headcount);
                return [g.key, g.count, g.accepted, g.headcount, short,
                        r == null ? '' : r.toFixed(1) + '%',
                        Array.from(g.workgroups).join(',')].join('\t');
            });
            copyText([head, ...lines].join('\n'));
            flash(e.target);
        });
    }

    /* ======================================================
       出勤者タブ
    ====================================================== */

    function renderEmp(body) {
        if (!DATA) { body.innerHTML = noDataHTML(); return; }

        const opps = targetOpportunities();
        const groups = aggregateShifts(opps);
        const dups = duplicatePeople(opps);
        const dupSet = new Set(dups.map(d => d.dateKey + '|' + d.key));

        let notice = '';
        if (DATA.employeeSource !== 'angular') {
            notice = `
<div class="lsp-err">
  承認済み従業員リストを取得できませんでした（AngularJS スコープ ${DATA.angular ? '検出済' : '未検出'} / 一致 ${DATA.scopeHit}/${DATA.scopeTried}）。<br>
  人数の集計はシフト別タブで確認できます。<b>調査タブ</b>の内容を共有いただければ、取得方法を合わせます。
</div>`;
        } else if (!DATA.hasEmployeeNames) {
            notice = `<div class="lsp-warn">承認済み従業員は 0 名でした（配列は取得できています）。</div>`;
        }

        // 名前 / IDが埋まっていない人数
        let missing = 0, total = 0;
        groups.forEach(g => g.employees.forEach(e => {
            total++;
            if (!e.name || !e.id) missing++;
        }));

        const allPeople = new Set();
        groups.forEach(g => g.people.forEach(k => allPeople.add(k)));

        body.innerHTML = `
${controlsHTML()}
${notice}

<div class="lsp-cards">
  <div class="lsp-card"><div class="k">シフト数</div><div class="v">${groups.length}</div></div>
  <div class="lsp-card"><div class="k">登録のべ人数</div><div class="v">${fmtNum(groups.reduce((s, g) => s + g.employees.length, 0))}</div></div>
  <div class="lsp-card"><div class="k">実人数</div><div class="v">${fmtNum(allPeople.size)}</div></div>
  <div class="lsp-card${dups.length ? ' warn' : ''}"><div class="k">重複</div><div class="v">${dups.length}</div></div>
</div>

${groups.map(g => {
            const open = expanded.has(g.key);
            return `
<div class="lsp-shift">
  <div class="hd" data-shift="${esc(g.key)}">
    <span>${open ? '&#9662;' : '&#9656;'}</span>
    <span class="t">${esc(g.key)}</span>
    <span class="c">${g.employees.length}名 / 定員 ${fmtNum(g.headcount)}</span>
    <span class="sp"></span>
  </div>
  ${open ? `<div class="bd">
    ${g.employees.length ? `<table class="lsp-t">
      <thead><tr><th style="width:24px">#</th><th>従業員名</th><th>従業員ログイン</th><th>従業員ID</th></tr></thead>
      <tbody>${g.employees.map((e, i) => {
                const dup = dupSet.has(e.dateKey + '|' + e.key);
                return `<tr class="${dup ? 'dup' : ''}" title="${esc(e.dateKey)}${dup ? ' / 同日に複数シフトへ登録' : ''}">
          <td class="num">${i + 1}</td>
          <td>${esc(e.name) || '<span class="lsp-note">-</span>'}</td>
          <td class="nw">${esc(e.login) || '<span class="lsp-note">-</span>'}</td>
          <td class="nw">${esc(e.id) || '<span class="lsp-note">-</span>'}</td>
        </tr>`;
            }).join('')}</tbody>
    </table>` : '<div class="lsp-note">登録者なし</div>'}
  </div>` : ''}
</div>`;
        }).join('')}

${dups.length ? `
<h4>同一日に複数シフトへ登録</h4>
<table class="lsp-t">
  <thead><tr><th>日付</th><th>従業員名</th><th>ログイン</th><th>従業員ID</th><th>シフト</th></tr></thead>
  <tbody>${dups.map(d => `<tr>
    <td class="nw">${esc(d.dateKey)}</td>
    <td>${esc(d.name) || '-'}</td>
    <td class="nw">${esc(d.login) || '-'}</td>
    <td class="nw">${esc(d.id) || '-'}</td>
    <td>${esc(d.shifts.join(' , '))}</td>
  </tr>`).join('')}</tbody>
</table>` : ''}

<div class="lsp-actions">
  ${total && missing ? `
  <div class="lsp-warn" style="margin-bottom:8px">
    従業員名・従業員IDは行データに含まれていないため、詳細モーダルから回収します（${missing}/${total} 件が未取得）。<br>
    下のボタンで詳細を順番に開いて閉じ、ログインで突き合わせてキャッシュします。
  </div>` : ''}
  <div class="lsp-row">
    <button class="lsp-btn" data-act="expand">すべて展開</button>
    <button class="lsp-btn" data-act="collapse">すべて閉じる</button>
    <button class="lsp-btn primary" data-act="copyemp">名簿TSVコピー</button>
  </div>
  <div class="lsp-row">
    <button class="lsp-btn" data-act="hday" title="各日付の「機会時間の概要」を開いて回収">日別サマリーから取得</button>
    <button class="lsp-btn" data-act="hopp" title="各機会の人員数セルを開いて回収">機会ごとに取得</button>
    <button class="lsp-btn" data-act="habort">中断</button>
  </div>
  <div class="lsp-note">名簿キャッシュ ${DIR.size} 件 <span data-hstat></span>
    ${DIR.size ? '<button class="lsp-btn" data-act="hclear" style="margin-left:6px">クリア</button>' : ''}
  </div>
</div>`;

        bindControls(body);

        body.querySelectorAll('[data-shift]').forEach(hd => {
            hd.addEventListener('click', () => {
                const k = hd.dataset.shift;
                if (expanded.has(k)) expanded.delete(k); else expanded.add(k);
                renderBody();
            });
        });
        body.querySelector('[data-act="expand"]').addEventListener('click', () => {
            groups.forEach(g => expanded.add(g.key));
            renderBody();
        });
        body.querySelector('[data-act="collapse"]').addEventListener('click', () => {
            expanded.clear();
            renderBody();
        });
        body.querySelector('[data-act="hday"]').addEventListener('click', () => harvestByDaySummary());
        body.querySelector('[data-act="hopp"]').addEventListener('click', () => harvestByOpportunity());
        body.querySelector('[data-act="habort"]').addEventListener('click', () => {
            harvestAbort = true;
            harvestStatus('中断要求');
        });
        const hc = body.querySelector('[data-act="hclear"]');
        if (hc) hc.addEventListener('click', () => {
            DIR.clear();
            saveDir();
            lastSignature = '';
            refresh(true);
        });

        body.querySelector('[data-act="copyemp"]').addEventListener('click', e => {
            const head = ['日付', 'シフト', '従業員名', '従業員ログイン', '従業員ID', 'ワークグループ'].join('\t');
            const lines = [];
            groups.forEach(g => g.employees.forEach(emp => {
                lines.push([emp.dateKey, g.key, emp.name || '', emp.login || '', emp.id || '',
                            (emp.workgroups || []).join(',')].join('\t'));
            }));
            copyText([head, ...lines].join('\n'));
            flash(e.target);
        });
    }

    /* ======================================================
       機会一覧タブ
    ====================================================== */

    function renderList(body) {
        if (!DATA) { body.innerHTML = noDataHTML(); return; }

        const opps = targetOpportunities().slice().sort((a, b) => {
            if (a.dateKey !== b.dateKey) return a.dateKey < b.dateKey ? -1 : 1;
            return timeToMin(a.start) - timeToMin(b.start);
        });

        body.innerHTML = `
${controlsHTML()}
<div class="lsp-row">
  <span class="lsp-lb">${opps.length} 件</span>
  <button class="lsp-btn" data-act="copylist">TSVコピー</button>
</div>
<table class="lsp-t">
  <thead><tr>
    <th>日付</th><th>開始</th><th>終了</th><th>人員</th><th>タイプ</th><th>WG</th><th>サインアップ終了</th>
  </tr></thead>
  <tbody>
    ${opps.map(o => `<tr>
      <td class="nw">${esc(o.dateKey)}</td>
      <td class="nw">${esc(o.start)}</td>
      <td class="nw">${esc(o.end)}</td>
      <td class="num${Number.isFinite(o.accepted) && Number.isFinite(o.headcount) && o.accepted < o.headcount ? ' short' : ' full'}">${o.accepted}/${o.headcount}</td>
      <td class="nw">${esc(o.type)}${o.cancelled ? ' <span style="color:#d13212">取消</span>' : ''}</td>
      <td title="${esc(o.workgroups.join(' / '))}">${esc(o.workgroups.length > 2 ? o.workgroups.slice(0, 2).join(' / ') + ' +' + (o.workgroups.length - 2) : o.workgroups.join(' / '))}</td>
      <td class="nw">${esc(o.signupEnd)}</td>
    </tr>`).join('')}
  </tbody>
</table>`;

        bindControls(body);
        body.querySelector('[data-act="copylist"]').addEventListener('click', e => {
            const head = ['日付', '開始', '終了', '承認済み', '定員', 'タイプ', 'ワークグループ', 'サインアップ開始', 'サインアップ終了', '取消'].join('\t');
            const lines = opps.map(o => [
                o.dateKey, o.start, o.end, o.accepted, o.headcount, o.type,
                o.workgroups.join(','), o.signupStart, o.signupEnd, o.cancelled ? 'Y' : ''
            ].join('\t'));
            copyText([head, ...lines].join('\n'));
            flash(e.target);
        });
    }

    /* ======================================================
       調査タブ
    ====================================================== */

    function renderProbe(body) {
        const ng = getAngular();
        const info = {
            url: location.href,
            siteId: currentSiteId(),
            capturedAt: new Date().toISOString(),
            dateContainers: document.querySelectorAll('div.date_container').length,
            opportunityRows: document.querySelectorAll('tr[opportunity-id]').length,
            angularDetected: !!ng,
            angularVersion: ng && ng.version ? ng.version.full : null,
            scopeMatched: DATA ? `${DATA.scopeHit}/${DATA.scopeTried}` : null,
            employeeSource: DATA ? DATA.employeeSource : null,
            // singleOpportunity が持つプロパティ名（値は出しません）
            opportunityKeys: DATA ? DATA.sampleOppKeys : null,
            // acceptedEmployees[0] が持つプロパティ名（値は出しません）
            acceptedEmployeeKeys: DATA ? DATA.sampleEmpKeys : null,
            days: DATA ? DATA.days.map(d => ({
                dateKey: d.dateKey,
                label: d.label,
                rows: d.opportunities.length,
                sample: d.opportunities.slice(0, 2).map(o => ({
                    start: o.start, end: o.end,
                    accepted: o.accepted, headcount: o.headcount,
                    type: o.type, workgroups: o.workgroups.length,
                    employees: Array.isArray(o.employees) ? o.employees.length : null
                }))
            })) : null
        };
        const json = JSON.stringify(info, null, 2);

        // 承認済み従業員 1 件の生データ（キーと値の対応を確認する用途）
        let sampleRaw = null;
        if (DATA) {
            for (const d of DATA.days) {
                const hit = d.opportunities.find(o => Array.isArray(o.employees) && o.employees.length);
                if (hit) { sampleRaw = hit.employees[0].raw; break; }
            }
        }
        let sampleJson = '';
        if (sampleRaw) {
            try { sampleJson = JSON.stringify(sampleRaw, null, 2); }
            catch (e) { sampleJson = '(循環参照のため出力できません)'; }
        }

        body.innerHTML = `
<div class="lsp-note">
  取得状況と <code>singleOpportunity</code> / <code>acceptedEmployees</code> のプロパティ名を書き出します。
  従業員名や従業員IDが「-」になる場合は、この JSON を共有してください。
</div>
<div class="lsp-row" style="margin-top:8px">
  <button class="lsp-btn primary" data-act="copyjson">JSONをコピー</button>
  <button class="lsp-btn" data-act="re">再取得</button>
</div>
<pre class="lsp-pre">${esc(json)}</pre>

<h4>承認済み従業員 1 件の生データ</h4>
${sampleRaw ? `
<div class="lsp-note">氏名などの個人情報を含みます。どのキーが名前 / ログイン / IDに対応するか確認する用途です。</div>
<div class="lsp-row" style="margin-top:6px">
  <button class="lsp-btn" data-act="copyemp1">この1件をコピー</button>
</div>
<pre class="lsp-pre">${esc(sampleJson)}</pre>`
            : '<div class="lsp-note">承認済み従業員が 0 名か、取得できていません。</div>'}`;

        body.querySelector('[data-act="copyjson"]').addEventListener('click', e => { copyText(json); flash(e.target, 'コピー済'); });
        body.querySelector('[data-act="re"]').addEventListener('click', () => refresh(true));
        const c1 = body.querySelector('[data-act="copyemp1"]');
        if (c1) c1.addEventListener('click', e => { copyText(sampleJson); flash(e.target); });
    }

    /* ======================================================
       設定タブ
    ====================================================== */

    function renderConf(body) {
        body.innerHTML = `
<h4>表示</h4>
<div class="lsp-row">
  <label class="lsp-lb">パネル幅</label>
  <input type="text" data-w value="${S.width}" style="flex:0 0 70px">
  <span class="lsp-lb">px</span>
  <button class="lsp-btn" data-act="applyw">適用</button>
  <button class="lsp-btn" data-act="resetw">既定 (${WIDTH_DEFAULT})</button>
</div>
<label class="lsp-chk"><input type="checkbox" data-c="containFixed"${S.containFixed ? ' checked' : ''}>すべての固定要素を分割幅に収める（実験）</label>
<div class="lsp-note">
  Bootstrap のモーダル・固定ナビは既定で分割幅に収まります。<br>
  この実験オプションは body に <code>transform</code> を当てるため、<code>position:fixed</code> の基準がずれて
  モーダルが画面外に出ることがあります（モーダル表示中は自動解除します）。通常は不要です。
</div>
<label class="lsp-chk"><input type="checkbox" data-c="allRoutes"${S.allRoutes ? ' checked' : ''}>${esc(ROUTE_KEYWORD)} 以外のページでも表示</label>

<h4>更新</h4>
<label class="lsp-chk"><input type="checkbox" data-c="autoRefresh"${S.autoRefresh ? ' checked' : ''}>ページの変化を検知して自動で再集計</label>

<h4>ショートカット</h4>
<div class="lsp-note">Alt + P：パネルの開閉</div>

<h4>その他</h4>
<div class="lsp-row"><button class="lsp-btn" data-act="reset">設定を初期化</button></div>
<div class="lsp-note">version ${esc((typeof GM_info !== 'undefined' && GM_info.script) ? GM_info.script.version : '2.0')}</div>`;

        const wInput = body.querySelector('[data-w]');
        body.querySelector('[data-act="applyw"]').addEventListener('click', () => {
            S.width = clampWidth(parseInt(wInput.value, 10));
            applyWidth(); saveSettings(); wInput.value = S.width;
        });
        body.querySelector('[data-act="resetw"]').addEventListener('click', () => {
            S.width = WIDTH_DEFAULT;
            applyWidth(); saveSettings(); wInput.value = S.width;
        });
        body.querySelectorAll('[data-c]').forEach(cb => {
            cb.addEventListener('change', () => {
                S[cb.dataset.c] = cb.checked;
                saveSettings();
                setupObserver();
                update();
            });
        });
        body.querySelector('[data-act="reset"]').addEventListener('click', () => {
            S = { ...DEFAULTS };
            saveSettings();
            applyWidth(); applySplit(); renderTabs(); renderBody();
        });
    }

    /* ======================================================
       更新フロー
    ====================================================== */

    function refresh(force) {
        const d = extractData();
        const sig = signatureOf(d);
        if (!force && sig === lastSignature) return;
        lastSignature = sig;
        DATA = d;

        // 選択中の日付が消えた場合は全日付に戻す
        if (DATA && S.dateKey !== 'ALL' && !DATA.days.some(x => x.dateKey === S.dateKey)) {
            S.dateKey = 'ALL';
            saveSettings();
        }

        renderBody();

        if (!d) {
            setFoot(`一覧を検出できません / 確認 ${nowLabel()}`);
            return;
        }
        const rows = d.days.reduce((s, x) => s + x.opportunities.length, 0);
        const emp = d.employeeSource === 'angular' ? '名簿OK' : '名簿NG';
        setFoot(`${d.days.length}日 / ${rows}機会 / ${emp} / 更新 ${nowLabel()}`);
    }

    function update() {
        const active = isTargetRoute();
        applyWidth();
        applySplit();

        if (!active) {
            const p = document.getElementById(PANEL_ID);
            if (p) p.remove();
            const b = document.getElementById(REOPEN_ID);
            if (b) b.remove();
            return;
        }

        if (!S.open) {
            const p = document.getElementById(PANEL_ID);
            if (p) p.remove();
            ensureReopenButton();
            return;
        }

        const b = document.getElementById(REOPEN_ID);
        if (b) b.remove();

        ensurePanel();
        renderTabs();
        refresh(true);
    }

    function scheduleRefresh() {
        if (!S.autoRefresh || harvesting) return;
        clearTimeout(refreshTimer);
        refreshTimer = setTimeout(() => {
            if (harvesting) return;
            if (!isTargetRoute() || !S.open) return;
            if (S.tab === 'conf' || S.tab === 'probe') return;
            refresh(false);
        }, 800);
    }

    // 利用者が手動で詳細モーダルを開いたときも名簿を取り込む
    let passiveTimer = null;
    function schedulePassiveScrape() {
        if (harvesting) return;
        clearTimeout(passiveTimer);
        passiveTimer = setTimeout(() => {
            const m = visibleModal();
            if (!m) return;
            const tables = findEmployeeTables(m);
            if (!tables.length) return;
            let added = 0;
            tables.forEach(t => { added += scrapeDirectory(t).added; });
            if (added) {
                saveDir();
                lastSignature = '';
                refresh(true);
            }
        }, 500);
    }

    function setupObserver() {
        if (observer) { observer.disconnect(); observer = null; }
        if (!S.autoRefresh) return;
        observer = new MutationObserver(muts => {
            if (resizing) return;
            for (const m of muts) {
                if (m.target && m.target.closest && m.target.closest('#' + PANEL_ID)) continue;
                syncModalState();
                schedulePassiveScrape();
                scheduleRefresh();
                return;
            }
        });
        observer.observe(document.body, { childList: true, subtree: true, characterData: true });
    }

    /* ======================================================
       起動
    ====================================================== */

    function init() {
        injectStyle();
        loadDir();
        saveSettings();      // 移行フラグを確定させる
        applyWidth();
        update();
        setupObserver();

        window.addEventListener('hashchange', () => setTimeout(update, 150));
        window.addEventListener('resize', () => applyWidth());

        // 自動再集計を切っていてもモーダル状態は追従させる
        document.addEventListener('click', () => setTimeout(syncModalState, 300), true);
        document.addEventListener('keyup', e => {
            if (e.key === 'Escape') setTimeout(syncModalState, 300);
        }, true);
        window.addEventListener('keydown', e => {
            if (e.altKey && !e.ctrlKey && !e.shiftKey && (e.key === 'p' || e.key === 'P')) {
                e.preventDefault();
                setOpen(!S.open);
            }
        });

        ['pushState', 'replaceState'].forEach(fn => {
            const orig = history[fn];
            history[fn] = function () {
                const r = orig.apply(this, arguments);
                setTimeout(update, 150);
                return r;
            };
        });
        window.addEventListener('popstate', () => setTimeout(update, 150));
    }

    if (document.body) init();
    else document.addEventListener('DOMContentLoaded', init, { once: true });
})();
