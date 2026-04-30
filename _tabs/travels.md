---
title: Travels
icon: fas fa-plane
order: 4
---

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<style>
#flight-map { height: 520px; width: 100%; border-radius: 12px; margin-bottom: 1rem; z-index: 1; }
@media (max-width: 849px) { #flight-map { height: 420px; } }
@media (max-width: 575px) { #flight-map { height: 340px; } }
.stat-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin-bottom: 0.5rem; }
@media (min-width: 576px) { .stat-grid { grid-template-columns: repeat(6, 1fr); } }
.stat-card { background: var(--card-bg, #f8f9fa); border: 1px solid var(--border-color, #e9ecef); border-radius: 10px; padding: 12px 8px; text-align: center; }
.stat-card .value { font-size: 1.5rem; font-weight: 700; color: var(--heading-color, #2c3e50); }
@media (min-width: 576px) { .stat-card .value { font-size: 1.8rem; } }
.stat-card .label { font-size: 0.72rem; color: var(--text-muted-color, #6c757d); margin-top: 2px; }
.airline-bar { display: flex; align-items: center; margin-bottom: 6px; }
.airline-bar .name { min-width: 130px; width: 130px; font-size: 0.82rem; text-align: right; padding-right: 10px; color: var(--text-color); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.airline-bar .bar { height: 20px; border-radius: 4px; min-width: 2px; transition: width 0.6s ease; }
.airline-bar .count { font-size: 0.78rem; padding-left: 8px; color: var(--text-muted-color, #6c757d); }
.dest-table { width: 100%; border-collapse: collapse; margin-top: 1rem; font-size: 0.85rem; }
.dest-table th, .dest-table td { padding: 8px 12px; text-align: left; border-bottom: 1px solid var(--border-color, #e9ecef); }
.dest-table th { color: var(--text-muted-color); font-weight: 600; }
.dest-table em { font-style: normal; font-size: 0.7rem; background: var(--border-color, #e9ecef); color: var(--text-muted-color); padding: 1px 6px; border-radius: 3px; margin-left: 4px; }
.legend-row { display: flex; gap: 16px; flex-wrap: wrap; margin: 8px 0 16px; font-size: 0.8rem; color: var(--text-muted-color); }
.legend-item { display: flex; align-items: center; gap: 4px; }
.legend-line { width: 24px; height: 3px; border-radius: 2px; }
</style>

2015년 첫 해외여행(베트남 하노이)부터 현재까지의 비행 기록입니다.

<div class="stat-grid">
  <div class="stat-card"><div class="value">77</div><div class="label">Total Flights</div></div>
  <div class="stat-card"><div class="value">33</div><div class="label">Trips</div></div>
  <div class="stat-card"><div class="value">11</div><div class="label">Years Active</div></div>
  <div class="stat-card"><div class="value">22</div><div class="label">Airports</div></div>
  <div class="stat-card"><div class="value">18</div><div class="label">Airlines</div></div>
  <div class="stat-card"><div class="value">10</div><div class="label">Countries</div></div>
</div>

<div id="flight-map" style="margin-top:1rem;"></div>

<div class="legend-row">
  <span class="legend-item"><span class="legend-line" style="background:#e74c3c"></span> International</span>
  <span class="legend-item"><span class="legend-line" style="background:#3498db"></span> Domestic</span>
  <span class="legend-item"><span class="legend-line" style="background:none;border:1.5px dashed #f39c12"></span> Planned</span>
</div>

<script>
(function() {
  var airports = {
    ICN: { lat: 37.4602, lng: 126.4407, name: 'Incheon', country: 'KR' },
    GMP: { lat: 37.5583, lng: 126.7906, name: 'Gimpo', country: 'KR' },
    CJU: { lat: 33.5104, lng: 126.4914, name: 'Jeju', country: 'KR' },
    PUS: { lat: 35.1796, lng: 128.9382, name: 'Busan', country: 'KR' },
    NRT: { lat: 35.7720, lng: 140.3929, name: 'Tokyo Narita', country: 'JP' },
    HND: { lat: 35.5494, lng: 139.7798, name: 'Tokyo Haneda', country: 'JP' },
    KIX: { lat: 34.4347, lng: 135.2440, name: 'Osaka Kansai', country: 'JP' },
    FUK: { lat: 33.5859, lng: 130.4511, name: 'Fukuoka', country: 'JP' },
    CTS: { lat: 42.7752, lng: 141.6925, name: 'Sapporo', country: 'JP' },
    NGO: { lat: 34.8584, lng: 136.8049, name: 'Nagoya', country: 'JP' },
    BKK: { lat: 13.6900, lng: 100.7501, name: 'Bangkok', country: 'TH' },
    HKT: { lat: 8.1132, lng: 98.3169, name: 'Phuket', country: 'TH' },
    SIN: { lat: 1.3644, lng: 103.9915, name: 'Singapore', country: 'SG' },
    CEB: { lat: 10.3075, lng: 123.9794, name: 'Cebu', country: 'PH' },
    TPE: { lat: 25.0797, lng: 121.2342, name: 'Taipei', country: 'TW' },
    HKG: { lat: 22.3080, lng: 113.9185, name: 'Hong Kong', country: 'HK' },
    HAN: { lat: 21.2187, lng: 105.8042, name: 'Hanoi', country: 'VN' },
    DPS: { lat: -8.7467, lng: 115.1668, name: 'Bali', country: 'ID' },
    SPN: { lat: 15.1190, lng: 145.7295, name: 'Saipan', country: 'MP' },
    DLC: { lat: 38.9657, lng: 121.5386, name: 'Dalian', country: 'CN' },
    SYD: { lat: -33.9461, lng: 151.1772, name: 'Sydney', country: 'AU' },
    PVG: { lat: 31.1443, lng: 121.8083, name: 'Shanghai Pudong', country: 'CN' }
  };

  var routes = [
    { from: 'ICN', to: 'HAN', flights: 2, years: '2015', type: 'intl', flt: 'VN415, VN416' },
    { from: 'ICN', to: 'NRT', flights: 4, years: '2024-2026', type: 'intl', flt: 'OZ110, WE501/502' },
    { from: 'GMP', to: 'HND', flights: 1, years: '2022', type: 'intl', flt: 'KE2101' },
    { from: 'ICN', to: 'KIX', flights: 6, years: '2016-2024', type: 'intl', flt: 'MM006/009, OZ114/113, LJ237' },
    { from: 'ICN', to: 'FUK', flights: 6, years: '2017-2019', type: 'intl', flt: 'ZE641/642, LJ227/226, KE789/782' },
    { from: 'ICN', to: 'CTS', flights: 6, years: '2017-2023', type: 'intl', flt: 'TW251/252, LJ231/232, KE769/770' },
    { from: 'ICN', to: 'NGO', flights: 6, years: '2018-2025', type: 'intl', flt: '7C1602/1601, LJ348, OZ122/123' },
    { from: 'ICN', to: 'BKK', flights: 6, years: '2023-2025', type: 'intl', flt: 'YP601/602, KE657/652, TG659/BR068' },
    { from: 'ICN', to: 'HKT', flights: 2, years: '2024', type: 'intl', flt: 'KE663/664' },
    { from: 'ICN', to: 'SIN', flights: 3, years: '2026', type: 'intl', flt: 'OZ751, SQ601(p), SQ608(p)' },
    { from: 'ICN', to: 'CEB', flights: 4, years: '2019', type: 'intl', flt: 'LJ025/026, Z29047/Z27046' },
    { from: 'ICN', to: 'TPE', flights: 4, years: '2018-2019', type: 'intl', flt: 'CI161/162, KE691/694' },
    { from: 'ICN', to: 'HKG', flights: 2, years: '2019', type: 'intl', flt: 'ZE931, RS502' },
    { from: 'ICN', to: 'DPS', flights: 2, years: '2025', type: 'intl', flt: 'KE633/634' },
    { from: 'ICN', to: 'SPN', flights: 2, years: '2017', type: 'intl', flt: 'OZ625, OZ626' },
    { from: 'BKK', to: 'TPE', flights: 1, years: '2025', type: 'intl', flt: 'BR068' },
    { from: 'TPE', to: 'FUK', flights: 1, years: '2025', type: 'intl', flt: 'BR106' },
    { from: 'FUK', to: 'ICN', flights: 1, years: '2025', type: 'intl', flt: 'OZ135' },
    { from: 'SIN', to: 'PVG', flights: 1, years: '2026', type: 'planned', flt: 'SQ826' },
    { from: 'PVG', to: 'ICN', flights: 1, years: '2026', type: 'planned', flt: 'OZ362' },
    { from: 'ICN', to: 'DLC', flights: 2, years: '2026', type: 'intl', flt: 'CZ676/675' },
    { from: 'SIN', to: 'SYD', flights: 2, years: '2026', type: 'planned', flt: 'SQ231, SQ222' },
    { from: 'GMP', to: 'CJU', flights: 6, years: '2022-2025', type: 'domestic', flt: 'TW705, KE1211/1276, KE1045/1324, TW711/730, KE1361/1344' },
    { from: 'GMP', to: 'PUS', flights: 3, years: '2018-2025', type: 'domestic', flt: 'BX8809, BX8807/8826' },
    { from: 'PUS', to: 'CJU', flights: 2, years: '2021', type: 'domestic', flt: 'BX' }
  ];

  function arcPoints(from, to, n) {
    var pts = [];
    var toRad = Math.PI / 180, toDeg = 180 / Math.PI;
    var lat1 = from.lat * toRad, lng1 = from.lng * toRad;
    var lat2 = to.lat * toRad, lng2 = to.lng * toRad;
    var d = 2 * Math.asin(Math.sqrt(
      Math.pow(Math.sin((lat1 - lat2) / 2), 2) +
      Math.cos(lat1) * Math.cos(lat2) * Math.pow(Math.sin((lng1 - lng2) / 2), 2)
    ));
    if (d < 0.0001) return [[from.lat, from.lng], [to.lat, to.lng]];
    for (var i = 0; i <= n; i++) {
      var f = i / n;
      var A = Math.sin((1 - f) * d) / Math.sin(d);
      var B = Math.sin(f * d) / Math.sin(d);
      var x = A * Math.cos(lat1) * Math.cos(lng1) + B * Math.cos(lat2) * Math.cos(lng2);
      var y = A * Math.cos(lat1) * Math.sin(lng1) + B * Math.cos(lat2) * Math.sin(lng2);
      var z = A * Math.sin(lat1) + B * Math.sin(lat2);
      pts.push([Math.atan2(z, Math.sqrt(x * x + y * y)) * toDeg, Math.atan2(y, x) * toDeg]);
    }
    return pts;
  }

  var map = L.map('flight-map', { zoomControl: true, scrollWheelZoom: false })
    .setView([25, 120], 4);

  L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
    attribution: '&copy; OpenStreetMap &copy; CARTO',
    maxZoom: 18
  }).addTo(map);

  var colors = { intl: '#e74c3c', domestic: '#3498db', planned: '#f39c12' };

  routes.forEach(function(r) {
    var from = airports[r.from], to = airports[r.to];
    var pts = arcPoints(from, to, 40);
    var weight = Math.min(1 + r.flights * 0.5, 4);
    var opts = {
      color: colors[r.type],
      weight: weight,
      opacity: r.type === 'planned' ? 0.5 : 0.7,
      dashArray: r.type === 'planned' ? '6 4' : null
    };
    var line = L.polyline(pts, opts).addTo(map);
    line.bindPopup(
      '<b>' + r.from + ' &rarr; ' + r.to + '</b><br>' +
      from.name + ' &rarr; ' + to.name + '<br>' +
      'Flights: ' + r.flights + '<br>' +
      'Years: ' + r.years + '<br>' +
      '<span style="color:#999">' + r.flt + '</span>'
    );
  });

  Object.keys(airports).forEach(function(code) {
    var ap = airports[code];
    var isHome = code === 'ICN' || code === 'GMP';
    var marker = L.circleMarker([ap.lat, ap.lng], {
      radius: isHome ? 7 : 5,
      fillColor: isHome ? '#2ecc71' : '#ecf0f1',
      color: '#2c3e50',
      weight: 1.5,
      fillOpacity: 0.9
    }).addTo(map);
    marker.bindTooltip(code, { permanent: false, direction: 'top', offset: [0, -8] });
    marker.bindPopup('<b>' + code + '</b> &mdash; ' + ap.name);
  });
})();
</script>

## Airlines

<div id="airline-chart"></div>

<script>
(function() {
  var airlines = [
    { name: 'Korean Air', code: 'KE', count: 20, color: '#0064D2' },
    { name: 'Asiana', code: 'OZ', count: 11, color: '#C8102E' },
    { name: 'Jin Air', code: 'LJ', count: 8, color: '#FF6D00' },
    { name: 'Singapore Air', code: 'SQ', count: 5, color: '#F9A825' },
    { name: "T'way Air", code: 'TW', count: 4, color: '#E91E63' },
    { name: 'Jeju Air', code: '7C', count: 3, color: '#FF5722' },
    { name: 'Eastar Jet', code: 'ZE', count: 3, color: '#8BC34A' },
    { name: 'Air Seoul', code: 'RS', count: 3, color: '#00BCD4' },
    { name: 'Air Busan', code: 'BX', count: 3, color: '#9C27B0' },
    { name: 'Peach', code: 'MM', count: 2, color: '#E040FB' },
    { name: 'Vietnam Air', code: 'VN', count: 2, color: '#1B5E20' },
    { name: 'China Airlines', code: 'CI', count: 2, color: '#1A237E' },
    { name: 'EVA Airways', code: 'BR', count: 2, color: '#2E7D32' },
    { name: 'China Southern', code: 'CZ', count: 2, color: '#01579B' },
    { name: 'AirAsia PH', code: 'Z2', count: 2, color: '#D32F2F' },
    { name: 'Parata Air', code: 'WE', count: 2, color: '#546E7A' },
    { name: 'Air Premia', code: 'YP', count: 2, color: '#4527A0' },
    { name: 'Thai Airways', code: 'TG', count: 1, color: '#6A1B9A' }
  ];

  var maxCount = airlines[0].count;
  var container = document.getElementById('airline-chart');
  airlines.forEach(function(a) {
    var row = document.createElement('div');
    row.className = 'airline-bar';

    var nameDiv = document.createElement('div');
    nameDiv.className = 'name';
    nameDiv.textContent = a.code + ' ' + a.name;

    var barDiv = document.createElement('div');
    barDiv.className = 'bar';
    barDiv.style.width = (a.count / maxCount * 100) + '%';
    barDiv.style.background = a.color;

    var countDiv = document.createElement('div');
    countDiv.className = 'count';
    countDiv.textContent = a.count;

    row.appendChild(nameDiv);
    row.appendChild(barDiv);
    row.appendChild(countDiv);
    container.appendChild(row);
  });
})();
</script>

## Destinations

<table class="dest-table">
<thead><tr><th>Country</th><th>City</th><th>Dep.</th><th>Outbound (from here)</th></tr></thead>
<tbody>
<tr><td rowspan="4">Korea</td><td>Incheon (ICN) <em>home</em></td><td>30</td><td>VN415, MM006, OZ114, LJ237, OZ625, CI161, KE691, ZE931, 7C1602, LJ025, Z29047, YP601, KE657, TG659, ZE641, KE789, LJ227, TW251, LJ231, KE769, OZ110, WE501, KE663, KE633, OZ751, CZ676, OZ122, SQ601(p)</td></tr>
<tr><td>Gimpo (GMP) <em>home</em></td><td>8</td><td>BX8809, TW705, KE1211, KE1045, KE2101, TW711, KE1361, BX8807</td></tr>
<tr><td>Jeju (CJU)</td><td>5</td><td>KE1276, KE1272, KE1324, TW730, KE1344</td></tr>
<tr><td>Busan (PUS)</td><td>1</td><td>BX8826</td></tr>
<tr><td rowspan="6">Japan</td><td>Tokyo Narita (NRT)</td><td>3</td><td>KE706, 7C1122, WE502</td></tr>
<tr><td>Tokyo Haneda (HND)</td><td>1</td><td>-</td></tr>
<tr><td>Osaka (KIX)</td><td>3</td><td>MM009, OZ113</td></tr>
<tr><td>Fukuoka (FUK)</td><td>4</td><td>ZE642, LJ226, KE782, OZ135(t)</td></tr>
<tr><td>Sapporo (CTS)</td><td>3</td><td>TW252, LJ232, KE770</td></tr>
<tr><td>Nagoya (NGO)</td><td>3</td><td>7C1601, LJ348, OZ123</td></tr>
<tr><td rowspan="2">Thailand</td><td>Bangkok (BKK)</td><td>3</td><td>YP602, KE652, BR068(t)</td></tr>
<tr><td>Phuket (HKT)</td><td>1</td><td>KE664</td></tr>
<tr><td>Taiwan</td><td>Taipei (TPE)</td><td>3</td><td>CI162, KE694, BR106(t)</td></tr>
<tr><td>Philippines</td><td>Cebu (CEB)</td><td>2</td><td>LJ026, Z27046</td></tr>
<tr><td>Singapore</td><td>Singapore (SIN)</td><td>3</td><td>SQ826(p), SQ231(p), SQ608(p)</td></tr>
<tr><td>Indonesia</td><td>Bali (DPS)</td><td>1</td><td>KE634</td></tr>
<tr><td>Hong Kong</td><td>Hong Kong (HKG)</td><td>1</td><td>RS502</td></tr>
<tr><td>Vietnam</td><td>Hanoi (HAN)</td><td>1</td><td>VN416</td></tr>
<tr><td>Saipan</td><td>Saipan (SPN)</td><td>1</td><td>OZ626</td></tr>
<tr><td rowspan="2">China</td><td>Dalian (DLC)</td><td>1</td><td>CZ675</td></tr>
<tr><td>Shanghai (PVG) <em>transit</em></td><td>1</td><td>OZ362(p)</td></tr>
<tr><td>Australia</td><td>Sydney (SYD)</td><td>1</td><td>SQ222(p)</td></tr>
</tbody>
</table>

<p style="font-size:0.75rem;color:var(--text-muted-color);margin-top:8px;">(t) transit &nbsp; (p) planned</p>

## Yearly Flights

<div id="yearly-chart" style="display:flex;align-items:flex-end;gap:8px;height:200px;padding:16px 0;border-bottom:1px solid var(--border-color,#e9ecef);"></div>

<script>
(function() {
  var data = [
    {year:'2015',count:2},{year:'2016',count:3},{year:'2017',count:10},{year:'2018',count:6},
    {year:'2019',count:14},{year:'2020',count:0},{year:'2021',count:0},{year:'2022',count:4},
    {year:'2023',count:5},{year:'2024',count:10},{year:'2025',count:12},{year:'2026',count:11}
  ];
  var max = 14;
  var container = document.getElementById('yearly-chart');
  data.forEach(function(d) {
    var col = document.createElement('div');
    col.style.cssText = 'flex:1;display:flex;flex-direction:column;align-items:center;gap:4px;';

    var countEl = document.createElement('div');
    countEl.textContent = d.count || '';
    countEl.style.cssText = 'font-size:0.75rem;color:var(--text-muted-color,#6c757d);';

    var bar = document.createElement('div');
    var h = d.count > 0 ? Math.max((d.count / max) * 160, 4) : 0;
    bar.style.cssText = 'width:100%;max-width:36px;border-radius:4px 4px 0 0;background:' +
      (d.count === 0 ? 'var(--border-color,#dee2e6)' : '#e74c3c') +
      ';height:' + h + 'px;transition:height 0.5s ease;';

    var label = document.createElement('div');
    label.textContent = d.year.slice(2);
    label.style.cssText = 'font-size:0.72rem;color:var(--text-muted-color,#6c757d);margin-top:4px;';

    col.appendChild(countEl);
    col.appendChild(bar);
    col.appendChild(label);
    container.appendChild(col);
  });
})();
</script>

<br>

> Data source: Flighty App and Korea Immigration Service. Last updated April 2026.
{: .prompt-info }
