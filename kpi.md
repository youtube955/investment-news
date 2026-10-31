---
layout: page
title: KPI
permalink: /kpi/
---

<style>
.kpi-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(170px, 1fr)); gap: 12px; margin-bottom: 32px; }
.kpi-tile { border: 1px solid #e5e5e5; border-radius: 8px; padding: 14px 16px; min-width: 0; }
.kpi-label, .kpi-sub, .kpi-note { font-size: 13px; color: #6b6b6b; }
.kpi-value { font-size: 24px; font-weight: 700; font-variant-numeric: tabular-nums; }
.kpi-delta { font-size: 13px; font-variant-numeric: tabular-nums; }
.kpi-delta.up { color: #d64545; }
.kpi-delta.down { color: #2f7de1; }
.kpi-delta.flat { color: #6b6b6b; }
.kpi-spark { display: block; width: 100%; height: 30px; margin-top: 8px; }
.kpi-spark polyline { fill: none; stroke: #41c9b4; stroke-width: 1.5; vector-effect: non-scaling-stroke; }
</style>

<div id="kpi-root">
  <p class="kpi-note">集計中…</p>
</div>

<script id="kpi-posts" type="application/json">
[{% for post in site.posts %}{"date":{{ post.date | date: "%Y-%m-%d" | jsonify }},"text":{{ post.content | strip_html | jsonify }}}{% unless forloop.last %},{% endunless %}{% endfor %}]
</script>

<script>
(function () {
  var posts = JSON.parse(document.getElementById('kpi-posts').textContent);
  var root = document.getElementById('kpi-root');
  if (!posts.length) { root.innerHTML = '<p class="kpi-note">記事がまだありません。</p>'; return; }

  // 本文から主要指標を抽出（2種類の記事フォーマットに対応）
  var patterns = {
    nikkei: /日経平均(?:株価)?[^\d]*?([\d,]+(?:\.\d+)?)/,
    usdjpy: /ドル円[^\d]*?([\d,]+(?:\.\d+)?)/,
    nasdaq: /NASDAQ[^\d]*?([\d,]+(?:\.\d+)?)/,
    sp500: /S&(?:amp;)?P\s*500[^\d]*?([\d,]+(?:\.\d+)?)/
  };
  function parse(text) {
    var out = {};
    for (var k in patterns) {
      var m = text.match(patterns[k]);
      if (m) out[k] = parseFloat(m[1].replace(/,/g, ''));
    }
    return out;
  }

  // site.posts は新しい順。日付ごとに最新の記事の値を採用
  var byDay = {};
  var days = [];
  posts.forEach(function (p) {
    if (!byDay[p.date]) { byDay[p.date] = parse(p.text); days.push(p.date); }
  });

  var latestDay = days[0];
  var latest = byDay[latestDay];
  var prev = byDay[days[1]] || {};
  var month = latestDay.slice(0, 7);
  var monthCount = posts.filter(function (p) { return p.date.slice(0, 7) === month; }).length;
  var cutoff = new Date(latestDay + 'T00:00:00');
  cutoff.setDate(cutoff.getDate() - 29);
  var last30 = posts.filter(function (p) { return new Date(p.date + 'T00:00:00') >= cutoff; }).length;

  function fmt(v, digits) {
    return v == null ? '–' : v.toLocaleString('ja-JP', { minimumFractionDigits: digits, maximumFractionDigits: digits });
  }
  function delta(cur, before, digits) {
    if (cur == null || before == null) return '';
    var d = cur - before;
    var pct = before ? (d / before) * 100 : 0;
    var cls = d > 0 ? 'up' : d < 0 ? 'down' : 'flat';
    var sign = d > 0 ? '+' : '';
    return '<span class="kpi-delta ' + cls + '">' + sign + fmt(d, digits) + '（' + sign + pct.toFixed(2) + '%）</span>';
  }
  function spark(key) {
    var vals = days.slice(0, 30).reverse().map(function (d) { return byDay[d][key]; }).filter(function (v) { return v != null; });
    if (vals.length < 2) return '';
    var min = Math.min.apply(null, vals), max = Math.max.apply(null, vals), span = max - min || 1;
    var pts = vals.map(function (v, i) {
      return (i / (vals.length - 1) * 100).toFixed(1) + ',' + (28 - (v - min) / span * 26).toFixed(1);
    }).join(' ');
    return '<svg class="kpi-spark" viewBox="0 0 100 30" preserveAspectRatio="none" aria-hidden="true"><polyline points="' + pts + '"/></svg>';
  }
  function tile(label, value, sub) {
    return '<div class="kpi-tile"><div class="kpi-label">' + label + '</div><div class="kpi-value">' + value + '</div>' + (sub || '') + '</div>';
  }
  function marketTile(label, key, digits, unit) {
    return tile(label, fmt(latest[key], digits) + unit, delta(latest[key], prev[key], digits) + spark(key));
  }

  root.innerHTML =
    '<p class="kpi-note">最終更新: ' + latestDay + '（前日比は ' + (days[1] || '–') + ' との比較・グラフは直近30営業日）</p>' +
    '<h2>サイト運用</h2><div class="kpi-grid">' +
      tile('累計記事数', posts.length.toLocaleString('ja-JP') + '本') +
      tile('今月の記事数', monthCount + '本', '<div class="kpi-sub">' + month + '</div>') +
      tile('直近30日の記事数', last30 + '本', '<div class="kpi-sub">1日平均 ' + (last30 / 30).toFixed(1) + '本</div>') +
      tile('更新日数', days.length + '日', '<div class="kpi-sub">' + days[days.length - 1] + ' 〜</div>') +
    '</div>' +
    '<h2>主要指標</h2><div class="kpi-grid">' +
      marketTile('日経平均', 'nikkei', 2, '円') +
      marketTile('ドル円', 'usdjpy', 2, '円') +
      marketTile('NASDAQ', 'nasdaq', 2, '') +
      marketTile('S&amp;P500', 'sp500', 2, '') +
    '</div>' +
    '<p class="kpi-note">※ 記事本文の数値から自動集計しています。投資助言ではありません。</p>';
})();
</script>
