<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>Echo Desk</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<link rel="preconnect" href="https://fonts.googleapis.com"><link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500;600&family=IBM+Plex+Sans+Condensed:wght@400;500;600;700&display=swap">
<style>
/* Layout: full-bleed map; floating instrument panels top-left (site/product), top-right (layers), bottom dock (timeline). */
:root{
  color-scheme:dark;
  --ink:#07090c; --panel:rgba(13,16,21,.88); --panel-solid:#0e1116; --line:rgba(255,236,210,.09); --line-2:rgba(255,236,210,.16);
  --fg:#ece6da; --mute:#8d9099; --dim:#5d616b;
  --amber:#ffb347; --amber-d:#c97f1d; --amber-glow:rgba(255,179,71,.18);
  --ok:#5fd28a; --warn:#ffcf4d; --bad:#ff5a5f;
  --ui:"IBM Plex Sans Condensed",system-ui,-apple-system,"Segoe UI",sans-serif;
  --mono:"IBM Plex Mono",ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
  --r:4px;
}
html,body{height:100%;margin:0;background:var(--ink);color:var(--fg);font:14px/1.35 var(--ui);overflow:hidden;-webkit-font-smoothing:antialiased}
#map{position:absolute;inset:0;background:#0b0d10}
#map2{position:absolute;top:0;bottom:0;left:50%;right:0;background:#0b0d10;border-left:2px solid rgba(255,179,71,.35)}
body.split #map{right:50%}
body.split #readout{left:25%}
#xsPanel{left:calc(10px + env(safe-area-inset-left,0px));bottom:calc(178px + env(safe-area-inset-bottom,0px));width:min(580px,calc(100vw - 20px));z-index:1050}
#xsPanel header{display:flex;align-items:center;gap:10px;padding:7px 10px;border-bottom:1px solid var(--line)}
#xsPanel header b{flex:1;font:600 11px var(--mono);letter-spacing:.06em;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
#xsPanel header button{color:var(--mute);font:600 10px var(--mono);letter-spacing:.06em;border:1px solid var(--line-2);border-radius:2px;padding:2px 6px}
#xsCanvas{display:block;width:100%;height:auto}
#xsNote{font:500 10.5px var(--mono);color:var(--mute);padding:5px 10px 7px}
#paneB{top:calc(10px + env(safe-area-inset-top,0px));left:calc(50% + 12px);max-width:calc(50% - 90px);padding:6px 8px;display:flex;flex-direction:column;gap:5px;z-index:1000}
#paneB .pb-row{display:flex;gap:6px;align-items:center;min-width:0}
#paneB .pb-tag{font:700 11px var(--mono);color:#1a1208;background:var(--amber);border-radius:2px;padding:2px 6px}
#pbStrip{display:flex;gap:3px;overflow-x:auto;scrollbar-width:none;min-width:0;flex:1}
#pbStrip::-webkit-scrollbar{display:none}
#pbStrip button{flex:none;padding:3px 6px;border:1px solid var(--line-2);border-radius:3px;font:600 10px var(--mono);color:var(--mute)}
#pbStrip button[aria-pressed="true"]{background:var(--amber);color:#1a1208;border-color:var(--amber)}
#pbStrip button:disabled{opacity:.28}
#paneB .pb-row2{display:flex;gap:10px;align-items:center;font:500 11px var(--mono);color:var(--mute);font-variant-numeric:tabular-nums}
#pbVal{color:var(--amber);font-weight:600}
#pbLegend{width:150px;height:8px;border-radius:2px}
.xhair{pointer-events:none}
.xhair i{position:absolute;left:-9px;top:-1px;width:18px;height:2px;background:#fff;box-shadow:0 0 0 1px #000}
.xhair b{position:absolute;left:-1px;top:-9px;width:2px;height:18px;background:#fff;box-shadow:0 0 0 1px #000}
@media (max-aspect-ratio:1/1){
  #map2{top:50%;left:0;border-left:0;border-top:2px solid rgba(255,179,71,.35)}
  body.split #map{right:0;bottom:50%}
  body.split #readout{left:50%}
  #paneB{top:calc(50% + 10px);left:10px;max-width:calc(100% - 80px)}
}
.leaflet-container{background:#0b0d10;font-family:var(--ui)}
.leaflet-control-attribution{background:rgba(7,9,12,.7)!important;color:var(--dim)!important;font:10px var(--ui)}
.leaflet-control-attribution a{color:var(--mute)!important}
.leaflet-control-zoom{border:1px solid var(--line-2)!important;border-radius:var(--r)!important;overflow:hidden}
.leaflet-control-zoom a{background:var(--panel)!important;color:var(--fg)!important;border-color:var(--line)!important;backdrop-filter:blur(10px)}
.leaflet-tooltip.site-tip{background:var(--panel-solid);color:var(--fg);border:1px solid var(--line-2);font:600 11px var(--mono);box-shadow:none;padding:2px 6px;border-radius:3px}
.leaflet-tooltip.site-tip:before{display:none}
.leaflet-popup-content-wrapper{background:var(--panel-solid);color:var(--fg);border:1px solid var(--line-2);border-radius:var(--r);box-shadow:0 8px 30px rgba(0,0,0,.5)}
.leaflet-popup-tip{background:var(--panel-solid)}
.leaflet-popup-content{margin:12px 14px;font:13px/1.4 var(--ui);min-width:220px}
.leaflet-container a.leaflet-popup-close-button{color:var(--mute)}
.radar-canvas,.ovl-canvas{position:absolute;left:0;top:0;pointer-events:none}

.panel{position:absolute;z-index:1000;background:var(--panel);border:1px solid var(--line);border-radius:var(--r);backdrop-filter:blur(14px) saturate(1.2);-webkit-backdrop-filter:blur(14px) saturate(1.2);box-shadow:0 10px 40px rgba(0,0,0,.35)}
button,select,input{font:inherit;color:inherit}
button{cursor:pointer;background:none;border:0;padding:0}
:focus-visible{outline:2px solid var(--amber);outline-offset:1px}

/* top-left */
#hud{top:calc(10px + env(safe-area-inset-top,0px));left:calc(10px + env(safe-area-inset-left,0px));display:flex;flex-direction:column;gap:0;width:min(340px,calc(100vw - 20px))}
.site-row{display:flex;align-items:stretch;border-bottom:1px solid var(--line)}
#siteBtn{flex:1;display:flex;align-items:center;gap:10px;padding:10px 12px;text-align:left;min-width:0}
#siteBtn:hover{background:rgba(255,255,255,.03)}
.site-id{font:600 22px/1 var(--mono);letter-spacing:.02em;color:var(--amber)}
.site-meta{min-width:0;display:flex;flex-direction:column;gap:2px}
.site-name{font-weight:600;font-size:13px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.site-type{font:500 10px var(--mono);letter-spacing:.08em;text-transform:uppercase;color:var(--mute)}
.live{display:flex;flex-direction:column;justify-content:center;align-items:flex-end;padding:8px 12px;border-left:1px solid var(--line);gap:3px}
#clock{font:500 13px var(--mono);font-variant-numeric:tabular-nums}
#ageChip{font:600 10px var(--mono);letter-spacing:.06em;padding:1px 6px;border-radius:2px;background:rgba(95,210,138,.14);color:var(--ok)}
#ageChip.stale{background:rgba(255,207,77,.14);color:var(--warn)}
#ageChip.old{background:rgba(255,90,95,.14);color:var(--bad)}
.prod-row{display:flex;gap:8px;padding:8px 10px;align-items:center;flex-wrap:wrap}
.seg{display:flex;border:1px solid var(--line-2);border-radius:3px;overflow:hidden}
.seg button{padding:5px 10px;font:600 11px var(--mono);letter-spacing:.06em;color:var(--mute);border-right:1px solid var(--line)}
.seg button:last-child{border-right:0}
.seg button[aria-pressed="true"]{background:var(--amber);color:#1a1208}
.seg button:disabled{opacity:.35;cursor:not-allowed}
#prodStrip{display:flex;gap:4px;padding:8px 10px 0;overflow-x:auto;scrollbar-width:none;-webkit-mask-image:linear-gradient(90deg,#000 92%,transparent)}
#prodStrip::-webkit-scrollbar{display:none}
#prodStrip button{flex:none;padding:5px 8px;border:1px solid var(--line-2);border-radius:3px;font:600 10.5px var(--mono);letter-spacing:.05em;color:var(--mute);white-space:nowrap}
#prodStrip button:hover:not(:disabled){color:var(--fg);border-color:var(--mute)}
#prodStrip button[aria-pressed="true"]{background:var(--amber);border-color:var(--amber);color:#1a1208}
#prodStrip button:disabled{opacity:.28;cursor:not-allowed}
#prodStrip button:last-child{margin-right:24px}
select.tilt:disabled{opacity:.5}
select.tilt{background:var(--panel-solid);border:1px solid var(--line-2);border-radius:3px;padding:4px 6px;font:600 11px var(--mono);color:var(--fg)}
.prod-desc{width:100%;font:500 11px var(--mono);color:var(--mute);display:flex;justify-content:space-between;gap:8px}
.prod-desc b{color:var(--fg);font-weight:500}

/* site picker */
#picker{top:calc(10px + env(safe-area-inset-top,0px));left:calc(10px + env(safe-area-inset-left,0px));width:min(340px,calc(100vw - 20px));max-height:min(520px,70vh);display:flex;flex-direction:column;z-index:1100}
#picker header{display:flex;gap:8px;padding:10px;border-bottom:1px solid var(--line)}
#siteSearch{flex:1;background:rgba(0,0,0,.35);border:1px solid var(--line-2);border-radius:3px;padding:7px 9px;font:500 13px var(--mono)}
#picker header button{padding:0 10px;color:var(--mute);font-size:12px}
#siteList{overflow:auto;padding:4px 0}
#siteList button{display:grid;grid-template-columns:54px 1fr auto;gap:8px;align-items:center;width:100%;padding:7px 12px;text-align:left}
#siteList button:hover,#siteList button.sel{background:rgba(255,179,71,.08)}
#siteList .i{font:600 13px var(--mono);color:var(--amber)}
#siteList .n{white-space:nowrap;overflow:hidden;text-overflow:ellipsis;font-size:13px}
#siteList .t{font:600 9px var(--mono);letter-spacing:.08em;color:var(--mute);border:1px solid var(--line-2);padding:1px 4px;border-radius:2px}
#siteList .t.tdwr{color:#7fe3d6;border-color:rgba(127,227,214,.35)}
#siteList .t.mrrl{color:#c3a6ff;border-color:rgba(195,166,255,.4)}
#legend.img #legendBar,#legend.img #legendTicks{display:none}
#legend.img #legendNote{display:block}
#legendNote{display:none;font:500 10px/1.3 var(--mono);color:var(--mute)}

/* top-right layers */
#layersBtn{top:calc(10px + env(safe-area-inset-top,0px));right:calc(10px + env(safe-area-inset-right,0px));padding:9px 12px;font:600 11px var(--mono);letter-spacing:.08em;display:flex;gap:8px;align-items:center}
#layersBtn svg{width:16px;height:16px}
#layers{top:calc(52px + env(safe-area-inset-top,0px));right:calc(58px + env(safe-area-inset-right,0px));width:250px;padding:6px 0;max-height:calc(100vh - 230px);overflow:auto}
#layers h3{margin:8px 14px 4px;font:600 10px var(--mono);letter-spacing:.12em;color:var(--dim);text-transform:uppercase}
.tog{display:flex;align-items:center;gap:10px;padding:7px 14px;cursor:pointer;user-select:none}
.tog:hover{background:rgba(255,255,255,.03)}
.tog input{appearance:none;width:28px;height:16px;border-radius:8px;background:#2a2d33;position:relative;flex:none;cursor:pointer;transition:background .15s}
.tog input:after{content:"";position:absolute;top:2px;left:2px;width:12px;height:12px;border-radius:50%;background:#8a8d94;transition:transform .15s,background .15s}
.tog input:checked{background:var(--amber-d)}
.tog input:checked:after{transform:translateX(12px);background:#fff3e0}
.tog span{flex:1}
.tog small{font:500 10px var(--mono);color:var(--dim)}
.basemaps{display:grid;grid-template-columns:repeat(2,1fr);gap:6px;padding:4px 14px 8px}
.basemaps button{padding:6px;border:1px solid var(--line-2);border-radius:3px;font:500 11px var(--mono);color:var(--mute)}
.basemaps button[aria-pressed="true"]{border-color:var(--amber);color:var(--amber);background:var(--amber-glow)}
.rowctl{display:flex;align-items:center;gap:10px;padding:6px 14px}
.rowctl label{flex:1;font-size:13px}
.rowctl output{font:500 11px var(--mono);color:var(--mute);width:36px;text-align:right}

/* tools rail */
#tools{top:calc(52px + env(safe-area-inset-top,0px));right:calc(10px + env(safe-area-inset-right,0px));display:flex;flex-direction:column;padding:3px;gap:2px}
#tools button{width:36px;height:36px;display:grid;place-items:center;border-radius:3px;color:var(--mute)}
#tools button:hover{color:var(--fg);background:rgba(255,255,255,.05)}
#tools button[aria-pressed="true"]{background:var(--amber);color:#1a1208}
#tools button svg{width:18px;height:18px;fill:none;stroke:currentColor;stroke-width:1.8;stroke-linecap:round;stroke-linejoin:round}
#tools hr{border:0;border-top:1px solid var(--line);margin:2px 4px}
#toolsPanel{top:calc(52px + env(safe-area-inset-top,0px));right:calc(58px + env(safe-area-inset-right,0px));width:300px;max-height:calc(100vh - 230px);overflow:auto;padding:6px 0 8px}
#toolsPanel h3{margin:10px 14px 4px;font:600 10px var(--mono);letter-spacing:.12em;color:var(--dim);text-transform:uppercase}
#toolsPanel .tg{display:grid;grid-template-columns:1fr 1fr;gap:4px;padding:0 10px}
#toolsPanel .tg button{display:flex;align-items:center;gap:8px;padding:7px 8px;border:1px solid var(--line);border-radius:3px;text-align:left;font-size:12.5px;color:var(--fg);min-width:0}
#toolsPanel .tg button:hover{border-color:var(--mute)}
#toolsPanel .tg button[aria-pressed="true"]{border-color:var(--amber);background:var(--amber-glow);color:var(--amber)}
#toolsPanel .tg button span{flex:1;min-width:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
#toolsPanel .tg kbd{font:600 9.5px var(--mono);color:var(--dim);border:1px solid var(--line-2);border-radius:2px;padding:0 4px}
#sidePanel{top:calc(176px + env(safe-area-inset-top,0px));left:calc(10px + env(safe-area-inset-left,0px));width:min(340px,calc(100vw - 20px));max-height:calc(100vh - 390px);display:flex;flex-direction:column}
#sidePanel header{display:flex;align-items:center;gap:8px;padding:9px 12px;border-bottom:1px solid var(--line)}
#sidePanel header b{flex:1;font:600 11px var(--mono);letter-spacing:.1em;text-transform:uppercase}
#sidePanel header button{color:var(--mute);font-size:12px}
#sideBody{overflow:auto;padding:4px 0 6px;font-size:12.5px}
#sideBody table{width:100%;border-collapse:collapse;font:500 11.5px var(--mono);font-variant-numeric:tabular-nums}
#sideBody th{text-align:left;color:var(--dim);font-weight:600;font-size:10px;letter-spacing:.06em;padding:4px 8px;position:sticky;top:0;background:var(--panel-solid)}
#sideBody td{padding:5px 8px;border-top:1px solid var(--line)}
#sideBody tr[data-ll]{cursor:pointer}
#sideBody tr[data-ll]:hover td{background:rgba(255,179,71,.06)}
#sideBody .row{display:flex;align-items:center;gap:8px;padding:7px 12px;border-top:1px solid var(--line);cursor:pointer}
#sideBody .row:hover{background:rgba(255,179,71,.06)}
#sideBody .row .sw{width:10px;height:10px;border-radius:2px;flex:none}
#sideBody .row .m{flex:1;min-width:0}
#sideBody .row .m div{white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
#sideBody .row small{color:var(--mute);font:500 10.5px var(--mono)}
#sideBody .empty{padding:14px;color:var(--mute)}
#sideBody .act{display:flex;gap:6px;padding:8px 12px}
#sideBody .act button,#sideBody .act input{border:1px solid var(--line-2);border-radius:3px;padding:6px 9px;font:500 12px var(--ui);background:rgba(0,0,0,.3)}
#sideBody .act input{flex:1;min-width:0}
#sideBody .del{color:var(--dim);padding:0 4px}
#sideBody .del:hover{color:var(--bad)}
#sideBody kbd{font:600 10px var(--mono);border:1px solid var(--line-2);border-radius:2px;padding:1px 5px;color:var(--fg)}
#toolHint input{background:rgba(0,0,0,.35);border:1px solid var(--line-2);border-radius:3px;padding:3px 7px;font:500 12px var(--mono);color:var(--fg);width:200px;max-width:40vw}
.tool-label.val{border-color:rgba(255,255,255,.55);font-weight:600;font-size:11.5px}
.tool-label.note{border-color:rgba(255,255,255,.35);font:500 12px "IBM Plex Sans Condensed",sans-serif}
#toolHint{left:50%;transform:translateX(-50%);top:calc(84px + env(safe-area-inset-top,0px));padding:6px 12px;font:500 11.5px var(--mono);display:flex;gap:10px;align-items:center;max-width:calc(100vw - 120px)}
#toolHint b{color:var(--amber);font-weight:600}
#toolHint button{color:var(--mute);font:600 10px var(--mono);letter-spacing:.06em;border:1px solid var(--line-2);border-radius:2px;padding:2px 6px}
.tool-label{background:rgba(8,10,14,.86);color:#fff;border:1px solid rgba(255,179,71,.55);border-radius:3px;font:600 10.5px "IBM Plex Mono",monospace;padding:2px 6px;white-space:nowrap;box-shadow:none}
.tool-label:before{display:none}
.tool-label.trk{border-color:rgba(92,200,255,.7)}
.tool-label.pin{border-color:rgba(255,255,255,.5);font-weight:500;line-height:1.35}
.leaflet-container.tool-on{cursor:crosshair}

/* bottom dock */
#dock{left:50%;transform:translateX(-50%);bottom:calc(10px + env(safe-area-inset-bottom,0px));width:min(900px,calc(100vw - 20px));display:flex;flex-direction:column}
.dock-top{display:flex;align-items:center;gap:12px;padding:10px 12px 6px}
.tbtn{width:34px;height:34px;display:grid;place-items:center;border-radius:3px;border:1px solid var(--line-2);flex:none}
.tbtn:hover{border-color:var(--amber);color:var(--amber)}
.tbtn svg{width:16px;height:16px;fill:currentColor}
#playBtn{width:42px;height:42px;background:var(--amber);color:#1a1208;border:0}
#playBtn:hover{background:#ffc46b;color:#1a1208}
.frame-time{display:flex;flex-direction:column;min-width:0;margin-left:4px}
#fTimeZ{font:600 20px/1 var(--mono);font-variant-numeric:tabular-nums;letter-spacing:-.01em}
#fTimeL{font:500 11px var(--mono);color:var(--mute);margin-top:4px;font-variant-numeric:tabular-nums}
.spacer{flex:1}
#glmStat{display:flex;align-items:center;gap:7px;padding:4px 10px 4px 8px;border:1px solid var(--line-2);border-radius:3px;margin-left:6px}
#glmStat[hidden]{display:none}
#glmStat svg{width:16px;height:16px;fill:#ffd23f;stroke:#000;stroke-width:1.2}
#glmStat .n{font:600 18px/1 var(--mono);font-variant-numeric:tabular-nums;color:#ffe9a0}
#glmStat .u{display:flex;flex-direction:column;font:500 9.5px/1.25 var(--mono);color:var(--mute);font-variant-numeric:tabular-nums}
#glmStat.loading .n{opacity:.6}
#legend{display:flex;flex-direction:column;gap:3px;width:min(300px,38vw)}
#legendBar{height:10px;border-radius:2px;width:100%;display:block}
#legendTicks{position:relative;height:12px;font:500 9px var(--mono);color:var(--mute)}
#legendTicks span{position:absolute;transform:translateX(-50%)}
#legendUnit{font:600 9px var(--mono);letter-spacing:.1em;color:var(--dim);display:flex;justify-content:space-between}
#scrub{position:relative;margin:4px 12px 0;height:30px;display:flex;gap:2px;align-items:flex-end;cursor:pointer;touch-action:none;user-select:none}
#scrub i{flex:1;min-width:0;height:14px;background:#262a31;border-radius:1px;position:relative;transition:height .1s,background .2s}
#scrub i.ok{background:#4b5260}
#scrub i.loading{background:repeating-linear-gradient(90deg,#3b3f47 0 3px,#2a2d33 3px 6px)}
#scrub i.err{background:rgba(255,90,95,.45)}
#scrub i.cur{background:var(--amber);height:24px;box-shadow:0 0 12px var(--amber-glow)}
#scrub i.hover:not(.cur){background:#8d9099}
.dock-bot{display:flex;align-items:center;gap:18px;padding:8px 12px 10px;flex-wrap:wrap;border-top:1px solid var(--line);margin-top:8px}
.slide{display:flex;align-items:center;gap:8px;flex:1;min-width:170px}
.slide label{font:600 10px var(--mono);letter-spacing:.1em;color:var(--dim);text-transform:uppercase;width:52px}
.slide output{font:600 12px var(--mono);width:44px;text-align:right;font-variant-numeric:tabular-nums;color:var(--amber)}
input[type=range]{flex:1;appearance:none;height:18px;background:transparent;min-width:60px}
input[type=range]::-webkit-slider-runnable-track{height:3px;background:#2a2d33;border-radius:2px}
input[type=range]::-moz-range-track{height:3px;background:#2a2d33;border-radius:2px}
input[type=range]::-webkit-slider-thumb{appearance:none;width:12px;height:12px;border-radius:50%;background:var(--amber);margin-top:-4.5px;border:2px solid #1a1208}
input[type=range]::-moz-range-thumb{width:10px;height:10px;border-radius:50%;background:var(--amber);border:2px solid #1a1208}
#status{font:500 11px var(--mono);color:var(--mute);white-space:nowrap;overflow:hidden;text-overflow:ellipsis;max-width:100%}
#status b{color:var(--fg);font-weight:500}

/* readout */
#readout{left:50%;transform:translateX(-50%);top:calc(10px + env(safe-area-inset-top,0px));padding:6px 12px;font:500 12px var(--mono);display:flex;gap:14px;align-items:baseline;font-variant-numeric:tabular-nums;pointer-events:none}
#readout .v{font-size:16px;font-weight:600;color:var(--amber)}
#readout .k{color:var(--mute)}
#alertChip{left:50%;transform:translateX(-50%);top:calc(48px + env(safe-area-inset-top,0px));padding:4px 10px;font:600 10px var(--mono);letter-spacing:.06em;display:flex;gap:10px}
#alertChip span{display:flex;align-items:center;gap:5px}
#alertChip i{width:8px;height:8px;border-radius:1px;display:inline-block}

.pop h4{margin:0 0 2px;font:700 15px var(--ui);letter-spacing:.01em}
.pop .when{font:500 11px var(--mono);color:var(--mute);margin-bottom:8px}
.pop dl{display:grid;grid-template-columns:auto 1fr;gap:3px 10px;margin:0;font-size:12px}
.pop dt{color:var(--mute);font:500 11px var(--mono)}
.pop dd{margin:0}
.pop .tag{display:inline-block;font:700 10px var(--mono);letter-spacing:.06em;padding:2px 6px;border-radius:2px;margin-bottom:6px}

#toast{left:50%;transform:translateX(-50%);bottom:calc(190px + env(safe-area-inset-bottom,0px));padding:8px 14px;font:500 12px var(--mono);max-width:calc(100vw - 40px)}

@media (max-width:640px){
  #toolsPanel{right:calc(58px + env(safe-area-inset-right,0px));width:calc(100vw - 78px)}
  #sidePanel{top:calc(170px + env(safe-area-inset-top,0px));max-height:calc(100vh - 470px)}
  #tools{top:calc(52px + env(safe-area-inset-top,0px))}
  #toolHint{top:auto;bottom:calc(272px + env(safe-area-inset-bottom,0px))}
  #hud{width:calc(100vw - 20px - 52px)}
  .site-id{font-size:18px}
  #layersBtn span{display:none}
  #readout{top:auto;bottom:calc(200px + env(safe-area-inset-bottom,0px))}
  #alertChip{top:auto;bottom:calc(236px + env(safe-area-inset-bottom,0px))}
  #legend{width:100%;order:5}
  .dock-top{flex-wrap:wrap}
  .slide{min-width:140px}
  .leaflet-control-zoom{display:none}
  #toast{bottom:calc(240px + env(safe-area-inset-bottom,0px))}
}
@media (prefers-reduced-motion:reduce){*{transition:none!important}}
</style>
</head>
<body>
<div id="map"></div>
<div id="map2" hidden></div>
<div id="paneB" class="panel" hidden><div class="pb-row"><span class="pb-tag">B</span><div id="pbStrip"></div><select id="pbTilt" class="tilt" aria-label="Pane B elevation"></select></div><div class="pb-row2"><span id="pbTime">--</span><canvas id="pbLegend" width="150" height="8"></canvas><span id="pbVal"></span></div></div>

<section id="hud" class="panel" aria-label="Radar site and product">
  <div class="site-row">
    <button id="siteBtn" aria-haspopup="dialog" title="Change radar site (S)">
      <span class="site-id" id="siteId">KTLX</span>
      <span class="site-meta"><span class="site-name" id="siteName">Oklahoma City</span><span class="site-type" id="siteType">WSR-88D</span></span>
    </button>
    <div class="live"><span id="clock">--:--:--Z</span><span id="ageChip">--</span></div>
  </div>
  <div id="prodStrip" role="group" aria-label="Product"></div>
  <div class="prod-row">
    <div class="seg" role="group" aria-label="Data level">
      <button id="srcL2" data-src="L2" aria-pressed="false">LVL II</button>
      <button id="srcL3" data-src="L3" aria-pressed="true">LVL III</button>
    </div>
    <select id="tilt" class="tilt" aria-label="Elevation"></select>
    <div class="prod-desc"><span id="prodName"><b>Base Reflectivity</b></span><span id="elevTxt">--</span></div>
  </div>
</section>

<section id="picker" class="panel" hidden role="dialog" aria-label="Choose radar">
  <header><input id="siteSearch" placeholder="Search ID, city, state" autocomplete="off" spellcheck="false"><button id="pickClose">Close</button></header>
  <div id="siteList"></div>
</section>

<button id="layersBtn" class="panel" aria-expanded="false" aria-controls="layers"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 3 2 8l10 5 10-5-10-5Z"/><path d="m2 13 10 5 10-5"/></svg><span>LAYERS</span></button>
<section id="tools" class="panel" aria-label="Tools">
  <button data-tool="measure" aria-pressed="false" title="Measure distance and bearing (M)"><svg viewBox="0 0 24 24"><path d="M3 17 17 3l4 4L7 21z"/><path d="m7 13 2 2M10 10l2 2M13 7l2 2"/></svg></button>
  <button data-tool="track" aria-pressed="false" title="Storm motion and ETA (T)"><svg viewBox="0 0 24 24"><circle cx="5" cy="18" r="2"/><path d="M7 16 19 5"/><path d="m14 5h5v5"/><path d="M10.5 9.5l2 2M13.5 6.5l2 2" opacity=".6"/></svg></button>
  <button data-tool="pin" aria-pressed="false" title="Drop a location pin (P)"><svg viewBox="0 0 24 24"><path d="M12 21s-6-5.5-6-11a6 6 0 0 1 12 0c0 5.5-6 11-6 11z"/><circle cx="12" cy="10" r="2.2"/></svg></button>
  <button id="ringsBtn" aria-pressed="false" title="Range rings (R)"><svg viewBox="0 0 24 24"><circle cx="12" cy="12" r="2"/><circle cx="12" cy="12" r="6"/><circle cx="12" cy="12" r="10"/></svg></button>
  <button id="allToolsBtn" aria-pressed="false" title="All tools"><svg viewBox="0 0 24 24"><rect x="4" y="4" width="6" height="6" rx="1"/><rect x="14" y="4" width="6" height="6" rx="1"/><rect x="4" y="14" width="6" height="6" rx="1"/><rect x="14" y="14" width="6" height="6" rx="1"/></svg></button>
  <hr>
  <button id="clearTools" title="Clear measurements, tracks and pins"><svg viewBox="0 0 24 24"><path d="M4 7h16M9 7V4h6v3M6 7l1 13h10l1-13"/></svg></button>
</section>
<section id="toolsPanel" class="panel" hidden aria-label="All tools"></section>
<section id="sidePanel" class="panel" hidden aria-label="Panel"><header><b id="sideTitle"></b><button id="sideClose">Close</button></header><div id="sideBody"></div></section>
<div id="toolHint" class="panel" hidden><span id="toolHintTxt"></span><button id="toolDone">DONE</button></div>
<section id="layers" class="panel" hidden aria-label="Layers">
  <h3>Overlays</h3>
  <label class="tog"><input type="checkbox" id="lyAlerts" checked><span>Warnings</span><small>NWS</small></label>
  <label class="tog"><input type="checkbox" id="lyTracks" checked><span>Storm tracks</span><small id="trkSrc">NST</small></label>
  <label class="tog"><input type="checkbox" id="lyTVS" checked><span>TVS</span><small id="tvsSrc">NTV</small></label>
  <label class="tog"><input type="checkbox" id="lyHail"><span>Hail</span><small id="hailSrc">NHI</small></label>
  <label class="tog"><input type="checkbox" id="lyMeso"><span>Mesocyclones</span><small id="mesoSrc">NMD</small></label>
  <label class="tog"><input type="checkbox" id="lyLSR"><span>Storm reports</span><small>LSR · IEM</small></label>
  <label class="tog"><input type="checkbox" id="lySN"><span>Spotter reports</span><small id="snSrc">SpotterNetwork</small></label>
  <label class="tog"><input type="checkbox" id="lyMetar"><span>METARs</span><small>IEM</small></label>
  <label class="tog"><input type="checkbox" id="lyWind"><span>Wind particles</span><small>radar velocity</small></label>
  <label class="tog"><input type="checkbox" id="lyGLM"><span>Lightning</span><small>GLM</small></label>
  <label class="tog"><input type="checkbox" id="lySites" checked><span>Radar sites</span><small>88D · TDWR</small></label>
  <label class="tog"><input type="checkbox" id="lyMrrl" checked><span>International sites</span><small>MRRL</small></label>
  <label class="tog"><input type="checkbox" id="lyLabels" checked><span>Map labels</span><small></small></label>
  <h3>Display</h3>
  <div class="rowctl"><label for="palRef">Reflectivity colors</label><select id="palRef" class="tilt"></select></div>
  <div class="rowctl"><label for="palVel">Velocity colors</label><select id="palVel" class="tilt"></select></div>
  <div class="rowctl"><label for="minRef">Hide below</label><input type="range" id="minRef" min="-10" max="40" step="5" value="-10"><output id="minRefOut">off</output></div>
  <label class="tog"><input type="checkbox" id="lyHideRF"><span>Hide range folding</span><small>RF</small></label>
  <label class="tog"><input type="checkbox" id="lyUTC"><span>Show times in UTC</span><small></small></label>
  <h3>Playback</h3>
  <div class="rowctl"><label for="playMode">Mode</label><select id="playMode" class="tilt"><option value="loop">Loop</option><option value="rock">Rock</option></select></div>
  <div class="rowctl"><label for="dwell">Last-frame pause</label><input type="range" id="dwell" min="0" max="10" value="2"><output id="dwellOut">1 s</output></div>
  <h3>Underlays</h3>
  <label class="tog"><input type="checkbox" id="lySat"><span>GOES satellite</span><small>IEM</small></label>
  <div class="rowctl"><label for="satCh">Channel</label><select id="satCh" class="tilt"><option value="ch13">IR (13)</option><option value="ch02">Visible (2)</option><option value="ch09">Water vapor (9)</option><option value="ch07">Shortwave IR (7)</option></select></div>
  <label class="tog"><input type="checkbox" id="lyOpera"><span>OPERA composite (Europe)</span><small id="operaSrc">EUMETNET</small></label>
  <label class="tog"><input type="checkbox" id="lyMosaic"><span>National radar mosaic</span><small>frame-synced</small></label>
  <h3>Alerts &amp; outlooks</h3>
  <label class="tog"><input type="checkbox" id="lyAlarm"><span>Warning alarm</span><small>sound + notify</small></label>
  <label class="tog"><input type="checkbox" id="lySPC"><span>SPC Day 1 outlook</span><small>SPC</small></label>
  <h3>Rendering</h3>
  <label class="tog"><input type="checkbox" id="lySmooth"><span>Smoothing</span><small>bilinear</small></label>
  <div class="rowctl"><label for="opacity">Opacity</label><input type="range" id="opacity" min="20" max="100" value="90"><output id="opacityOut">90%</output></div>
  <label class="tog"><input type="checkbox" id="lyGlmFade" checked><span>Lightning fade</span><small>by age</small></label>
  <label class="tog"><input type="checkbox" id="lyGlmRange"><span>Radar range only</span><small>GLM</small></label>
  <div class="rowctl"><label for="rptWin">Report window</label><input type="range" id="rptWin" min="0" max="7" value="3"><output id="rptWinOut">2 h</output></div>
  <div class="rowctl"><label for="glmWin">GLM window</label><input type="range" id="glmWin" min="1" max="15" value="5"><output id="glmWinOut">5 m</output></div>
  <div class="rowctl"><label for="units">Distance units</label><select id="units" class="tilt"><option value="mi">mi</option><option value="km">km</option><option value="nm">nm</option></select></div>
  <h3>Basemap</h3>
  <div class="basemaps">
    <button data-bm="dark" aria-pressed="true">Dark</button>
    <button data-bm="light" aria-pressed="false">Light</button>
    <button data-bm="sat" aria-pressed="false">Satellite</button>
    <button data-bm="osm" aria-pressed="false">Streets</button>
  </div>
</section>

<div id="readout" class="panel" hidden><span class="v" id="roV">--</span><span class="k" id="roK"></span></div>
<div id="alertChip" class="panel" hidden></div>
<div id="toast" class="panel" hidden></div>

<section id="dock" class="panel" aria-label="Animation">
  <div class="dock-top">
    <button class="tbtn" id="prevBtn" title="Previous frame (←)"><svg viewBox="0 0 24 24"><path d="M6 5h2v14H6zM20 5v14L9 12z"/></svg></button>
    <button class="tbtn" id="playBtn" title="Play / pause (Space)"><svg viewBox="0 0 24 24" id="playIco"><path d="M7 4v16l13-8z"/></svg></button>
    <button class="tbtn" id="nextBtn" title="Next frame (→)"><svg viewBox="0 0 24 24"><path d="M16 5h2v14h-2zM4 5v14l11-7z"/></svg></button>
    <div class="frame-time"><span id="fTimeZ">--:--Z</span><span id="fTimeL">loading</span></div>
    <div id="glmStat" hidden title="GLM flashes in the lightning window"><svg viewBox="0 0 24 24"><path d="M14 2 4 14h7l-2 8 10-12h-7l2-8z"/></svg><span class="n" id="glmN">0</span><span class="u"><span id="glmRate">0/min</span><span id="glmScope">5 min</span></span></div>
    <span class="spacer"></span>
    <div id="legend"><div id="legendUnit"><span id="legendName">REFLECTIVITY</span><span id="legendU">dBZ</span></div><canvas id="legendBar" width="300" height="10"></canvas><div id="legendNote"></div><div id="legendTicks"></div></div>
  </div>
  <div id="scrub" aria-label="Frames"></div>
  <div class="dock-bot">
    <div class="slide"><label for="frames">Frames</label><input type="range" id="frames" min="1" max="200" value="12"><output id="framesOut">12</output></div>
    <div class="slide"><label for="speed">Speed</label><input type="range" id="speed" min="0" max="15" value="3"><output id="speedOut">1×</output></div>
    <div id="status">Starting…</div>
  </div>
</section>

<script id="worker-src" type="text/plain">// Compact bzip2 decoder: Uint8Array in -> Uint8Array out. Handles concatenated streams.
function bunzip2(src){
  let pos=0, bitBuf=0, bitCnt=0;
  const nbits=n=>{ while(bitCnt<n){ if(pos>=src.length) throw new Error('bz2: eof'); bitBuf=((bitBuf<<8)|src[pos++])>>>0; bitCnt+=8; }
    bitCnt-=n; const v=(bitBuf>>>bitCnt)&((1<<n)-1); bitBuf&=(1<<bitCnt)-1; return v; };
  const bit=()=>nbits(1);
  let out=new Uint8Array(Math.max(1024,src.length*6)), op=0;
  const put=b=>{ if(op>=out.length){ const n=new Uint8Array(out.length*2); n.set(out); out=n; } out[op++]=b; };
  let tt=null;
  while(true){
    // stream header
    if(pos+4>src.length) break;
    if(src[pos]!==0x42||src[pos+1]!==0x5a||src[pos+2]!==0x68){ break; }
    const level=src[pos+3]-48; pos+=4; bitBuf=0; bitCnt=0;
    if(level<1||level>9) throw new Error('bz2: level');
    const bsize=level*100000; if(!tt||tt.length<bsize) tt=new Uint32Array(bsize);
    while(true){
      const m1=nbits(24), m2=nbits(24);
      if(m1===0x177245&&m2===0x385090){ nbits(32); bitBuf=0; bitCnt=0; break; }
      if(m1!==0x314159||m2!==0x265359) throw new Error('bz2: magic');
      nbits(32); if(bit()) throw new Error('bz2: randomized');
      const origPtr=nbits(24);
      const used=nbits(16), seqToUnseq=[];
      for(let i=0;i<16;i++) if(used&(1<<(15-i))){ const u=nbits(16); for(let j=0;j<16;j++) if(u&(1<<(15-j))) seqToUnseq.push(i*16+j); }
      const nInUse=seqToUnseq.length, alpha=nInUse+2;
      const nGroups=nbits(3), nSel=nbits(15);
      const mtfG=[0,1,2,3,4,5], selectors=new Uint8Array(nSel);
      for(let i=0;i<nSel;i++){ let j=0; while(bit()) j++; const v=mtfG[j]; for(;j>0;j--) mtfG[j]=mtfG[j-1]; mtfG[0]=v; selectors[i]=v; }
      const groups=[];
      for(let g=0;g<nGroups;g++){
        const len=new Uint8Array(alpha); let c=nbits(5);
        for(let s=0;s<alpha;s++){ while(bit()){ c+= bit()? -1:1; } len[s]=c; }
        let minL=32,maxL=0; for(const l of len){ if(l<minL)minL=l; if(l>maxL)maxL=l; }
        const perm=new Uint16Array(alpha), base=new Int32Array(23), limit=new Int32Array(23).fill(-1);
        let pp=0; for(let l=minL;l<=maxL;l++) for(let s=0;s<alpha;s++) if(len[s]===l) perm[pp++]=s;
        for(let s=0;s<alpha;s++) base[len[s]+1]++;
        for(let i=1;i<23;i++) base[i]+=base[i-1];
        let vec=0; for(let l=minL;l<=maxL;l++){ vec+=base[l+1]-base[l]; limit[l]=vec-1; vec<<=1; }
        for(let l=minL+1;l<=maxL;l++) base[l]=((limit[l-1]+1)<<1)-base[l];
        groups.push({minL,maxL,perm,base,limit});
      }
      // decode MTF/RLE2
      const mtf=new Uint8Array(256); for(let i=0;i<256;i++) mtf[i]=i;
      const ll=new Uint8Array(bsize), cnt=new Int32Array(256);
      let n=0, gi=-1, left=0, g=null, runAcc=-1, runN=1;
      const EOB=alpha-1;
      while(true){
        if(left===0){ gi++; if(gi>=nSel) throw new Error('bz2: sel'); g=groups[selectors[gi]]; left=50; }
        left--;
        let zn=g.minL, zv=nbits(zn);
        while(zn<=g.maxL && zv>g.limit[zn]){ zn++; zv=(zv<<1)|bit(); }
        if(zn>g.maxL) throw new Error('bz2: huff');
        const sym=g.perm[zv-g.base[zn]];
        if(sym<=1){ if(runAcc<0){runAcc=0;runN=1;} runAcc+=(sym+1)*runN; runN<<=1; continue; }
        if(runAcc>=0){ const b=seqToUnseq[mtf[0]]; cnt[b]+=runAcc; if(n+runAcc>bsize) throw new Error('bz2: overflow'); ll.fill(b,n,n+runAcc); n+=runAcc; runAcc=-1; }
        if(sym===EOB) break;
        const idx=sym-1, v=mtf[idx]; mtf.copyWithin(1,0,idx); mtf[0]=v;
        const b=seqToUnseq[v]; cnt[b]++; ll[n++]=b;
      }
      // inverse BWT
      const cf=new Int32Array(257); for(let i=0;i<256;i++) cf[i+1]=cf[i]+cnt[i];
      for(let i=0;i<n;i++) tt[cf[ll[i]]++]=i;
      let tp=tt[origPtr], last=-1, run=0;
      for(let k=0;k<n;k++){
        const b=ll[tp]; tp=tt[tp];
        if(run===4){ for(let r=0;r<b;r++) put(last); run=0; last=-1; continue; }
        if(b===last) run++; else { last=b; run=1; }
        put(b);
      }
    }
  }
  return out.subarray(0,op);
}
if(typeof module!=='undefined') module.exports=bunzip2;

/* ===== Echo Desk decode worker ===== */
// Grids hold the product's raw data codes; `dec` tells the main thread how to turn a code into a value.
const NAZ=720;
const G=typeof self!=='undefined'?self:globalThis;

function fillGrid(radials,nGates){
  const grid=new Uint8Array(NAZ*nGates);
  for(const r of radials){
    let s=Math.round(r.start/0.5), e=Math.round((r.start+r.width)/0.5);
    if(e<=s) e=s+1;
    const d=r.data, n=Math.min(d.length,nGates), src=n===d.length?d:d.subarray(0,n);
    for(let k=s;k<e;k++){ grid.set(src,(((k%NAZ)+NAZ)%NAZ)*nGates); }
  }
  return grid;
}

/* ---- Level II (Archive II, V06) from a byte source ----
   src: async iterator of Uint8Array pieces. Archive files stream from one URL;
   real-time volumes come as a list of chunk objects (S = header + metadata, I/E = LDM records). */
async function* urlStream(url,signal){
  const res=await fetch(url,{signal}); if(!res.ok) throw new Error('HTTP '+res.status);
  const reader=res.body.getReader();
  try{ while(true){ const {value,done}=await reader.read(); if(done) return; yield value; } }
  finally{ try{ reader.cancel(); }catch(e){} }
}
async function* chunkStream(urls,signal){
  for(const u of urls){ const r=await fetch(u,{signal}); if(!r.ok) throw new Error('HTTP '+r.status); yield new Uint8Array(await r.arrayBuffer()); }
}
async function decodeL2(src,block,tilt,requireComplete){
  const it=src[Symbol.asyncIterator]();
  let buf=new Uint8Array(1<<22), len=0, rp=24, done=false, bytes=0;
  const st={ lat:null, lon:null, elevs:[], seen:new Map(), target:-1, radials:[], finished:false, endSeen:false, gate:0, first:0, angle:0, scale:0, offset:0 };
  async function need(n){
    while(len-rp<n){
      if(done) return false;
      const {value,done:d}=await it.next();
      if(d){ done=true; return len-rp>=n; }
      bytes+=value.length;
      if(len+value.length>buf.length){
        const keep=len-rp; const nb=new Uint8Array(Math.max(buf.length,(keep+value.length)*2)); nb.set(buf.subarray(rp,len)); buf=nb; len=keep; rp=0;
      }
      buf.set(value,len); len+=value.length;
    }
    return true;
  }
  try{
    while(!st.finished){
      if(!(await need(4))) break;
      const size=Math.abs(new DataView(buf.buffer,buf.byteOffset+rp,4).getInt32(0)); rp+=4;
      if(size===0) continue;
      if(!(await need(size))) break;
      const rec=bunzip2(buf.subarray(rp,rp+size)); rp+=size;
      parseL2Messages(rec,st,block,tilt);
    }
  } finally { try{ await it.return(); }catch(e){} }
  if(!st.radials.length) throw new Error(requireComplete?'incomplete':'No '+block.trim()+' at tilt '+(tilt+1));
  if(requireComplete && !st.finished && !st.endSeen) throw new Error('incomplete');
  let nG=0; for(const r of st.radials) if(r.data.length>nG) nG=r.data.length;
  const mul=(block==='DVEL'||block==='DSW ')?1.943844:1;
  return { lat:st.lat, lon:st.lon, nGates:nG, gateKm:st.gate/1000, firstKm:st.first/1000, grid:fillGrid(st.radials,nG), elev:st.angle,
    dec:{t:'so',scale:st.scale,offset:st.offset,lead:2,mul,rf:block==='DVEL'||block==='DSW '}, bytes };
}
function parseL2Messages(rec,st,blk,tilt){
  const dv=new DataView(rec.buffer,rec.byteOffset,rec.byteLength);
  let o=0; const N=rec.length;
  while(o+28<=N){
    const size=dv.getUint16(o+12), type=dv.getUint8(o+15);
    if(type===31){
      const d=o+28;
      const azAng=dv.getFloat32(d+12), azRes=dv.getUint8(d+20), elNum=dv.getUint8(d+22), elAng=dv.getFloat32(d+24), nb=dv.getUint16(d+30);
      let mom=null, vol=null;
      for(let b=0;b<nb && b<10;b++){
        const p=dv.getUint32(d+32+b*4); if(!p) continue; const q=d+p; if(q+4>N) continue;
        const name=String.fromCharCode(rec[q],rec[q+1],rec[q+2],rec[q+3]);
        if(name==='RVOL') vol=q; else if(name===blk) mom=q;
      }
      if(vol!==null && st.lat===null){ st.lat=dv.getFloat32(vol+8); st.lon=dv.getFloat32(vol+12); }
      let info=st.seen.get(elNum);
      if(!info){
        info={has:!!mom, ang:Math.round(elAng*10)/10};
        st.seen.set(elNum,info);
        if(st.target>=0 && elNum!==st.target){ st.finished=true; return; }
        if(info.has && st.target<0 && !st.elevs.some(a=>Math.abs(a-info.ang)<0.2)){
          st.elevs.push(info.ang);
          if(st.elevs.length-1===tilt){ st.target=elNum; st.angle=info.ang; }
        }
      }
      const rstat=dv.getUint8(d+21);
      if(elNum===st.target && (rstat===2||rstat===4)) st.endSeen=true;
      if(elNum===st.target && mom!==null){
        const ng=dv.getUint16(mom+8), first=dv.getInt16(mom+10), gs=dv.getInt16(mom+12), ws=dv.getUint8(mom+19), sc=dv.getFloat32(mom+20), of=dv.getFloat32(mom+24);
        if(ws===8 && sc>0){
          if(!st.scale){ st.scale=sc; st.offset=of; st.first=first; st.gate=gs; }
          const w=azRes===1?0.5:1.0;
          st.radials.push({start:azAng-w/2,width:w,data:rec.slice(mom+28,mom+28+ng)});
        }
      }
      o+=12+size*2;
    } else o+=2432;
  }
}

/* ---- Level III ---- */
const L3_GATE={153:0.25,154:0.25,159:0.25,161:0.25,163:0.25,165:0.25,94:1,99:0.25,180:0.15,182:0.15,181:0.15,183:0.15,134:1,135:1};
const RASTER_RANGE={35:230,36:460,37:230,38:460,41:230,57:230,63:230,64:230,65:230,66:230,90:230};
const L3_RANGE={186:417,187:417,78:230,79:230,80:230,56:230,169:230,170:230,171:230,172:230,173:230,174:230,175:230,177:230,138:230};
const L3_COMPRESSED=new Set([32,94,99,113,134,135,138,149,152,153,154,155,159,161,163,165,166,169,170,171,172,173,174,175,176,177,178,179,180,182,186,193,195,202]);
const VEL_CODES=new Set([154,99,182,183,27,56]);
function f16(h){ const s=(h>>15)&1, e=(h>>10)&31, f=h&1023; const v=e===0?2*(f/1024):Math.pow(2,e-16)*(1+f/1024); return s?-v:v; }
function findMsgStart(u8){
  const dv=new DataView(u8.buffer,u8.byteOffset,u8.byteLength);
  for(let p=0;p<Math.min(200,u8.length-32);p++){
    const c=dv.getInt16(p);
    if(c>15 && c<300 && dv.getInt16(p+18)===-1 && dv.getInt16(p+30)===c) return p;
  }
  throw new Error('No L3 header');
}
function l3Message(u8){
  const s=findMsgStart(u8);
  let m=u8.subarray(s);
  let dv=new DataView(m.buffer,m.byteOffset,m.byteLength);
  const code=dv.getInt16(0);
  const lat=dv.getInt32(20)/1000, lon=dv.getInt32(24)/1000;
  const hw=[]; for(let i=0;i<16;i++) hw.push(dv.getInt16(60+i*2));
  const hwu=hw.map(v=>v&0xffff);
  const f32a=dv.getFloat32(60), f32b=dv.getFloat32(64);
  const elevAng=dv.getInt16(58)/10;
  const vtime=dv.getInt16(40), vsec=dv.getInt32(42);
  if(dv.getInt16(100)===1 && m[120]===0x42 && m[121]===0x5a && m[122]===0x68){
    const body=bunzip2(m.subarray(120));
    const full=new Uint8Array(120+body.length); full.set(m.subarray(0,120)); full.set(body,120);
    m=full; dv=new DataView(m.buffer,m.byteOffset,m.byteLength);
  }
  const symOff=dv.getInt32(108)*2, graOff=dv.getInt32(112)*2, tabOff=dv.getInt32(116)*2;
  return {m,dv,code,lat,lon,hw,hwu,f32a,f32b,elevAng,symOff,graOff,tabOff,vol:(vtime-1)*864e5+vsec*1000};
}
function* l3Packets(L){
  const {dv,symOff,m}=L; if(!symOff) return;
  let o=symOff;
  if(dv.getInt16(o)!==-1) return;
  const blockLen=dv.getInt32(o+4), nLayers=dv.getInt16(o+8); const end=Math.min(m.length,symOff+blockLen);
  o+=10;
  for(let L_=0;L_<nLayers && o<end;L_++){
    if(dv.getInt16(o)!==-1) break;
    const lLen=dv.getInt32(o+2); o+=6; const lEnd=Math.min(end,o+lLen);
    while(o<lEnd-4){
      const pc=dv.getUint16(o);
      yield {code:pc,o};
      const len=packetLen(dv,o,pc,lEnd);
      if(len<=0) break;
      o+=len;
    }
    o=lEnd;
  }
}
function packetLen(dv,o,pc,end){
  if(pc===16){ const nr=dv.getInt16(o+12); let q=o+14; for(let r=0;r<nr && q<end;r++){ const nb=dv.getUint16(q); q+=6+nb+(nb&1); } return q-o; }
  if(pc===0xAF1F){ const nr=dv.getInt16(o+12); let q=o+14; for(let r=0;r<nr && q<end;r++){ const nh=dv.getUint16(q); q+=6+nh*2; } return q-o; }
  if(pc===0xBA0F||pc===0xBA07){ const nrows=dv.getInt16(o+18); let q=o+22; for(let r=0;r<nrows&&q<end;r++){ q+=2+dv.getInt16(q);} return q-o; }
  return 4+dv.getUint16(o+2);
}
function legacyTable(L){
  const vals=[];
  for(let i=0;i<16;i++){
    const h=L.hwu[i], flags=h>>8, v=h&0xff;
    if(flags&0x80){ vals.push(v===3?'RF':null); continue; }
    let x=v; if(flags&0x40) x*=0.01; else if(flags&0x20) x*=0.05; else if(flags&0x10) x*=0.1;
    if(flags&0x01) x=-x;
    vals.push(x);
  }
  return vals;
}
function l3Dec(L){
  const c=L.code, hw=L.hw;
  if([165,177].includes(c)) return {t:'cat'};
  if(c===134) return {t:'vil',ls:f16(L.hwu[0]),lo:f16(L.hwu[1]),start:hw[2],gs:f16(L.hwu[3]),go:f16(L.hwu[4])};
  if(c===135) return {t:'eet',mask:L.hwu[0],scale:hw[1],offset:hw[2]};
  if([159,161,163,170,172,173,174,175].includes(c)) return {t:'so',scale:L.f32a,offset:L.f32b,lead:Math.max(2,hw[6]||2),mul:c>=170?0.01:1,rf:false};
  const vel=VEL_CODES.has(c);
  return {t:'lin',min:hw[0]/10,inc:hw[1]/10,mul:vel?1.943844:1,rf:vel};
}
function decodeL3Radar(u8){
  const L=l3Message(u8);
  const {dv,m}=L; const radials=[]; let nG=0,firstBin=0, dec=null;
  for(const p of l3Packets(L)){
    if(p.code===16){
      let o=p.o; firstBin=dv.getInt16(o+2); nG=dv.getInt16(o+4); const nr=dv.getInt16(o+12); o+=14;
      for(let r=0;r<nr;r++){
        const nb=dv.getUint16(o), sa=dv.getInt16(o+2)/10, da=dv.getInt16(o+4)/10;
        radials.push({start:sa,width:da>0?da:0.5,data:m.slice(o+6,o+6+Math.min(nb,nG))});
        o+=6+nb+(nb&1);
      }
      dec=l3Dec(L); break;
    }
    if(p.code===0xAF1F){
      let o=p.o; firstBin=dv.getInt16(o+2); nG=dv.getInt16(o+4); const nr=dv.getInt16(o+12); o+=14;
      for(let r=0;r<nr;r++){
        const nh=dv.getUint16(o), sa=dv.getInt16(o+2)/10, da=dv.getInt16(o+4)/10; const q=o+6;
        const out=new Uint8Array(nG); let g=0;
        for(let k=0;k<nh*2;k++){ const b=m[q+k], run=b>>4, cc=b&15; for(let t=0;t<run&&g<nG;t++) out[g++]=cc; }
        radials.push({start:sa,width:da>0?da:1,data:out}); o=q+nh*2;
      }
      dec={t:'tab',vals:legacyTable(L)}; break;
    }
    if(p.code===0xBA0F||p.code===0xBA07) return rasterToPolar(L,p.o);
  }
  if(!radials.length) throw new Error('No radial data (product '+L.code+')');
  let gate=L3_GATE[L.code]; if(!gate) gate=Math.round((L3_RANGE[L.code]||460)/nG*1000)/1000;
  return { lat:L.lat, lon:L.lon, nGates:nG, gateKm:gate, firstKm:firstBin*gate, grid:fillGrid(radials,nG), elev:L.elevAng, dec, code:L.code };
}

/* ---- raster products (NCR, NET, NVL ...) resampled onto the polar grid ---- */
function rasterToPolar(L,o){
  const {dv,m}=L;
  const i0=dv.getInt16(o+6), j0=dv.getInt16(o+8), xs=dv.getInt16(o+10), ys=dv.getInt16(o+14), nrows=dv.getInt16(o+18);
  let q=o+22; const rows=[]; let ncols=0;
  for(let r=0;r<nrows;r++){ const nb=dv.getInt16(q); q+=2; const row=[];
    for(let k=0;k<nb;k++){ const b=m[q+k], run=b>>4, c=b&15; for(let t=0;t<run;t++) row.push(c); }
    q+=nb; rows.push(row); if(row.length>ncols) ncols=row.length; }
  // screen units are km; scale = km per cell. Fall back to the product's nominal range when scale is odd.
  let cell=(xs>=1&&xs<=16)?xs:2*(RASTER_RANGE[L.code]||230)/Math.max(nrows,1);
  let cellY=(ys>=1&&ys<=16)?ys:cell;
  let x0=i0, y0=j0;
  if(!(Math.abs(i0)>0&&Math.abs(i0)<=2048&&Math.abs(j0)<=2048)){ x0=-ncols*cell/2; y0=-nrows*cellY/2; }
  const maxKm=Math.max(Math.hypot(x0,y0),Math.hypot(x0+ncols*cell,y0),Math.hypot(x0,y0+nrows*cellY),Math.hypot(x0+ncols*cell,y0+nrows*cellY));
  const gate=Math.max(.5,Math.min(cell,cellY)/2), nG=Math.min(2000,Math.ceil(maxKm/gate));
  const grid=new Uint8Array(720*nG);
  for(let a=0;a<720;a++){ const az=(a+.5)*.5*Math.PI/180, sa=Math.sin(az), ca=Math.cos(az), row0=a*nG;
    for(let g=0;g<nG;g++){ const r=(g+.5)*gate, x=r*sa, yd=-r*ca;
      const c=Math.floor((x-x0)/cell), rr=Math.floor((yd-y0)/cellY);
      if(rr<0||rr>=nrows||c<0) continue; const row=rows[rr]; if(c>=row.length) continue; grid[row0+g]=row[c]; } }
  return { lat:L.lat, lon:L.lon, nGates:nG, gateKm:gate, firstKm:0, grid, elev:null, dec:{t:'tab',vals:legacyTable(L)}, code:L.code, raster:true };
}

/* ---- NST / NTV ---- */
function ijToLL(lat0,lon0,xkm,ykm){
  const R=6371, d=Math.hypot(xkm,ykm)/R, b=Math.atan2(xkm,ykm), p1=lat0*Math.PI/180, l1=lon0*Math.PI/180;
  const p2=Math.asin(Math.sin(p1)*Math.cos(d)+Math.cos(p1)*Math.sin(d)*Math.cos(b));
  const l2=l1+Math.atan2(Math.sin(b)*Math.sin(d)*Math.cos(p1),Math.cos(d)-Math.sin(p1)*Math.sin(p2));
  return [p2*180/Math.PI,l2*180/Math.PI];
}
// azimuth (deg) / range (nm) from the radar -> [lat, lon]
function azRanLL(lat0,lon0,az,nm){ const km=nm*1.852, r=az*Math.PI/180; return ijToLL(lat0,lon0,km*Math.sin(r),km*Math.cos(r)); }
const num=(s)=>{ if(s==null) return null; const t=s.replace(/[<>]/g,'').trim(); if(!t||!/^-?[\d.]+$/.test(t)) return null; const v=+t; return isFinite(v)?v:null; };
const str=(dv,m,o,n)=>{ let s=''; for(let i=0;i<n;i++) s+=String.fromCharCode(m[o+i]); return s; };
// Tabular alphanumeric block -> pages of lines
function tabularPages(L){
  const {dv,m,tabOff}=L; const pages=[]; if(!tabOff||tabOff+130>m.length) return pages;
  let o=tabOff; if(dv.getInt16(o)!==-1) return pages; o+=8+120;
  if(dv.getInt16(o)!==-1) return pages; o+=2;
  const np=dv.getInt16(o); o+=2;
  for(let p=0;p<np && o<m.length;p++){ const lines=[];
    while(o+2<=m.length){ const n=dv.getInt16(o); o+=2; if(n===-1) break; if(n<0||o+n>m.length) return pages; lines.push(str(dv,m,o,n)); o+=n; }
    pages.push(lines); }
  return pages;
}
// Graphic alphanumeric block -> text strings (packet 8 / 1)
function graphicLines(L){
  const {dv,m,graOff}=L; const out=[]; if(!graOff||graOff+10>m.length) return out;
  let o=graOff; if(dv.getInt16(o)!==-1) return out; const end=Math.min(m.length,graOff+dv.getInt32(o+4)); o+=8;
  const np=dv.getInt16(o); o+=2;
  for(let p=0;p<np && o<end;p++){ o+=2; const plen=dv.getInt16(o); o+=2; const pend=Math.min(end,o+plen);
    while(o+4<=pend){ const c=dv.getUint16(o), l=dv.getUint16(o+2);
      if(c===8) out.push(str(dv,m,o+10,l-6)); else if(c===1) out.push(str(dv,m,o+8,l-4));
      if(l<=0) break; o+=4+l; }
    o=pend; }
  return out;
}
function parseStiDate(s){ const m=s&&s.match(/(\d\d):(\d\d):(\d\d)\/(\d\d):(\d\d):(\d\d)/); return m?Date.UTC(2000+ +m[3],m[1]-1,+m[2],+m[4],+m[5],+m[6]):null; }

function decodeSTI(L){
  const recs={}; const get=id=>recs[id]||(recs[id]={id,cur:null,dir:null,kt:null,fcst:[],err:null,mean:null,dbzm:null,hgt:null});
  let time=null;
  for(const page of tabularPages(L)){
    if(!page.length||page[0].indexOf('STORM POSITION/FORECAST')<0) continue;
    page.forEach((line,i)=>{
      if(i===1){ time=time||parseStiDate(line.slice(29,46))||parseStiDate(line); return; }
      if(i<7||line.length<40) return;
      const id=line.slice(2,4); if(!/^[A-Z]\d$/.test(id)) return;
      const r=get(id);
      const az=num(line.slice(9,12)), rn=num(line.slice(13,16)); if(az!=null&&rn!=null&&!r.cur) r.cur=[az,rn];
      const dir=num(line.slice(19,22)), kt=num(line.slice(23,26)); if(dir!=null&&r.dir==null){ r.dir=dir; r.kt=kt; }
      if(!r.fcst.length) for(let j=0;j<4;j++){ const a=num(line.slice(31+j*10,34+j*10)), b=num(line.slice(35+j*10,38+j*10)); if(a==null||b==null) break; r.fcst.push([a,b]); }
      if(r.err==null){ r.err=num(line.slice(71,75)); r.mean=num(line.slice(76,80)); }
    });
  }
  // graphic block: STORM ID / AZ/RAN / FCST MVT / ERR/MEAN / DBZM HGT rows (10-char stride)
  let ids=[];
  for(const t of graphicLines(L)){
    if(t.startsWith(' STORM ID')){ ids=[]; for(let k=0,o=17;k<6;k++,o+=10){ const s=t.slice(o,o+2); if(/^[A-Z]\d$/.test(s)) ids.push(s); } }
    else if(t.startsWith(' AZ/RAN')) ids.forEach((id,k)=>{ const r=get(id); if(!r.cur){ const a=num(t.slice(11+k*10,14+k*10)), b=num(t.slice(15+k*10,18+k*10)); if(a!=null&&b!=null) r.cur=[a,b]; } });
    else if(t.startsWith(' FCST MVT')) ids.forEach((id,k)=>{ const r=get(id); if(r.dir==null){ r.dir=num(t.slice(11+k*10,14+k*10)); r.kt=num(t.slice(15+k*10,18+k*10)); } });
    else if(t.startsWith(' DBZM HGT')) ids.forEach((id,k)=>{ const r=get(id); r.dbzm=num(t.slice(12+k*10,14+k*10)); r.hgt=num(t.slice(15+k*10,19+k*10)); });
    else if(t.startsWith(' ERR/MEAN')) ids.forEach((id,k)=>{ const r=get(id); if(r.err==null){ r.err=num(t.slice(10+k*10,14+k*10)); r.mean=num(t.slice(15+k*10,19+k*10)); } });
  }
  return {recs,time};
}
function decodeTVSTable(L){
  const out=[];
  const re=/\b(E?TVS)\s+([A-Z]\d)\s+(\d{1,3})\/\s*(\d{1,3})\s+(\d+)\s+(\d+)\s+(\d+)\/\s*([\d.]+)\s+([<>]?\s*[\d.]+)\s+([<>]?\s*[\d.]+)\/\s*([<>]?\s*[\d.]+)\s+(\d+)\/\s*([\d.]+)/;
  for(const page of tabularPages(L)) for(const line of page){ const m=line.match(re); if(!m) continue;
    out.push({type:m[1],cell:m[2],az:+m[3],ran:+m[4],avgdv:+m[5],lldv:+m[6],mxdv:+m[7],mxdvhgt:+m[8],depth:m[9].replace(/\s/g,''),base:m[10].replace(/\s/g,''),top:m[11].replace(/\s/g,''),maxshear:+m[12],maxshearhgt:+m[13]}); }
  return out;
}
function decodeL3Cells(u8){
  const L=l3Message(u8); const {dv,m}=L;
  const pk=[]; let cur=null; const tvsPk=[]; const hailPk=[]; const featPk=[];
  for(const p of l3Packets(L)){
    const o=p.o;
    if(p.code===15){ cur={id:String.fromCharCode(m[o+8],m[o+9]),ij:[dv.getInt16(o+4),dv.getInt16(o+6)]}; pk.push(cur); }
    else if(p.code===12||p.code===26){ tvsPk.push({ij:[dv.getInt16(o+4),dv.getInt16(o+6)],elevated:p.code===26}); }
    else if(p.code===19){ const len=dv.getUint16(o+2); for(let k=o+4;k+9<o+4+len+1;k+=10) hailPk.push({ij:[dv.getInt16(k),dv.getInt16(k+2)],pHail:dv.getInt16(k+4),pSvr:dv.getInt16(k+6),size:dv.getInt16(k+8)}); }
    else if(p.code===20){ const len=dv.getUint16(o+2); for(let k=o+4;k+7<o+4+len+1;k+=8) featPk.push({ij:[dv.getInt16(k),dv.getInt16(k+2)],type:dv.getInt16(k+4),attr:dv.getInt16(k+6)}); }
  }
  // screen J axis points south; confirm against storm ID positions when an STI table is present
  const J=G.__jsign||-1;
  const at=(az,nm)=>azRanLL(L.lat,L.lon,az,nm);
  const out={code:L.code,lat:L.lat,lon:L.lon,vol:L.vol,storms:[],tvs:[],hail:[],meso:[],time:null};
  const ijLL=ij=>ijToLL(L.lat,L.lon,ij[0]/4,J*ij[1]/4);
  if(L.code===58){
    const {recs,time}=decodeSTI(L); out.time=time;
    for(const r of Object.values(recs)){
      if(!r.cur) continue;
      out.storms.push({id:r.id,pos:at(r.cur[0],r.cur[1]),fcst:r.fcst.map(f=>at(f[0],f[1])),dir:r.dir,kt:r.kt,isNew:r.dir==null,err:r.err,mean:r.mean,dbzm:r.dbzm,hgt:r.hgt});
    }
    // learn the J-axis sign from packet 15 vs. tabular azimuth (used for NHI/NMD/NTV packets)
    let dn=0,up=0; for(const s of pk){ const r=recs[s.id]; if(!r||!r.cur) continue; const az=r.cur[0];
      const df=a=>{ const d=Math.abs(((a+360)%360)-az)%360; return d>180?360-d:d; };
      const ad=Math.atan2(s.ij[0],-s.ij[1])*180/Math.PI, au=Math.atan2(s.ij[0],s.ij[1])*180/Math.PI; if(df(ad)<df(au)) dn++; else up++; }
    if(dn+up>=2) G.__jsign=up>dn?1:-1;
    if(!out.storms.length){
      for(const s of pk) out.storms.push({id:s.id,pos:ijLL(s.ij),fcst:[],dir:null,kt:null,isNew:true});
    }
  } else if(L.code===59){
    out.hail=hailPk.map(h=>({pos:ijLL(h.ij),pHail:h.pHail,pSvr:h.pSvr,size:h.size}));
    // storm IDs for hail come from the tabular page order when available
    const ids=[]; for(const page of tabularPages(L)) for(const line of page){ const mm=line.match(/STORM ID\s+(.*)$/); if(mm) for(const t of mm[1].trim().split(/\s+/)) if(/^[A-Z]\d$/.test(t)) ids.push(t); }
    if(ids.length===out.hail.length) out.hail.forEach((h,i)=>h.id=ids[i]);
  } else if(L.code===141||L.code===60){
    out.meso=featPk.map(f=>({pos:ijLL(f.ij),type:f.type,radiusKm:f.attr/4}));
  } else {
    const tab=decodeTVSTable(L);
    if(tab.length) out.tvs=tab.map(t=>({...t,pos:at(t.az,t.ran),elevated:t.type==='ETVS'}));
    else out.tvs=tvsPk.map(t=>({pos:ijLL(t.ij),elevated:t.elevated}));
  }
  return out;
}

/* ---- GLM (NetCDF4/HDF5 via jsfive) ---- */
let h5ready=null;
function loadH5(){
  if(h5ready) return h5ready;
  h5ready=new Promise((res,rej)=>{
    const urls=['https://cdn.jsdelivr.net/npm/jsfive@0.3.10/dist/browser/hdf5.js','https://unpkg.com/jsfive@0.3.10/dist/browser/hdf5.js','https://cdn.jsdelivr.net/npm/jsfive@0.3.10/dist/hdf5.js'];
    for(const u of urls){ try{ importScripts(u); if(G.hdf5&&G.hdf5.File) return res(G.hdf5); }catch(e){} }
    rej(new Error('HDF5 reader unavailable'));
  });
  return h5ready;
}
function h5var(f,name){
  let d; try{ d=f.get(name); }catch(e){ return null; } if(!d) return null; const v=d.value; const a=d.attrs||{};
  const one=x=>Array.isArray(x)||ArrayBuffer.isView(x)?x[0]:x;
  const sf=a.scale_factor!==undefined?one(a.scale_factor):1, ao=a.add_offset!==undefined?one(a.add_offset):0;
  const uns=String(one(a._Unsigned)||'')==='true';
  const fv=a._FillValue!==undefined?one(a._FillValue):null;
  const out=new Float32Array(v.length);
  for(let i=0;i<v.length;i++){ let x=v[i]; if(fv!==null&&x===fv){out[i]=NaN;continue;} if(uns&&x<0) x+=65536; out[i]=x*sf+ao; }
  return out;
}
async function decodeGLM(url,bbox,signal){
  const H=await loadH5();
  const r=await fetch(url,{signal}); if(!r.ok) throw new Error('HTTP '+r.status);
  const f=new H.File(await r.arrayBuffer(),'glm.nc');
  const la=h5var(f,'flash_lat'), lo=h5var(f,'flash_lon'), en=h5var(f,'flash_energy');
  const n=la?la.length:0; const lat=[],lon=[],e=[];
  for(let i=0;i<n;i++){ const y=la[i],x=lo[i]; if(!(y>=bbox[0]&&y<=bbox[2]&&x>=bbox[1]&&x<=bbox[3])) continue; lat.push(y); lon.push(x); e.push(en?en[i]:0); }
  return {lat:Float32Array.from(lat),lon:Float32Array.from(lon),energy:Float32Array.from(e),total:n};
}

/* ---- ODIM HDF5 polar data (DWD sweeps, FMI volumes) ---- */
async function decodeODIM(url,quants,tilt,vel,signal){
  const H=await loadH5();
  const r=await fetch(url,{signal}); if(!r.ok) throw new Error('HTTP '+r.status);
  const ab=await r.arrayBuffer();
  const f=new H.File(ab,'v.h5');
  const one=x=>Array.isArray(x)||ArrayBuffer.isView(x)?x[0]:x;
  const has=p=>{ try{ return !!f.get(p); }catch(e){ return false; } };
  const A=p=>{ try{ const g=f.get(p); return (g&&g.attrs)||{}; }catch(e){ return {}; } };
  const str=v=>{ v=one(v); return v==null?'':String(v).replace(/\0/g,'').trim(); };
  const rw=A('where'); const lat=+one(rw.lat), lon=+one(rw.lon);
  const cands=[];
  for(let k=1;k<300&&has('dataset'+k);k++){
    const w=A(`dataset${k}/where`), dw=A(`dataset${k}/what`);
    for(let j=1;j<64&&has(`dataset${k}/data${j}`);j++){
      const wh=A(`dataset${k}/data${j}/what`); const q=str(wh.quantity!=null?wh.quantity:dw.quantity);
      if(quants.includes(q)){ cands.push({k,j,el:+one(w.elangle),w,dw,wh,q}); break; }
    }
  }
  if(!cands.length) throw new Error('No '+quants[0]+' in file');
  cands.sort((a,b)=>a.el-b.el);
  const uniq=[]; for(const c of cands) if(!uniq.length||Math.abs(c.el-uniq[uniq.length-1].el)>0.15) uniq.push(c);
  const c=uniq[Math.min(tilt,uniq.length-1)];
  const ds=f.get(`dataset${c.k}/data${c.j}/data`), raw=ds.value, shape=ds.shape||[];
  const nrays=+one(c.w.nrays)||shape[0], nbins=+one(c.w.nbins)||shape[1];
  const pick=(n,d)=>{ for(const o of [c.wh,c.dw]) if(o[n]!=null) return +one(o[n]); return d; };
  const gain=pick('gain',1), off=pick('offset',0), nod=pick('nodata',NaN), und=pick('undetect',NaN);
  const rscale=+one(c.w.rscale)||250, rstart=+one(c.w.rstart)||0;
  const radials=[], w=360/nrays;
  for(let i=0;i<nrays;i++){
    const out=new Uint8Array(nbins), base=i*nbins;
    for(let g=0;g<nbins;g++){
      const x=raw[base+g]; if(x===nod||x===und) continue;
      const v=x*gain+off; let b;
      if(vel){ b=Math.round(v*2)+129; b=b<2?2:b>255?255:b; }
      else { b=Math.round((v+32)*2)+2; if(b<2) continue; if(b>255) b=255; }
      out[g]=b;
    }
    radials.push({start:i*w,width:w,data:out});
  }
  return { lat, lon, nGates:nbins, gateKm:rscale/1000, firstKm:rstart, grid:fillGrid(radials,nbins), elev:c.el,
    dec:vel?{t:'so',scale:2,offset:129,lead:2,mul:1.943844,rf:false}:{t:'so',scale:2,offset:66,lead:2,mul:1,rf:false},
    bytes:ab.byteLength, elevs:uniq.map(u=>u.el) };
}


/* ---- full Level II volume (all tilts of one moment) for cross sections ---- */
async function decodeL2Volume(src,block){
  const it=src[Symbol.asyncIterator]();
  let buf=new Uint8Array(1<<22), len=0, rp=24, done=false, bytes=0, lat=null, lon=null;
  const cuts=new Map(); // elNum -> {ang, radials, first, gate, scale, offset}
  async function need(n){ while(len-rp<n){ if(done) return false; const {value,done:d}=await it.next(); if(d){ done=true; return len-rp>=n; } bytes+=value.length;
      if(len+value.length>buf.length){ const keep=len-rp, nb=new Uint8Array(Math.max(buf.length,(keep+value.length)*2)); nb.set(buf.subarray(rp,len)); buf=nb; len=keep; rp=0; }
      buf.set(value,len); len+=value.length; } return true; }
  try{
    while(true){
      if(!(await need(4))) break; const size=Math.abs(new DataView(buf.buffer,buf.byteOffset+rp,4).getInt32(0)); rp+=4; if(!size) continue;
      if(!(await need(size))) break; const rec=bunzip2(buf.subarray(rp,rp+size)); rp+=size;
      const dv=new DataView(rec.buffer,rec.byteOffset,rec.byteLength); let o=0; const N=rec.length;
      while(o+28<=N){ const sz=dv.getUint16(o+12), type=dv.getUint8(o+15);
        if(type===31){ const d=o+28, az=dv.getFloat32(d+12), azRes=dv.getUint8(d+20), el=dv.getUint8(d+22), ang=dv.getFloat32(d+24), nb=dv.getUint16(d+30);
          let mom=null; for(let b=0;b<nb&&b<10;b++){ const p=dv.getUint32(d+32+b*4); if(!p) continue; const q=d+p; if(q+4>N) continue; const nm=String.fromCharCode(rec[q],rec[q+1],rec[q+2],rec[q+3]);
            if(nm==='RVOL'&&lat===null){ lat=dv.getFloat32(q+8); lon=dv.getFloat32(q+12); } else if(nm===block) mom=q; }
          if(mom!==null){ const ng=dv.getUint16(mom+8), ws=dv.getUint8(mom+19), sc=dv.getFloat32(mom+20), of=dv.getFloat32(mom+24);
            if(ws===8&&sc>0){ let c=cuts.get(el); if(!c){ c={ang:Math.round(ang*10)/10,radials:[],first:dv.getInt16(mom+10),gate:dv.getInt16(mom+12),scale:sc,offset:of}; cuts.set(el,c); }
              const w=azRes===1?0.5:1.0; c.radials.push({start:az-w/2,width:w,data:rec.slice(mom+28,mom+28+ng)}); } }
          o+=12+sz*2; } else o+=2432; }
    }
  } finally { try{ await it.return(); }catch(e){} }
  const list=[...cuts.entries()].sort((a,b)=>a[0]-b[0]).map(e=>e[1]).filter(c=>c.radials.length>100);
  const tilts=[]; for(const c of list){ if(tilts.some(t=>Math.abs(t.elev-c.ang)<0.15)) continue;
    let nG=0; for(const r of c.radials) if(r.data.length>nG) nG=r.data.length;
    tilts.push({elev:c.ang,nGates:nG,gateKm:c.gate/1000,firstKm:c.first/1000,grid:fillGrid(c.radials,nG),dec:{t:'so',scale:c.scale,offset:c.offset,lead:2,mul:(block==='DVEL'||block==='DSW ')?1.943844:1,rf:block==='DVEL'||block==='DSW '}}); }
  tilts.sort((a,b)=>a.elev-b.elev);
  if(!tilts.length) throw new Error('No '+block.trim()+' in volume');
  return {lat,lon,tilts,bytes};
}
/* ---- Sigmet / Vaisala IRIS RAW (IDEAM) ---- */
// encodings shared with the main thread: v=(code-offset)/scale*mul
const ENC={REF:{scale:2,offset:66,mul:1,rf:false},VEL:{scale:2,offset:129,mul:1.943844,rf:false},ZDR:{scale:16,offset:130,mul:1,rf:false},CC:{scale:200,offset:0,mul:1,rf:false},KDP:{scale:20,offset:60,mul:1,rf:false},SW:{scale:20,offset:2,mul:1.943844,rf:false}};
const IRIS_TYPES={REF:[9,2,21,7,8,1],VEL:[10,22],ZDR:[12,5],CC:[20,19],KDP:[15],SW:[11]};
function irisValue(type,N){
  if(N===0) return NaN;
  switch(type){
    case 1: case 2: case 7: return (N-64)/2;
    case 8: case 9: case 21: case 10: case 22: case 12: case 15: return (N-32768)/100;
    case 5: return (N-128)/16;
    case 19: return Math.sqrt(Math.max(0,N-1)/253);
    case 20: return (N-1)/65533;
    case 11: return N/100;
  } return NaN;
}
function decodeIRIS(u8,prod,tilt){
  const RS=6144, dv=new DataView(u8.buffer,u8.byteOffset,u8.byteLength);
  if(dv.getInt16(0,true)!==27) throw new Error('Not an IRIS RAW file');
  const ic=RS+12, tc=RS+12+480, ri=tc+12+120+320+320;
  let lat=dv.getUint32(ic+168,true)*360/4294967296, lon=dv.getUint32(ic+172,true)*360/4294967296; if(lat>180) lat-=360; if(lon>180) lon-=360;
  const firstKm=dv.getInt32(ri,true)/100000, stepKm=dv.getInt32(ri+16,true)/100000;
  // gather each sweep's payload (records minus their 12-byte headers)
  const sweeps=new Map();
  for(let r=2;(r+1)*RS<=u8.length;r++){ const base=r*RS, sw=dv.getInt16(base+2,true); if(sw<=0) continue; (sweeps.get(sw)||sweeps.set(sw,[]).get(sw)).push(u8.subarray(base+12,base+RS)); }
  const want=IRIS_TYPES[prod]||IRIS_TYPES.REF, enc=ENC[prod]||ENC.REF, found=[];
  for(const [sw,parts] of sweeps){
    const len=parts.reduce((a,p)=>a+p.length,0), s=new Uint8Array(len); let o=0; for(const p of parts){ s.set(p,o); o+=p.length; }
    const d=new DataView(s.buffer); let pos=0; const hdrs=[];
    while(pos+76<=s.length&&d.getInt16(pos,true)===24){ hdrs.push({fixed:d.getUint16(pos+34,true)*360/65536,bits:d.getInt16(pos+36,true),type:d.getUint16(pos+38,true),nrays:d.getInt16(pos+30,true)}); pos+=76; }
    if(!hdrs.length) continue;
    const ti=hdrs.findIndex(h=>want.includes(h.type)); if(ti<0) continue;
    // choose the most preferred type present
    let best=-1,bp=99; hdrs.forEach((h,i)=>{ const p=want.indexOf(h.type); if(p>=0&&p<bp){ bp=p; best=i; } });
    found.push({sw,s,d,pos,hdrs,idx:best,fixed:hdrs[best].fixed});
  }
  if(!found.length) throw new Error('No '+prod+' in this IRIS file');
  found.sort((a,b)=>a.fixed-b.fixed);
  const uniq=[]; for(const f of found) if(!uniq.length||Math.abs(f.fixed-uniq[uniq.length-1].fixed)>0.15) uniq.push(f);
  const F=uniq[Math.min(tilt,uniq.length-1)], h=F.hdrs[F.idx], nT=F.hdrs.length, d=F.d, s=F.s;
  // walk the compressed ray stream: rays for each data type are interleaved in header order
  let pos=F.pos; const radials=[]; let nG=0;
  const words=[];
  const readRay=()=>{ words.length=0;
    while(pos+2<=s.length){ const w=d.getUint16(pos,true); pos+=2;
      if(w===1) break;
      if(w&0x8000){ const n=w&0x7fff; for(let k=0;k<n&&pos+2<=s.length;k++){ words.push(d.getUint16(pos,true)); pos+=2; } }
      else { for(let k=0;k<w;k++) words.push(0); } }
    return words.length; };
  for(let ray=0;ray<4000&&pos<s.length;ray++){
    for(let t=0;t<nT;t++){
      const n=readRay(); if(t!==F.idx||n<6) continue;
      const az0=words[0]*360/65536, az1=words[2]*360/65536, nb=words[4]; if(nb<=0) continue;
      const out=new Uint8Array(nb), sixteen=h.bits===16;
      for(let g=0;g<nb;g++){
        let N; if(sixteen){ N=words[6+g]; if(N===undefined) break; } else { const w=words[6+(g>>1)]; if(w===undefined) break; N=(g&1)?(w>>8):(w&255); }
        const v=irisValue(h.type,N); if(isNaN(v)) continue;
        let c=Math.round(v*enc.scale+enc.offset); if(c<2) c=prod==='REF'?0:2; if(c>255) c=255; out[g]=c;
      }
      let wdt=(az1-az0+360)%360; if(wdt<=0||wdt>2) wdt=360/(h.nrays>0?h.nrays:360);
      radials.push({start:az0,width:wdt,data:out}); if(nb>nG) nG=nb;
    }
    if(pos>=s.length-2) break;
  }
  if(!radials.length) throw new Error('No rays decoded');
  return {lat,lon,nGates:nG,gateKm:stepKm,firstKm:Math.max(0,firstKm-stepKm/2),grid:fillGrid(radials,nG),elev:Math.round(F.fixed*10)/10,
    dec:{t:'so',scale:enc.scale,offset:enc.offset,lead:2,mul:enc.mul,rf:false},bytes:u8.length,elevs:uniq.map(u=>Math.round(u.fixed*10)/10)};
}

/* ---- OPERA composite (ODIM HDF5, LAEA grid) resampled to a Web Mercator raster ---- */
function laeaFromProj(pd){
  const g=k=>{ const m=pd.match(new RegExp('\\+'+k+'=([^\\s]+)')); return m?m[1]:null; };
  const lat0=+(g('lat_0')||55)*Math.PI/180, lon0=+(g('lon_0')||10)*Math.PI/180, x0=+(g('x_0')||0), y0=+(g('y_0')||0);
  let a=6378137, f=1/298.257223563; if(g('R')){ a=+g('R'); f=0; } else if(g('a')){ a=+g('a'); const b=g('b')?+g('b'):a; f=(a-b)/a; } else if(/ellps=sphere/i.test(pd)){ a=6370997; f=0; }
  const e2=f*(2-f), e=Math.sqrt(e2);
  if(e<1e-9){ return (lon,lat)=>{ const p=lat*Math.PI/180, dl=lon*Math.PI/180-lon0, k=Math.sqrt(2/(1+Math.sin(lat0)*Math.sin(p)+Math.cos(lat0)*Math.cos(p)*Math.cos(dl))); return [a*k*Math.cos(p)*Math.sin(dl)+x0, a*k*(Math.cos(lat0)*Math.sin(p)-Math.sin(lat0)*Math.cos(p)*Math.cos(dl))+y0]; }; }
  const q=p=>{ const s=Math.sin(p); return (1-e2)*(s/(1-e2*s*s)-(1/(2*e))*Math.log((1-e*s)/(1+e*s))); };
  const qp=q(Math.PI/2), Rq=a*Math.sqrt(qp/2), b1=Math.asin(q(lat0)/qp), D=a*(Math.cos(lat0)/Math.sqrt(1-e2*Math.sin(lat0)**2))/(Rq*Math.cos(b1));
  return (lon,lat)=>{ const b=Math.asin(Math.max(-1,Math.min(1,q(lat*Math.PI/180)/qp))), dl=lon*Math.PI/180-lon0, B=Rq*Math.sqrt(2/(1+Math.sin(b1)*Math.sin(b)+Math.cos(b1)*Math.cos(b)*Math.cos(dl)));
    return [B*D*Math.cos(b)*Math.sin(dl)+x0,(B/D)*(Math.cos(b1)*Math.sin(b)-Math.sin(b1)*Math.cos(b)*Math.cos(dl))+y0]; };
}
async function decodeOperaComp(url,outW,signal){
  const H=await loadH5(); const r=await fetch(url,{signal}); if(!r.ok) throw new Error('HTTP '+r.status);
  const f=new H.File(await r.arrayBuffer(),'c.h5'), one=x=>Array.isArray(x)||ArrayBuffer.isView(x)?x[0]:x;
  const A=p=>{ try{ const g=f.get(p); return (g&&g.attrs)||{}; }catch(e){ return {}; } };
  const w=A('where'), str=v=>String(one(v)||'').replace(/\0/g,'');
  const pd=str(w.projdef), nx=+one(w.xsize), ny=+one(w.ysize), sx=+one(w.xscale), sy=+one(w.yscale);
  let dsPath=null, wh={}; for(let k=1;k<8&&!dsPath;k++) for(let j=1;j<8;j++){ const a=A(`dataset${k}/data${j}/what`); const qn=str(a.quantity||A(`dataset${k}/what`).quantity); if(/DBZH|TH/.test(qn)||(!qn&&j===1)){ dsPath=`dataset${k}/data${j}`; wh=Object.assign({},A(`dataset${k}/what`),a); break; } }
  if(!dsPath) throw new Error('No DBZH in composite');
  const raw=f.get(dsPath+'/data').value, gain=+one(wh.gain)||1, off=+one(wh.offset)||0, nod=one(wh.nodata), und=one(wh.undetect);
  const fwd=laeaFromProj(pd), ul=fwd(+one(w.UL_lon),+one(w.UL_lat));
  const corners=[['UL_lat','UL_lon'],['UR_lat','UR_lon'],['LL_lat','LL_lon'],['LR_lat','LR_lon']].map(([a,b])=>[+one(w[a]),+one(w[b])]);
  const S=Math.min(...corners.map(c=>c[0])), N=Math.max(...corners.map(c=>c[0]))+2, Wl=Math.min(...corners.map(c=>c[1])), E=Math.max(...corners.map(c=>c[1]));
  const my=lat=>Math.log(Math.tan(Math.PI/4+lat*Math.PI/360)), yS=my(S), yN=my(N), W_=outW||1800, H_=Math.round(W_*(yN-yS)/((E-Wl)*Math.PI/180));
  const codes=new Uint8Array(W_*H_);
  for(let j=0;j<H_;j++){ const lat=(2*Math.atan(Math.exp(yN-(j+.5)/H_*(yN-yS)))-Math.PI/2)*180/Math.PI;
    for(let i=0;i<W_;i++){ const lon=Wl+(i+.5)/W_*(E-Wl), p=fwd(lon,lat), gi=Math.floor((p[0]-ul[0])/sx), gj=Math.floor((ul[1]-p[1])/sy);
      if(gi<0||gj<0||gi>=nx||gj>=ny) continue; const x=raw[gj*nx+gi]; if(x===nod||x===und) continue; const v=x*gain+off; let c=Math.round((v+32)*2)+2; if(c<2) continue; codes[j*W_+i]=c>255?255:c; } }
  return {w:W_,h:H_,codes,bounds:[[S,Wl],[N,E]]};
}
/* ---- dispatcher ---- */
const jobs=new Map();
if(typeof importScripts==='function'){
  G.onmessage=async ev=>{
    const {id,type,url,opts,cancel}=ev.data;
    if(cancel){ const j=jobs.get(id); if(j) j.abort(); return; }
    const ac=new AbortController(); jobs.set(id,ac);
    try{
      let result, transfer=[];
      if(type==='l2'){ result=await decodeL2(urlStream(url,ac.signal),opts.block,opts.tilt||0,false); transfer=[result.grid.buffer]; }
      else if(type==='l2c'){ result=await decodeL2(chunkStream(opts.urls,ac.signal),opts.block,opts.tilt||0,true); transfer=[result.grid.buffer]; }
      else if(type==='l3'){ const r=await fetch(url,{signal:ac.signal}); if(!r.ok) throw new Error('HTTP '+r.status); const u=new Uint8Array(await r.arrayBuffer()); result=decodeL3Radar(u); result.bytes=u.length; transfer=[result.grid.buffer]; }
      else if(type==='cells'){ const r=await fetch(url,{signal:ac.signal}); if(!r.ok) throw new Error('HTTP '+r.status); result=decodeL3Cells(new Uint8Array(await r.arrayBuffer())); }
      else if(type==='l2vol'){ result=await decodeL2Volume(opts.urls?chunkStream(opts.urls,ac.signal):urlStream(url,ac.signal),opts.block); transfer=result.tilts.map(t=>t.grid.buffer); }
      else if(type==='iris'){ const r=await fetch(url,{signal:ac.signal}); if(!r.ok) throw new Error('HTTP '+r.status); result=decodeIRIS(new Uint8Array(await r.arrayBuffer()),opts.prod||'REF',opts.tilt||0); transfer=[result.grid.buffer]; }
      else if(type==='operacomp'){ result=await decodeOperaComp(url,opts.w,ac.signal); transfer=[result.codes.buffer]; }
      else if(type==='odim'){ result=await decodeODIM(url,opts.quants,opts.tilt||0,!!opts.vel,ac.signal); transfer=[result.grid.buffer]; }
      else if(type==='glm'){ result=await decodeGLM(url,opts.bbox,ac.signal); transfer=[result.lat.buffer,result.lon.buffer,result.energy.buffer]; }
      G.postMessage({id,ok:true,result},transfer);
    }catch(e){ G.postMessage({id,ok:false,error:String(e&&e.message||e)}); }
    finally{ jobs.delete(id); }
  };
}
if(typeof module!=='undefined') module.exports={decodeIRIS,laeaFromProj,decodeL2,decodeL3Radar,decodeL3Cells,legacyTable,f16,decodeSTI,tabularPages};
</script>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
"use strict";
/* ================= constants ================= */
const S3={ l2:'https://unidata-nexrad-level2.s3.amazonaws.com', l2rt:'https://unidata-nexrad-level2-chunks.s3.amazonaws.com', l3:'https://unidata-nexrad-level3.s3.amazonaws.com', g19:'https://noaa-goes19.s3.amazonaws.com', g18:'https://noaa-goes18.s3.amazonaws.com' };
const SPEEDS=[0.25,0.5,0.75,1,1.25,1.5,2,3,4,5,6,8,10,12,15,18];
const CATALOG=[
  {id:'REF',label:'REF',name:'Base Reflectivity',pal:'REF',l2:'DREF',l3:['N0B','N1B','N2B','N3B'],fb:['N0Q','N1Q','N2Q','N3Q'],tdwr:['TZL','TR0','TR1','TR2'],tdwrNames:['Long range','Tilt 1','Tilt 2','Tilt 3']},
  {id:'VEL',label:'VEL',name:'Base Velocity',pal:'VEL',l2:'DVEL',l3:['N0G','N1G','N2G','N3G'],fb:['N0U','N1U','N2U','N3U'],tdwr:['TV0','TV1','TV2']},
  {id:'SRV',label:'SRV',name:'Storm-Relative Velocity',pal:'VEL',l3:['N0S','N1S','N2S','N3S']},
  {id:'SW',label:'SW',name:'Spectrum Width',pal:'SW',l2:'DSW '},
  {id:'ZDR',label:'ZDR',name:'Differential Reflectivity',pal:'ZDR',l2:'DZDR',l3:['N0X','N1X','N2X','N3X']},
  {id:'CC',label:'CC',name:'Correlation Coefficient',pal:'CC',l2:'DRHO',l3:['N0C','N1C','N2C','N3C']},
  {id:'KDP',label:'KDP',name:'Specific Differential Phase',pal:'KDP',l3:['N0K','N1K','N2K','N3K']},
  {id:'HCA',label:'HCA',name:'Hydrometeor Classification',pal:'HCA',l3:['N0H','N1H','N2H','N3H']},
  {id:'HHC',label:'HHC',name:'Hybrid Hydrometeor Class',pal:'HCA',l3:['HHC']},
  {id:'CR',label:'COMP',name:'Composite Reflectivity',pal:'REF',l3:['NCR'],tdwr:['NCR']},
  {id:'VIL',label:'VIL',name:'Vertically Integrated Liquid',pal:'VIL',l3:['DVL'],fb:['NVL'],tdwr:['NVL']},
  {id:'ET',label:'TOPS',name:'Echo Tops',pal:'ET',l3:['EET'],fb:['NET'],tdwr:['NET']},
  {id:'P1',label:'1H PCP',name:'1-Hour Precipitation',pal:'PCP',l3:['DAA'],fb:['N1P'],tdwr:['N1P']},
  {id:'P3',label:'3H PCP',name:'3-Hour Precipitation',pal:'PCP',l3:['DU3']},
  {id:'P24',label:'24H PCP',name:'24-Hour Precipitation',pal:'PCP',l3:['DU6']},
  {id:'PST',label:'STORM TOT',name:'Storm-Total Precipitation',pal:'PCP',l3:['DTA'],fb:['NTP'],tdwr:['NTP']}
];
const famById=id=>CATALOG.find(f=>f.id===id)||CATALOG[0];
const famAvail=(f,site,src)=>{ if(isMrrl(site)) return famAnywhere(f,site); const td=site&&site.type==='TDWR'; if(td) return src==='L3'&&!!f.tdwr; return src==='L2'?!!f.l2:!!f.l3; };
const famAnywhere=(f,site)=>isMrrl(site)?(f.id==='REF'||(mrrlRaw(site)&&(f.id==='VEL'||(site.src==='ideam'&&['ZDR','CC','KDP','SW'].includes(f.id))))):(site&&site.type==='TDWR'?!!f.tdwr:true);
const FALLBACK_SITES=`KTLX 35.333 -97.278 Oklahoma City OK|KFWS 32.573 -97.303 Dallas/Fort Worth TX|KICT 37.654 -97.443 Wichita KS|KAMA 35.233 -101.709 Amarillo TX|KLBB 33.654 -101.814 Lubbock TX|KDDC 37.761 -99.969 Dodge City KS|KINX 36.175 -95.564 Tulsa OK|KVNX 36.741 -98.128 Vance AFB OK|KFDR 34.362 -98.976 Frederick OK|KSRX 35.290 -94.362 Fort Smith AR|KLZK 34.836 -92.262 Little Rock AR|KSHV 32.451 -93.841 Shreveport LA|KLCH 30.125 -93.216 Lake Charles LA|KHGX 29.472 -95.079 Houston TX|KEWX 29.704 -98.028 Austin/San Antonio TX|KGRK 30.722 -97.383 Fort Hood TX|KDYX 32.538 -99.254 Dyess AFB TX|KMAF 31.943 -102.189 Midland TX|KLSX 38.699 -90.683 St. Louis MO|KEAX 38.810 -94.264 Kansas City MO|KSGF 37.235 -93.400 Springfield MO|KTWX 38.997 -96.232 Topeka KS|KOAX 41.320 -96.367 Omaha NE|KUEX 40.321 -98.442 Hastings NE|KLNX 41.958 -100.576 North Platte NE|KGLD 39.367 -101.700 Goodland KS|KDMX 41.731 -93.723 Des Moines IA|KDVN 41.612 -90.581 Quad Cities IA|KLOT 41.605 -88.085 Chicago IL|KILX 40.151 -89.337 Lincoln IL|KIND 39.708 -86.280 Indianapolis IN|KIWX 41.359 -85.700 North Webster IN|KLVX 37.975 -85.944 Louisville KY|KPAH 37.068 -88.772 Paducah KY|KOHX 36.247 -86.563 Nashville TN|KNQA 35.345 -89.873 Memphis TN|KHTX 34.931 -86.084 Huntsville AL|KBMX 33.172 -86.770 Birmingham AL|KGWX 33.897 -88.329 Columbus AFB MS|KDGX 32.280 -89.984 Jackson MS|KFFC 33.364 -84.566 Atlanta GA|KMOB 30.679 -88.240 Mobile AL|KLIX 30.337 -89.826 New Orleans LA|KTLH 30.398 -84.329 Tallahassee FL|KJAX 30.485 -81.702 Jacksonville FL|KTBW 27.705 -82.402 Tampa Bay FL|KMLB 28.113 -80.654 Melbourne FL|KAMX 25.611 -80.413 Miami FL|KCLX 32.656 -81.042 Charleston SC|KGSP 34.883 -82.220 Greer SC|KRAX 35.665 -78.490 Raleigh NC|KMHX 34.776 -76.876 Morehead City NC|KAKQ 36.984 -77.007 Wakefield VA|KLWX 38.976 -77.487 Sterling VA|KDOX 38.826 -75.440 Dover AFB DE|KDIX 39.947 -74.411 Philadelphia PA|KOKX 40.865 -72.864 New York City NY|KBOX 41.956 -71.137 Boston MA|KENX 42.586 -74.064 Albany NY|KBGM 42.200 -75.985 Binghamton NY|KBUF 42.949 -78.737 Buffalo NY|KCLE 41.413 -81.860 Cleveland OH|KILN 39.420 -83.822 Wilmington OH|KPBZ 40.532 -80.218 Pittsburgh PA|KDTX 42.700 -83.472 Detroit MI|KGRR 42.894 -85.545 Grand Rapids MI|KMKX 42.968 -88.551 Milwaukee WI|KGRB 44.498 -88.111 Green Bay WI|KMPX 44.849 -93.565 Minneapolis MN|KDLH 46.837 -92.210 Duluth MN|KFSD 43.588 -96.729 Sioux Falls SD|KABR 45.456 -98.413 Aberdeen SD|KBIS 46.771 -100.761 Bismarck ND|KMVX 47.528 -97.325 Grand Forks ND|KUDX 44.125 -102.830 Rapid City SD|KCYS 41.152 -104.806 Cheyenne WY|KFTG 39.787 -104.546 Denver CO|KPUX 38.460 -104.181 Pueblo CO|KABX 35.150 -106.824 Albuquerque NM|KFDX 34.634 -103.619 Cannon AFB NM|KEPZ 31.873 -106.698 El Paso TX|KEMX 31.894 -110.630 Tucson AZ|KIWA 33.289 -111.670 Phoenix AZ|KESX 35.701 -114.891 Las Vegas NV|KSOX 33.818 -117.636 Santa Ana Mtns CA|KVTX 34.412 -119.179 Los Angeles CA|KMUX 37.155 -121.898 San Francisco CA|KDAX 38.501 -121.678 Sacramento CA|KRTX 45.715 -122.965 Portland OR|KATX 48.195 -122.496 Seattle WA|KMTX 41.263 -112.448 Salt Lake City UT|KLRX 40.740 -116.803 Elko NV|KBLX 45.854 -108.607 Billings MT|KTFX 47.460 -111.385 Great Falls MT`
+`|TOKC 35.276 -97.510 Oklahoma City OK|TTUL 36.071 -95.827 Tulsa OK|TICH 37.507 -97.437 Wichita KS|TDFW 33.065 -96.918 Dallas/Fort Worth TX|TDAL 32.926 -96.968 Dallas Love Field TX|TIAH 30.065 -95.567 Houston Intercontinental TX|THOU 29.516 -95.242 Houston Hobby TX|TATL 33.647 -84.262 Atlanta GA|TORD 41.797 -87.858 Chicago O'Hare IL|TMDW 41.651 -87.730 Chicago Midway IL|TSTL 38.805 -90.489 St. Louis MO|TMCI 39.498 -94.742 Kansas City MO|TBNA 35.980 -86.662 Nashville TN|TMEM 34.896 -89.993 Memphis TN|TMSY 30.022 -90.403 New Orleans LA|TMIA 25.758 -80.491 Miami FL|TFLL 26.143 -80.344 Fort Lauderdale FL|TPBI 26.688 -80.273 West Palm Beach FL|TMCO 28.344 -81.326 Orlando FL|TTPA 27.860 -82.518 Tampa FL|TCLT 35.337 -80.885 Charlotte NC|TRDU 36.002 -78.697 Raleigh-Durham NC|TDCA 38.759 -76.962 Washington National MD|TIAD 39.084 -77.529 Washington Dulles VA|TADW 38.695 -76.845 Andrews AFB MD|TBWI 39.090 -76.630 Baltimore MD|TPHL 39.949 -75.069 Philadelphia PA|TEWR 40.593 -74.270 Newark NJ|TJFK 40.589 -73.881 New York JFK NY|TBOS 42.158 -70.933 Boston MA|TPIT 40.501 -80.486 Pittsburgh PA|TCLE 41.290 -82.008 Cleveland OH|TCMH 40.006 -82.715 Columbus OH|TCVG 38.898 -84.580 Cincinnati KY|TDAY 40.022 -84.123 Dayton OH|TDTW 42.111 -83.515 Detroit MI|TIDS 39.637 -86.436 Indianapolis IN|TSDF 38.046 -85.611 Louisville KY|TMKE 42.819 -88.046 Milwaukee WI|TMSP 44.871 -92.933 Minneapolis MN|TDEN 39.728 -104.526 Denver CO|TPHX 33.421 -112.163 Phoenix AZ|TLAS 36.144 -115.007 Las Vegas NV|TSJU 18.474 -66.179 San Juan PR`;

/* ================= MRRL (custom radar list) ================= */
const MRRL_RAW=[["MF36",48.04694,-2.89417,"FR","FR","NOYAL"],["MF37",41.95306,8.70056,"FR","FR","AJACCIO"],["MF38",46.06639,2.96056,"FR","FR","ST-REMY"],["MF40",50.13583,1.83472,"FR","FR","ABBEVILLE"],["MF41",44.83139,-0.69194,"FR","FR","BORDEAUX"],["MF42",47.05861,2.35944,"FR","FR","BOURGES"],["MF43",45.14778,5.63944,"FR","FR","MOUCHEROTTE"],["MF44",45.10444,1.36972,"FR","FR","BRIVE GREZES"],["MF45",48.92722,-0.14944,"FR","FR","FALAISE CAEN"],["MF47",48.71583,6.58167,"FR","FR","NANCY"],["MF49",43.80611,4.50278,"FR","FR","NIMES"],["MF50",43.57444,1.37611,"FR","FR","TOULOUSE"],["MF51",48.77444,2.00833,"FR","FR","TRAPPES"],["MF52",48.46222,4.30944,"FR","FR","ARCIS TROYES"],["MF53",45.28917,3.70944,"FR","FR","SEMBADEL"],["MF54",47.3375,-1.65639,"FR","FR","TREILLIERES"],["MF55",44.32306,4.76222,"FR","FR","BOLLENE"],["MF56",48.46083,-4.42972,"FR","FR","PLABENNEC"],["MF57",42.91833,2.865,"FR","FR","OPOUL"],["MF58",46.06778,4.44528,"FR","FR","ST.NIZIER"],["MF59",43.21667,6.37278,"FR","FR","COLLOBRIERES"],["MF61",42.12972,9.49639,"FR","FR","ALERIA"],["MF62",43.99056,2.60972,"FR","FR","MONTCLAR"],["MF63",50.12833,3.81194,"FR","FR","L'AVESNOIS"],["MF64",46.69861,0.06556,"FR","FR","CHERVES"],["MF65",47.35528,4.77583,"FR","FR","BLAISY-HAUT"],["MF66",43.62444,-0.60944,"FR","FR","MOMUY"],["MF67",47.36861,7.01889,"FR","FR","MONTANCY"],["MF68",44.01278,6.52917,"FR","FR","MAUREL"],["MF69",44.49611,6.22056,"FR","FR","COLOMBIS"],["MF90",16.31444,-61.34889,"GP","GP","GUADELOUPE LE MOULE"],["MF91",14.50139,-61.0175,"MQ","MQ","MARTINIQUE"],["MF92",-20.91167,55.42194,"RE","RE","LA RÉUNION COLORADO"],["MF93",-21.19056,55.57361,"RE","RE","LA REUNION PITON VILLERS"],["MF94",-22.27611,166.45278,"NC","NC","NOUVELLE-CALÉDONIE NOUMEA"],["MF96",-20.93889,167.21806,"NC","NC","NOUVELLE-CALÉDONIE LIFOU"],["CANC",21.02901,-86.8521,"MX","MX","Cancun"],["GUAS",25.5675,-108.46333,"MX","MX","Guasave"],["MEXI",19.55,-99.52083,"MX","MX","Catedral"],["SABA",18.97242,-91.17262,"MX","MX","Sabancuy"],["CABO",22.88137,-109.92679,"MX","MX","Los Cabos"],["ALTA",22.3876,-97.9254,"MX","MX","Altamira"],["BAR_",47.0118,27.5825,"RO","RO","Poiana cu Cetate"],["BOB_",46.3602,24.2252,"RO","RO","Bobohalma"],["MED_",44.2434,28.2506,"RO","RO","Medgidia"],["TIM_",45.7717,21.2577,"RO","RO","Timișoara"],["ORA_",47.0922,21.9429,"RO","RO","Oradea"],["CRA_",44.31029,23.86743,"RO","RO","Craiova"],["BUC_",44.5127,26.0773,"RO","RO","Bucharest"],["SJAV",48.25564,17.15238,"CA","SK","Svätý Jur"],["SKOJ",48.78273,20.98733,"CA","SK","Kojšov"],["SKUB",49.27167,19.24935,"CA","SK","Dolný Kubín"],["SLAZ",48.24041,19.25732,"CA","SK","Horné Plachtince"],["PLEG",52.4053,20.9611,"PL","PL","Legionowo"],["PRZE",50.1139,22.0367,"PL","PL","Rzeszów"],["PBRZ",50.3942,20.0833,"PL","PL","Brzuchania"],["PRAM",50.1514,18.725,"PL","PL","Ramża"],["PPAS",50.8925,16.0394,"PL","PL","Pastewnik"],["PPOZ",52.4131,16.7969,"PL","PL","Poznań"],["PSWI",53.7944,15.8381,"PL","PL","Świdwin"],["PGDY",54.3842,18.4564,"PL","PL","Gdańsk"],["PGSA",50.46386,18.15321,"PL","PL","Leśnica"],["PUZR",53.85573,21.41233,"PL","PL","Użranki"],["SAPP",43.13889,141.00972,"JP","JP","Sapporo"],["KUSH",42.96083,144.5175,"JP","JP","Kushiro"],["HAKO",41.93361,140.78139,"JP","JP","Hakodate"],["SEND",38.26222,140.89667,"JP","JP","Sendai"],["AKIT",39.71797,140.09928,"JP","JP","Akita"],["KASH",35.85972,139.95972,"JP","JP","Kashiwa-Tokyo"],["YAHI",37.71861,138.81611,"JP","JP","Yahiko"],["TOJI",36.2375,136.1425,"JP","JP","Tojinbo"],["KURU",36.10306,138.19583,"JP","JP","Kurumayama"],["MAKI",34.74262,138.1337,"JP","JP","Makinohara"],["NAGO",35.16833,136.96472,"JP","JP","Nagoya"],["TAKA",34.61611,135.65611,"JP","JP","Mt. Takayasu"],["MISA",35.54167,133.10333,"JP","JP","Mt. Misaka"],["HAIG",34.27028,132.59333,"JP","JP","Haigamine (Hiroshima)"],["MURO",33.25222,134.17722,"JP","JP","Muroto"],["SEFU",33.43472,130.35694,"JP","JP","Mt. Sefuri"],["TANE",30.63972,130.97917,"JP","JP","Tanegashima"],["FUNC",28.39417,129.55194,"JP","JP","Naze"],["ITOK",26.15333,127.765,"JP","JP","Okinawa"],["ISHI",24.42667,124.18222,"JP","JP","Ishigaki"],["NAND",69.2414,16.0028,"NO","NO","Andoya"],["NBER",70.5107,29.0184,"NO","NO","Berlevaag"],["NBML",59.854,5.09,"NO","NO","Boemlo"],["NHAS",70.6052,22.443,"NO","NO","Hasvik"],["NHFJ",61.2318,10.5273,"NO","NO","Hafjell"],["NHGB",58.3601,7.1648,"NO","NO","Haegebostad"],["NHUR",59.6272,10.5645,"NO","NO","Hurum"],["NRSA",63.6905,10.2037,"NO","NO","Rissa"],["NRSG",69.2186,23.4397,"NO","NO","Rassegalvarri"],["NRST",67.5307,12.0986,"NO","NO","Rost"],["NSMN",65.2201,11.9926,"NO","NO","Soemna"],["NSTA",62.1871,5.1275,"NO","NO","Stad"],["DBOR",55.1127,14.8875,"DK","DK","Bornholm"],["DROM",55.1731,8.552,"DK","DK","Römö"],["DSAM",55.8119,10.5853,"DK","DK","Samsø"],["DSIN",57.4893,10.1365,"DK","DK","Sindal"],["DSTE",55.3262,12.4493,"DK","DK","Stevns"],["FI49",63.1048,23.8209,"FI","FI","Vimpeli"],["FI53",60.5562,24.4956,"FI","FI","Vihti"],["FI47",64.7745,26.3188,"FI","FI","Alakylä"],["FI51",62.3045,25.4409,"FI","FI","Petäjävesi"],["FI52",63.8378,29.4489,"FI","FI","Mujejärvi"],["FI48",67.1391,26.8962,"FI","FI","Aarniluosto"],["FI45",62.8659,27.3814,"FI","FI","Kuopio"],["FI46",60.1284,21.6433,"FI","FI","Inikorp"],["FI50",61.9069,29.7977,"FI","FI","Kotalahti"],["FI54",61.8108,22.502,"FI","FI","Ylinen"],["FI44",60.9038,27.108,"FI","FI","Kaipiainen"],["FI55",68.43443,27.44399,"FI","FI","Saariselkä"],["ASB_",53.56401,6.74829,"DE","DE","ASR Borkum"],["BOO_",54.00438,10.0469,"DE","DE","Boostedt"],["DRS_",51.12464,13.76864,"DE","DE","Dresden"],["EIS_",49.54067,12.40279,"DE","DE","Eisberg"],["ESS_",51.40565,6.96711,"DE","DE","Essen"],["FBG_",47.87361,8.00361,"DE","DE","Feldberg"],["FLD_",51.3112,8.802,"DE","DE","Flechtdorf"],["HNR_",52.46008,9.69453,"DE","DE","Hannover"],["ISN_",48.17471,12.10178,"DE","DE","Isen"],["MEM_",48.04214,10.21922,"DE","DE","Memmingen"],["NEU_",50.50011,11.13503,"DE","DE","Neuhaus"],["NHB_",50.10966,6.54833,"DE","DE","Neuheilenbach"],["OFT_",49.98474,8.71293,"DE","DE","Offenthal"],["PRO_",52.64867,13.85821,"DE","DE","Prötzel"],["ROS_",54.17566,12.05808,"DE","DE","Rostock"],["TUR_",48.58538,9.78267,"DE","DE","Türkheim"],["UMD_",52.16009,11.17609,"DE","DE","Ummendorf"],["FLER",45.48145,10.17676,"IT","IT","Flero"],["DESI",45.62727,9.19632,"IT","IT","Desio"],["BRIC",45.03417,7.73272,"IT","IT","Bric della Croce"],["SETT",44.245,8.19778,"IT","IT","Monte Settepani"],["TVDR",45.36278,11.67361,"IT","IT","Teolo Monte Grande"],["ILON",45.78,12.76,"IT","IT","Concordia Sagittaria"],["IFOS",45.72666,13.4775,"IT","IT","Fossalon"],["SI42",46.09798,14.22824,"SI","SI","Pasjaravan"],["SI41",46.06777,15.2849,"SI","SI","Sevnica"],["HBIL",45.8835,17.2005,"HR","HR","Bilogora"],["HDEB",44.0455,15.3765,"HR","HR","Debeljak"],["HGRA",45.1592,18.7033,"HR","HR","Gradište"],["HPUN",45.9078,15.9683,"HR","HR","Puntijarka"],["HULJ",42.8944,17.4783,"HR","HR","Uljenje"],["HGOL",45.0205,14.1223,"HR","HR","Gora"],["IDUB",53.4299,-6.2443,"IE","IE","Dublin"],["ISHA",52.69279,-8.91999,"IE","IE","Shannon"],["ESUR",58.4823,25.5187,"EE","EE","Surgavere"],["EHAR",59.3971,24.6021,"EE","EE","Harku"],["ISKA",66.05573,-20.26798,"IS","IS","Lake Selvatn"],["IBJO",65.26588,-14.0618,"IS","IS","Seydisfijordur"],["IKEF",64.0257,-22.63536,"IS","IS","Sandgeroi"],["ISX2",63.86958,-20.2024,"IS","IS","Vogar"],["CPLC",49.31472,17.57417,"CZ","CZ","Holesov"],["CBRD",49.6583,13.817,"CZ","CZ","Pizen"],["CSKA",49.5011,16.7885,"CZ","CZ","Ziln"],["RHOF",48.73688,13.92089,"AT","AT","Hochficht"],["LT40",55.60904,22.2395,"LT","LT","Laukuva"],["LT41",54.62622,25.10678,"LT","LT","Vilnius"],["MGUD",35.85278,14.47472,"MT","MT","Malta International Airport"],["BJAB",51.1917,3.0642,"BE","BE","Jabbeke"],["BWID",49.9136,5.5044,"BE","BE","Wideumont"],["BHEL",51.07023,5.40537,"BE","BE","Helchteren"],["CALB",47.28433,8.512,"CH","CH","Langnau am Albis"],["CDOL",46.42511,6.09941,"CH","CH","La Dôle"],["CLEM",46.04076,8.83322,"CH","CH","Lema"],["CPPM",46.37065,7.48655,"CH","CH","Icogne"],["CWEI",46.83497,9.79446,"CH","CH","Weissfluh"],["NDHL",52.95279,4.79061,"CA","NL","Den Helder"],["NHRW",51.8369,5.1381,"CA","NL","Herwijnen"],["SANG",56.36745,12.85169,"SE","SE","Angelholm"],["SATV",58.10591,15.9365,"SE","SE","Atvidaberg"],["SBAA",59.61102,17.58334,"SE","SE","Balsta"],["SHEM",57.30347,18.40008,"SE","SE","Hemse"],["SHUV",61.57711,16.71443,"SE","SE","Hudiksvall"],["SKAA",56.29554,15.61022,"SE","SE","Karlskrona"],["SKRN",67.70876,20.61779,"SE","SE","Kiruna"],["SLEK",60.72304,14.87757,"SE","SE","Leksand"],["SLLA",65.43089,21.86499,"SE","SE","Lulea"],["SOER",63.63947,18.40188,"SE","SE","Ornskoldsvik"],["SOSD",63.29508,14.7591,"SE","SE","Ostersund"],["SVAX",58.25565,12.82602,"SE","SE","Vara"],["SP51",36.6134,-4.6593,"ES","ES","Malaga"],["SP60",28.0188,-15.6145,"ES","ES","Gran Canaria"],["SP62",28.3109,-16.8238,"ES","ES","Buenavista del Norte (Tenerife)"],["SP52",37.6887,-6.3331,"ES","ES","Sevilla"],["SP45",41.4082,1.8849,"ES","ES","Barcelona (Corbera de Llobregat)"],["SP41",41.9956,-4.6028,"ES","ES","Valladolid"],["SP53",36.8324,-2.0821,"ES","ES","Nijar (Almeria)"],["SP47",41.734,-0.5459,"ES","ES","Zaragoza"],["SP50",39.4288,-6.2853,"ES","ES","Caceres"],["SP49",43.4033,-2.8419,"ES","ES","Sollube (Bizkaia)"],["SP42",40.1759,-3.7137,"ES","ES","Madrid"],["KBPP",46.187,-103.428,"US","ND","Bowman ARB"],["KXWA",48.26343,-103.74728,"US","ND","Williston ARB"],["ARMR",34.64615,-86.77153,"US","AL","ARMOR C-band"],["FWLX",35.254,-87.325,"US","TN","WLX X-Band"],["FUSA",38.86949,-75.81616,"US","MD","FUSA/Denton"],["GAWX",33.98047,-84.00345,"US","GA","GAWX/Lawrenceville"],["WILU",40.465,-90.685,"US","IL","Western Illinois University (WIU)"],["MZZU",38.906,-92.269,"US","MO","Columbia/Mizzou"],["CSBI",45.7928,-80.5328,"CA","ON","Britt"],["CSMR",47.2478,-84.5958,"CA","ON","Montreal River Harbour"],["CSCV",53.5606,-114.1447,"CA","AB","Carvel"],["CSKR",43.9639,-79.5714,"CA","ON","King City"],["CSET",43.3703,-81.3803,"CA","ON","Exeter"],["CSHR",47.3266,-53.1264,"CA","NL","Holyrood"],["CSAG",49.0165,-122.487,"CA","BC","Aldergrove"],["CSSF",46.45,-71.9153,"CA","QC","Villeroy"],["CSSR",55.6925,-119.2342,"CA","AB","Spirit River"],["CSVD",48.4803,-67.60109,"CA","QC","Val d'Irène"],["CSBE",50.5711,-105.1828,"CA","SK","Bethune"],["CSSU",50.3125,-110.1956,"CA","AB","Schuler"],["CSDR",49.8583,-92.7972,"CA","ON","Dryden"],["CSFT",45.0408,-76.1136,"CA","ON","Franktown"],["CSFW",50.5492,-101.0856,"CA","MB","Foxwarren"],["CSGO",45.0989,-63.7033,"CA","NS","Gore"],["CSLA",48.5508,-77.8097,"CA","QC","Landrienne"],["CSMB",45.9483,-60.2061,"CA","NS","Marion Bridge"],["CSMM",48.9305,-57.8344,"CA","NL","Marble Mountain"],["CSCM",46.2222,-65.6986,"CA","NB","Chipman"],["CSPG",53.6153,-122.9547,"CA","BC","Prince George"],["CSRA",52.5206,-107.4436,"CA","SK","Radisson"],["CSSM",51.2064,-113.3981,"CA","AB","Strathmore"],["CSSS",50.3695,-119.0633,"CA","BC","Silver Star Mountain"],["CSRF",49.28,-81.7939,"CA","ON","Timmins"],["CSWL",50.1539,-97.7783,"CA","MB","Woodlands"],["CSBV",45.7064,-73.85853,"CA","QC","Montreal-Blainville"],["CSFM",56.37563,-111.21499,"CA","AB","Fort McMurray"],["CSMA",47.97792,-71.43102,"CA","QC","Mont Apica"],["CSHP",49.52703,-123.85371,"CA","BC","Halfmoon Peak"],["CSSN",48.596,-89.10004,"CA","ON","Shuniah"],["CSCL",54.37888,-110.06138,"CA","SK","Cold Lake"],["CPFO",34.77085,32.54716,"CY","CY","Paphos"],["CLCA",34.93654,33.56654,"CY","CY","Larnaca"],["HHAR",46.1775,18.3372,"HU","HU","Harmashegy"],["HNAP",47.9622,21.8867,"HU","HU","Napkor"],["HPOG",46.6604,17.0624,"HU","HU","Poganyvar"],["HSZE",46.6397,20.4325,"HU","HU","Szentes"],["HBUD",47.4294,19.1817,"HU","HU","Budapest"],["ABRD",-37.691,144.946,"AU","AU","Broadmeadows"],["AMEL",-37.8552,144.7552,"AU","AU","Melbourne (Laverton)"],["AWAP",-34.2623,150.8751,"AU","AU","Wollongong (Appin)"],["ANEW",-32.7298,152.0254,"AU","AU","Lemon Tree Passage (Newcastle)"],["AGER",-28.8047,114.6973,"AU","AU","Geraldton"],["AGMK",-25.9572,152.5769,"AU","AU","Gympie (Mt Kanigan)"],["ADAR",-12.4247,130.8919,"AU","AU","Darwin AP"],["AMTG",-37.7475,140.7746,"AU","AU","Mt Gambier"],["ADAM",-20.6544,116.6822,"AU","AU","Dampier"],["ABEL",-20.3718,118.6317,"AU","AU","Port Hedland"],["ABRO",-17.9475,122.2353,"AU","AU","Broome"],["AWEI",-12.6778,141.9208,"AU","AU","Weipa"],["ACSM",-16.8182,145.663,"AU","AU","Cairns (Saddle Mt)"],["AMAC",-21.1172,149.2169,"AU","AU","Mackay"],["AGLA",-23.8553,151.2628,"AU","AU","Gladstone"],["ABOW",-19.8858,148.076,"AU","AU","Bowen"],["AASP",-23.7951,133.889,"AU","AU","Alice Springs"],["APAP",-31.9273,115.9756,"AU","AU","Perth AP"],["AWOO",-31.1569,136.803,"AU","AU","Woomera"],["AGRA",-29.6206,152.9632,"AU","AU","Grafton"],["ALCR",-22.103,113.999,"AU","AU","Learmonth"],["ALBA",-34.9419,117.8018,"AU","AU","Albany"],["AESP",-33.83,121.8909,"AU","AU","Esperance"],["ACED",-32.1296,133.6962,"AU","AU","Ceduna"],["AMIS",-16.664,139.1837,"AU","AU","Mornington Island"],["AHAP",-42.8374,147.5008,"AU","AU","Hobart AP"],["ANWD",-33.097,119.009,"AU","AU","Newdegate"],["AHCK",-18.231,127.663,"AU","AU","Halls Creek"],["ACFC",-35.6614,149.5123,"AU","AU","Captains Flat (Canberra)"],["AWIS",-16.2878,149.9652,"AU","AU","Willis Island"],["ATKA",-14.5131,132.4461,"AU","AU","Tindal (Katherine)"],["AGIL",-25.0341,128.301,"AU","AU","Giles"],["ASMT",-35.3295,138.5025,"AU","AU","Sellick's Hill (Mt Terrible)"],["AKAL",-30.7839,121.4536,"AU","AU","Kalgoorlie"],["AYAR",-36.0294,146.0306,"AU","AU","Yarrawonga"],["AMAR",-27.6064,152.5401,"AU","AU","Marburg"],["AWTA",-41.1806,145.5783,"AU","AU","West Takone"],["AMOR",-29.4903,149.8461,"AU","AU","Moree"],["AWAW",-35.1582,147.4563,"AU","AU","Wagga Wagga"],["ALON",-23.4398,144.2822,"AU","AU","Longreach"],["ASDO",-31.777,117.953,"AU","AU","South Doodlakine"],["ANLK",-29.033,167.933,"AU","AU","Norfolk Island"],["ABER",-12.4559,130.9266,"AU","AU","Berrimah"],["AABP",-34.6169,138.4689,"AU","AU","Adelaide (Buckland Park)"],["ABMS",-27.7177,153.2399,"AU","AU","Brisbane-Mt Stapylton"],["AWAR",-26.4401,147.3491,"AU","AU","Warrego"],["ABGP",-37.8874,147.5756,"AU","AU","Bairnsdale"],["ANAM",-31.0236,150.1917,"AU","AU","Namoi (Blackjack Mountain)"],["APSE",-32.3917,115.867,"AU","AU","Perth Serpentine"],["ASTH",-33.7008,151.2094,"AU","AU","Sydney (Terrey Hills)"],["AEMR",-23.5494,148.2392,"AU","AU","Emerald"],["ATHR",-19.4195,146.5506,"AU","AU","Townsville"],["AGNV",-18.9803,144.9405,"AU","AU","Greenvale"],["AMTH",-20.7114,139.5553,"AU","AU","Mount Isa (Telstra Hill)"],["AHMK",-43.1122,147.8061,"AU","AU","Hobart (Mt Koonya)"],["AARA",-11.6485,133.38,"AU","AU","Arafura (aka Warruwi)"],["AWAT",-30.36,116.29,"AU","AU","Watheroo"],["ABRW",-29.9708,146.814,"AU","AU","Brewarrina"],["AHIL",-33.552,145.5286,"AU","AU","Hilston"],["ARNB",-35.998,142.013,"AU","AU","Rainbow"],["AYEO",-32.7444,148.7081,"AU","AU","Yeoval"],["ACUL",-34.2781,141.5982,"AU","AU","Mildura (Cullulleraine)"],["ATAR",-25.6962,149.8982,"AU","AU","Taroom"],["ARCH",-20.7518,143.1414,"AU","AU","Richmond"],["ATOO",-27.274,151.993,"AU","AU","Toowoomba"],["AGOV",-12.2689,136.8204,"AU","AU","Gove"],["ACAR",-24.8883,113.669,"AU","AU","Gascoyne"],["AKRR",-20.992,116.876,"AU","AU","Karratha"],["AWWB",-15.4519,128.121,"AU","AU","Wyndham"],["AWEP",-12.6778,141.9208,"AU","AU","Weipa"],["DOW7",42.09646,-111.97097,"US","US","Mobile Radar"]];
// DWD open data publishes per-sweep ODIM HDF5 for every German site; FMI publishes ODIM volumes on AWS.
const FMI_CODES={FI44:'fianj',FI45:'fikuo',FI46:'fikor',FI47:'fiuta',FI48:'filuo',FI49:'fivim',FI50:'fikes',FI51:'fipet',FI52:'finur',FI53:'fivih',FI54:'fikan'};
const XBAND=new Set(['FWLX','FUSA','GAWX','WILU','MZZU']);
const DWD_SWEEPS=[['05',0.5],['04',1.5],['03',2.5],['02',3.5],['01',4.5],['00',5.5]];
const SRC_NAME={dwd:'DWD sweep volume',fmi:'FMI volume',opera:'OPERA volume',ideam:'IDEAM volume',geomet:'ECCC composite',rv:'RainViewer composite'};
const EU=new Set(['FR','RO','SK','PL','NO','DK','FI','DE','IT','SI','HR','IE','EE','IS','CZ','AT','LT','MT','BE','CH','NL','SE','ES','CY','HU','PT','LV','GB','BG','RS','GR']);
const RAW_SRC=new Set(['dwd','fmi','opera','ideam']);
function mrrlSites(){
  return MRRL_RAW.map(([id,lat,lon,cc,region,name])=>{
    let src='rv', code=null;
    if(cc==='DE'){ src='dwd'; code=id.replace(/_+$/,'').toLowerCase(); }
    else if(FMI_CODES[id]){ src='fmi'; code=FMI_CODES[id]; }
    else if(cc==='CA') src='geomet';
    else if(EU.has(cc)) src='opera';
    const rangeKm=XBAND.has(id)?75:id==='DOW7'?60:id==='ARMR'?150:cc==='CA'?300:cc==='DE'?180:250;
    return {id,lat,lon,cc,region,name:`${name}${region&&region!==cc?', '+region:''}`,type:'MRRL',src,code,rangeKm};
  });
}
const IDEAM_SITES=[['Barrancabermeja',7.063,-73.851],['Carimagua',4.568,-71.333],['Guaviare',2.565,-72.640],['Munchique',2.527,-76.961],['San_Andres',12.582,-81.703],['Corozal',9.317,-75.286],['Tablazo',4.918,-74.290],['Santa_Elena',6.209,-75.496]];
function ideamSites(){ const fix=store.get('ideamFix',{});
  return IDEAM_SITES.map(([folder,lat,lon])=>{ const f=fix[folder]; return {id:'CO'+folder.slice(0,3).toUpperCase(),folder,lat:f?f[0]:lat,lon:f?f[1]:lon,approx:!f,cc:'CO',region:'CO',name:`${folder.replace(/_/g,' ')} (IDEAM)`,type:'MRRL',src:'ideam',rangeKm:240}; }); }
const isMrrl=s=>s&&s.type==='MRRL';
const mrrlRaw=s=>isMrrl(s)&&RAW_SRC.has(s.src)&&!s.rawBlocked;

/* ================= utils ================= */
const $=s=>document.querySelector(s), $$=s=>[...document.querySelectorAll(s)];
const pad=n=>String(n).padStart(2,'0');
const zTime=d=>`${pad(d.getUTCHours())}:${pad(d.getUTCMinutes())}Z`;
const lTime=d=>(typeof state!=='undefined'&&state.utc)?`${String(d.getUTCHours()).padStart(2,'0')}:${String(d.getUTCMinutes()).padStart(2,'0')} UTC`:d.toLocaleTimeString([], {hour:'numeric',minute:'2-digit',timeZoneName:'short'});
const ymdU=d=>[d.getUTCFullYear(),pad(d.getUTCMonth()+1),pad(d.getUTCDate())];
const store={get(k,d){try{const v=localStorage.getItem('echodesk.'+k);return v===null?d:JSON.parse(v);}catch(e){return d;}},set(k,v){try{localStorage.setItem('echodesk.'+k,JSON.stringify(v));}catch(e){}}};
let toastT=0;
function toast(msg,ms=3500){ const t=$('#toast'); t.textContent=msg; t.hidden=false; clearTimeout(toastT); toastT=setTimeout(()=>t.hidden=true,ms); }
async function s3list(base,prefix){
  const keys=[]; let token=null, n=0;
  do{
    const u=`${base}/?list-type=2&prefix=${encodeURIComponent(prefix)}${token?'&continuation-token='+encodeURIComponent(token):''}`;
    const r=await fetch(u); if(!r.ok) throw new Error('S3 list '+r.status);
    const t=await r.text();
    for(const m of t.matchAll(/<Key>([^<]+)<\/Key>/g)) keys.push(m[1]);
    const nt=t.match(/<NextContinuationToken>([^<]+)<\/NextContinuationToken>/);
    token=/<IsTruncated>true<\/IsTruncated>/.test(t)&&nt?nt[1]:null;
  }while(token && ++n<10);
  return keys;
}
async function s3prefixes(base,prefix){
  const r=await fetch(`${base}/?list-type=2&delimiter=%2F&prefix=${encodeURIComponent(prefix)}`); if(!r.ok) throw new Error('S3 list '+r.status);
  const t=await r.text(); return [...t.matchAll(/<Prefix>([^<]+)<\/Prefix>/g)].map(m=>m[1]).filter(p=>p!==prefix);
}
const listCache=new Map();
async function cachedList(base,prefix,ttl){
  const k=base+'|'+prefix, c=listCache.get(k);
  if(c && Date.now()-c.at<ttl) return c.p;
  const p=s3list(base,prefix).catch(e=>{listCache.delete(k);throw e;});
  listCache.set(k,{at:Date.now(),p}); return p;
}

/* ================= worker pool ================= */
const workerURL=URL.createObjectURL(new Blob([document.getElementById('worker-src').textContent],{type:'text/javascript'}));
class Pool{
  constructor(n,perWorker){ this.q=[]; this.live=new Map(); this.seq=0; this.per=perWorker;
    this.ws=Array.from({length:n},()=>{ const o={w:new Worker(workerURL),busy:0}; o.w.onmessage=e=>this._done(o,e.data); return o; }); }
  run(type,url,opts={},prio=0,tag=''){
    return new Promise((res,rej)=>{ this.q.push({id:++this.seq,type,url,opts,prio,tag,res,rej}); this.q.sort((a,b)=>b.prio-a.prio); this._pump(); });
  }
  bump(pred,prio){ for(const j of this.q) if(pred(j)) j.prio=prio; this.q.sort((a,b)=>b.prio-a.prio); }
  drop(pred){ this.q=this.q.filter(j=>{ if(pred(j)){ j.rej(new Error('cancelled')); return false; } return true; }); }
  _pump(){
    for(const o of this.ws.slice().sort((a,b)=>a.busy-b.busy)){
      while(o.busy<this.per && this.q.length){ const j=this.q.shift(); o.busy++; this.live.set(j.id,{o,j}); o.w.postMessage({id:j.id,type:j.type,url:j.url,opts:j.opts}); }
    }
  }
  _done(o,d){ const L=this.live.get(d.id); if(!L) return; this.live.delete(d.id); o.busy--; d.ok?L.j.res(d.result):L.j.rej(new Error(d.error)); this._pump(); }
}
const pool=new Pool(Math.max(2,Math.min(6,(navigator.hardwareConcurrency||4)-1)),2);

/* ================= palettes ================= */
const HCA_CLASSES=[[10,'BI','Biological','#8c8c8c'],[20,'GC','Ground clutter','#5c5c5c'],[30,'IC','Ice crystals','#f2a8ff'],[40,'DS','Dry snow','#86ccff'],[50,'WS','Wet snow','#3a68ff'],[60,'RA','Rain','#8ff09a'],[70,'HR','Heavy rain','#1aa83a'],[80,'BD','Big drops','#ffe14a'],[90,'GR','Graupel','#b48cff'],[100,'HA','Hail/rain','#ff3434'],[110,'LH','Large hail','#b3001f'],[120,'GH','Giant hail','#ff00c8'],[140,'UK','Unknown','#2f2f2f'],[150,'RF','Range folded','#7a3fa3']];
const PAL={
  REF:{name:'REFLECTIVITY',unit:'dBZ',min:-10,max:80,ticks:[0,10,20,30,40,50,60,70],smooth:true,
    stops:[[-10,40,40,60,0],[4,60,70,95,0],[8,85,95,125,.55],[14,62,112,178,.82],[19,50,160,210,.9],[20,28,196,96,1],[27,14,148,62,1],[34,10,98,40,1],[35,252,226,48,1],[40,246,168,28,1],[45,240,108,26,1],[50,222,34,34,1],[55,160,16,42,1],[60,236,64,156,1],[65,176,74,232,1],[70,250,250,255,1],[75,150,236,255,1],[95,60,190,255,1]],
    fmt:v=>v.toFixed(1)+' dBZ'},
  VEL:{name:'VELOCITY',unit:'kt',min:-120,max:120,ticks:[-100,-50,0,50,100],smooth:true,
    stops:[[-140,255,250,205,1],[-100,40,236,250,1],[-70,0,205,120,1],[-48,0,150,62,1],[-24,10,96,42,1],[-6,70,102,82,1],[0,92,92,92,1],[6,104,72,72,1],[24,112,16,22,1],[48,178,16,32,1],[70,242,32,52,1],[100,255,140,182,1],[140,255,232,240,1]],
    fmt:v=>(v>0?'+':'')+v.toFixed(0)+' kt'},
  SW:{name:'SPECTRUM WIDTH',unit:'kt',min:0,max:40,ticks:[0,10,20,30,40],smooth:true,
    stops:[[0,90,90,100,0],[2,90,90,100,.5],[6,120,130,150,.8],[10,30,170,90,1],[15,240,220,50,1],[20,245,140,30,1],[25,225,40,40,1],[32,200,60,200,1],[40,255,255,255,1]],fmt:v=>v.toFixed(0)+' kt'},
  ZDR:{name:'DIFF. REFLECTIVITY',unit:'dB',min:-3,max:7,ticks:[-2,0,2,4,6],smooth:true,
    stops:[[-4,40,30,90,1],[-2,110,70,170,1],[-.5,150,150,160,1],[0,190,190,190,1],[.5,110,150,220,1],[1,40,190,230,1],[1.5,40,200,120,1],[2,90,220,70,1],[3,245,225,50,1],[4,245,150,30,1],[5,230,50,40,1],[6,240,90,170,1],[8,255,255,255,1]],fmt:v=>v.toFixed(2)+' dB'},
  CC:{name:'CORRELATION COEFF.',unit:'ρhv',min:.2,max:1.05,ticks:[.3,.5,.7,.8,.9,.95,1],smooth:true,
    stops:[[.2,20,20,80,1],[.45,40,60,160,1],[.65,30,150,210,1],[.75,40,200,110,1],[.85,230,225,60,1],[.9,240,160,40,1],[.93,230,80,30,1],[.96,180,15,45,1],[.98,140,10,110,1],[1,240,210,245,1],[1.05,255,255,255,1]],fmt:v=>v.toFixed(3)},
  KDP:{name:'SPECIFIC DIFF. PHASE',unit:'°/km',min:-2,max:8,ticks:[-1,0,1,2,4,6,8],smooth:true,
    stops:[[-2,80,80,90,.8],[-.5,120,120,130,.6],[0,140,140,150,.35],[.3,60,170,220,.9],[1,40,200,110,1],[2,240,225,50,1],[3,245,140,30,1],[4,225,40,40,1],[6,220,70,200,1],[8,255,255,255,1]],fmt:v=>v.toFixed(2)+' °/km'},
  VIL:{name:'VERT. INTEGRATED LIQUID',unit:'kg/m²',min:0,max:80,ticks:[0,10,20,30,40,50,60,70],smooth:true,
    stops:[[0,60,70,110,0],[.5,60,70,110,.5],[2,70,100,170,.85],[5,40,160,230,1],[10,30,190,90,1],[20,245,225,50,1],[30,245,140,30,1],[40,225,40,40,1],[50,170,15,90,1],[60,230,100,230,1],[75,255,255,255,1]],fmt:v=>v.toFixed(1)+' kg/m²'},
  ET:{name:'ECHO TOPS',unit:'kft',min:0,max:70,ticks:[0,10,20,30,40,50,60,70],smooth:true,
    stops:[[0,60,70,110,0],[2,70,80,120,.6],[10,60,110,200,.9],[20,30,180,220,1],[30,30,190,90,1],[40,245,225,50,1],[45,245,140,30,1],[50,225,40,40,1],[60,200,60,200,1],[70,255,255,255,1]],fmt:v=>v.toFixed(0)+' kft'},
  PCP:{name:'PRECIPITATION',unit:'in',min:0,max:6,ticks:[0,.5,1,2,3,4,6],smooth:true,
    stops:[[0,100,120,150,0],[.01,100,120,150,0],[.03,110,140,180,.7],[.1,60,150,220,.9],[.25,30,190,200,1],[.5,40,190,90,1],[1,240,225,50,1],[1.5,245,150,30,1],[2,225,40,40,1],[3,170,20,120,1],[4,230,110,230,1],[6,255,255,255,1],[10,170,240,255,1]],fmt:v=>v.toFixed(2)+' in'},
  HCA:{name:'HYDROMETEOR CLASS',unit:'',cat:true,smooth:false}
};
const hex2rgb=h=>[parseInt(h.slice(1,3),16),parseInt(h.slice(3,5),16),parseInt(h.slice(5,7),16)];
function hcaClass(v){ let best=null,bd=6; for(const c of HCA_CLASSES){ const d=Math.abs(c[0]-v); if(d<bd){bd=d;best=c;} } return best; }
function palColor(p,v){
  const s=p.stops; if(v<=s[0][0]) return s[0].slice(1); if(v>=s[s.length-1][0]) return s[s.length-1].slice(1);
  for(let i=1;i<s.length;i++) if(v<s[i][0]){ const a=s[i-1],b=s[i],t=(v-a[0])/(b[0]-a[0]); return [1,2,3,4].map(k=>a[k]+(b[k]-a[k])*t); }
}
// code -> physical value table for a decoded product
const decCache=new Map();
function decodeTable(dec){
  const k=JSON.stringify(dec); let r=decCache.get(k); if(r) return r;
  const v=new Float32Array(256).fill(NaN), rf=new Uint8Array(256);
  if(dec.t==='lin'){ for(let c=2;c<256;c++) v[c]=(dec.min+(c-2)*dec.inc)*dec.mul; if(dec.rf) rf[1]=1; }
  else if(dec.t==='so'){ if(dec.scale) for(let c=dec.lead||2;c<256;c++) v[c]=(c-dec.offset)/dec.scale*dec.mul; if(dec.rf) rf[1]=1; }
  else if(dec.t==='tab'){ dec.vals.forEach((x,i)=>{ if(x==='RF') rf[i]=1; else if(x!==null&&i>0) v[i]=x; }); }
  else if(dec.t==='cat'){ for(let c=1;c<256;c++){ if(Math.abs(c-150)<5) rf[c]=1; else v[c]=c; } }
  else if(dec.t==='vil'){ for(let c=2;c<256;c++) v[c]=c<dec.start?(c-dec.lo)/dec.ls:Math.exp((c-dec.go)/dec.gs); }
  else if(dec.t==='eet'){ for(let c=2;c<256;c++) v[c]=(c&dec.mask)/dec.scale-dec.offset; }
  r={v,rf}; decCache.set(k,r); return r;
}
let lutVer=0;
const PALETTES={
  REF:{'Echo Desk':null,
    'NWS classic':(()=>{ const c=[[5,'04e9e7'],[10,'019ff4'],[15,'0300f4'],[20,'02fd02'],[25,'01c501'],[30,'008e00'],[35,'fdf802'],[40,'e5bc00'],[45,'fd9500'],[50,'fd0000'],[55,'d40000'],[60,'bc0000'],[65,'f800fd'],[70,'9854c6'],[75,'fdfdfd']]; const o=[[-10,0,0,0,0],[4.99,0,0,0,0]]; c.forEach(([v,h])=>{ const r=parseInt(h.slice(0,2),16),g=parseInt(h.slice(2,4),16),b=parseInt(h.slice(4,6),16); o.push([v,r,g,b,1],[v+4.99,r,g,b,1]); }); o.push([95,253,253,253,1]); return o; })(),
    'Grayscale':[[-10,40,40,40,0],[5,70,70,70,.5],[20,120,120,120,.9],[40,190,190,190,1],[60,245,245,245,1],[75,255,255,255,1],[95,255,255,255,1]],
    'Dark ramp':[[-10,20,30,60,0],[10,30,60,110,.7],[20,20,110,170,.9],[30,20,170,130,1],[40,230,210,60,1],[50,240,110,40,1],[60,210,30,60,1],[70,250,240,255,1],[95,255,255,255,1]]},
  VEL:{'Echo Desk':null,
    'Classic red/green':[[-140,0,255,0,1],[-64,0,255,0,1],[-48,0,210,0,1],[-32,0,160,0,1],[-16,0,110,0,1],[-2,60,100,60,1],[0,120,120,120,1],[2,100,60,60,1],[16,110,0,0,1],[32,160,0,0,1],[48,210,0,0,1],[64,255,0,0,1],[140,255,120,120,1]],
    'High contrast':[[-140,230,255,255,1],[-80,0,255,255,1],[-50,0,120,255,1],[-25,0,50,140,1],[-5,40,40,70,1],[0,60,60,60,1],[5,70,40,40,1],[25,150,40,0,1],[50,255,120,0,1],[80,255,255,0,1],[140,255,255,230,1]]}
};
const PAL_DEFAULT={REF:null,VEL:null};
function applyPalette(kind,name){ if(!PAL_DEFAULT[kind]) PAL_DEFAULT[kind]=PAL[kind].stops; const st=PALETTES[kind][name]; PAL[kind].stops=st||PAL_DEFAULT[kind]; lutVer++; }
function buildLUT(dec,palKey){
  const p=PAL[palKey], T=decodeTable(dec), lut=new Uint8Array(256*4);
  for(let c=0;c<256;c++){
    let col=null;
    if(T.rf[c]) col=state.hideRF?null:[128,64,176,.92];
    else if(palKey==='REF'&&!isNaN(T.v[c])&&T.v[c]<state.minRef) col=null;
    else if(!isNaN(T.v[c])){ if(p.cat){ const h=hcaClass(T.v[c]); if(h) col=[...hex2rgb(h[3]),h[1]==='UK'?.5:1]; } else col=palColor(p,T.v[c]); }
    if(col){ lut[c*4]=col[0]; lut[c*4+1]=col[1]; lut[c*4+2]=col[2]; lut[c*4+3]=Math.round(col[3]*255); }
  }
  return lut;
}
function fmtValue(d,code){
  const p=PAL[d.pal], T=decodeTable(d.dec);
  if(T.rf[code]) return 'RF'; const v=T.v[code]; if(isNaN(v)) return '—';
  if(p.cat){ const h=hcaClass(v); return h?h[2]:'—'; }
  return p.fmt(v);
}
function drawLegend(palKey){
  const p=PAL[palKey], c=$('#legendBar'), x=c.getContext('2d'), w=c.width;
  x.clearRect(0,0,w,c.height);
  if(p.cat){
    const cls=HCA_CLASSES.filter(k=>k[1]!=='RF'), sw=w/cls.length;
    cls.forEach((k,i)=>{ x.fillStyle=k[3]; x.fillRect(i*sw,0,sw-1,c.height); });
    $('#legendTicks').innerHTML=cls.map((k,i)=>`<span style="left:${(i+.5)/cls.length*100}%">${k[1]}</span>`).join('');
  } else {
    for(let i=0;i<w;i++){ const v=p.min+(p.max-p.min)*i/(w-1); const col=palColor(p,v); x.fillStyle=`rgba(${col[0]|0},${col[1]|0},${col[2]|0},${Math.max(col[3],.25)})`; x.fillRect(i,0,1,c.height); }
    $('#legendTicks').innerHTML=p.ticks.map(t=>`<span style="left:${(t-p.min)/(p.max-p.min)*100}%">${t}</span>`).join('');
  }
  $('#legendName').textContent=p.name; $('#legendU').textContent=p.unit;
}

/* ================= map ================= */
const map=L.map('map',{zoomControl:true,attributionControl:true,preferCanvas:false,worldCopyJump:true,zoomSnap:0.25,wheelPxPerZoomLevel:90,fadeAnimation:false}).setView([35.33,-97.28],7);
map.zoomControl.setPosition('bottomright');
[['radar',350],['alerts',420],['labels',450],['cells',460],['sites',470]].forEach(([n,z])=>{ const p=map.createPane(n); p.style.zIndex=z; if(n==='labels'||n==='radar'||n==='cells') p.style.pointerEvents='none'; });
const ESRI='https://server.arcgisonline.com/ArcGIS/rest/services/';
const esriAttr='Tiles &copy; Esri';
const BASEMAPS={
  dark:{base:ESRI+'Canvas/World_Dark_Gray_Base/MapServer/tile/{z}/{y}/{x}',ref:ESRI+'Canvas/World_Dark_Gray_Reference/MapServer/tile/{z}/{y}/{x}',max:16,attr:esriAttr},
  light:{base:ESRI+'Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}',ref:ESRI+'Canvas/World_Light_Gray_Reference/MapServer/tile/{z}/{y}/{x}',max:16,attr:esriAttr},
  sat:{base:ESRI+'World_Imagery/MapServer/tile/{z}/{y}/{x}',ref:ESRI+'Reference/World_Boundaries_and_Places/MapServer/tile/{z}/{y}/{x}',max:18,attr:esriAttr+', Maxar, Earthstar'},
  osm:{base:'https://tile.openstreetmap.org/{z}/{x}/{y}.png',ref:null,max:19,attr:'&copy; OpenStreetMap contributors'}
};
let baseLayer=null, refLayer=null;
function setBasemap(k){
  const b=BASEMAPS[k]; if(!b) return;
  if(baseLayer) map.removeLayer(baseLayer); if(refLayer) map.removeLayer(refLayer);
  baseLayer=L.tileLayer(b.base,{maxNativeZoom:b.max,maxZoom:19,attribution:b.attr+' · Data: NOAA/NWS via AWS Open Data',keepBuffer:4,updateWhenZooming:false,crossOrigin:'anonymous'}).addTo(map);
  refLayer=b.ref?L.tileLayer(b.ref,{pane:'labels',maxNativeZoom:b.max,maxZoom:19,keepBuffer:4,updateWhenZooming:false,crossOrigin:'anonymous'}):null;
  if(refLayer && $('#lyLabels').checked) refLayer.addTo(map);
  if(typeof split!=='undefined') split.setBasemap(k);
  $$('.basemaps button').forEach(x=>x.setAttribute('aria-pressed',x.dataset.bm===k));
  store.set('bm',k);
}

/* ===== canvas layer base (zoom-animation aware) ===== */
const ScreenCanvas=L.Layer.extend({
  initialize(pane,cls){ this._paneName=pane; this._cls=cls; },
  onAdd(m){
    this._c=L.DomUtil.create('canvas',this._cls+' leaflet-zoom-animated');
    m.getPane(this._paneName).appendChild(this._c);
    this.init&&this.init();
    m.on('move zoomend resize viewreset',this._reset,this); m.on('zoomanim',this._anim,this);
    this._reset();
  },
  onRemove(m){ m.off('move zoomend resize viewreset',this._reset,this); m.off('zoomanim',this._anim,this); L.DomUtil.remove(this._c); },
  _anim(e){ const m=this._map, sc=m.getZoomScale(e.zoom,this._zoom), half=m.getSize().multiplyBy(.5), cp=m.project(this._center,e.zoom), off=half.multiplyBy(-sc).add(cp).subtract(m._getNewPixelOrigin(e.center,e.zoom)); L.DomUtil.setTransform(this._c,off,sc); },
  _reset(){
    const m=this._map; if(!m) return; const sz=m.getSize(), dpr=Math.min(window.devicePixelRatio||1,2);
    this._center=m.getCenter(); this._zoom=m.getZoom();
    L.DomUtil.setPosition(this._c,m.containerPointToLayerPoint([0,0]));
    const w=Math.round(sz.x*dpr), h=Math.round(sz.y*dpr);
    if(this._c.width!==w||this._c.height!==h){ this._c.width=w; this._c.height=h; this._c.style.width=sz.x+'px'; this._c.style.height=sz.y+'px'; }
    this.dpr=dpr; this.redraw();
  },
  redraw(){ if(this._raf) return; this._raf=requestAnimationFrame(()=>{ this._raf=0; this.draw&&this.draw(); }); },
  drawNow(){ if(this._raf){cancelAnimationFrame(this._raf);this._raf=0;} this.draw&&this.draw(); }
});

/* ===== WebGL radar renderer ===== */
const VS=`attribute vec2 p;void main(){gl_Position=vec4(p,0.,1.);}`;
const FS=`precision highp float;
uniform sampler2D uData,uLut;uniform vec2 uOrigin,uRadM,uSizePx;uniform float uWorld,uDpr,uLat0,uFirst,uGate,uNG,uSmooth,uOpacity;
const float PI=3.141592653589793;
float cd(float g,float a){a=mod(a,720.);g=clamp(g,0.,uNG-1.);return floor(texture2D(uData,vec2((g+.5)/uNG,(a+.5)/720.)).r*255.+.5);}
void main(){
 vec2 css=vec2(gl_FragCoord.x,uSizePx.y-gl_FragCoord.y)/uDpr;
 vec2 d=uOrigin+css;
 float my=uRadM.y+d.y/uWorld;
 float lat=2.*atan(exp(PI*(1.-2.*my)))-PI*.5;
 float dlon=d.x/uWorld*2.*PI;
 float x=6371.*dlon*cos(.5*(lat+uLat0)), y=6371.*(lat-uLat0);
 float r=sqrt(x*x+y*y);
 float az=degrees(atan(x,y)); if(az<0.)az+=360.;
 float gf=(r-uFirst)/uGate; if(gf<0.||gf>=uNG)discard;
 float af=az/.5; float c;
 if(uSmooth>.5){
  float g0=floor(gf-.5),a0=floor(af-.5),tg=gf-.5-g0,ta=af-.5-a0;
  float c00=cd(g0,a0),c10=cd(g0+1.,a0),c01=cd(g0,a0+1.),c11=cd(g0+1.,a0+1.);
  if(min(min(c00,c10),min(c01,c11))>=2.) c=mix(mix(c00,c10,tg),mix(c01,c11,tg),ta);
  else c=cd(floor(gf),floor(af));
 } else c=cd(floor(gf),floor(af));
 if(c<.5)discard;
 vec4 col=texture2D(uLut,vec2((c+.5)/256.,.5));
 if(col.a<.02)discard;
 gl_FragColor=vec4(col.rgb,col.a*uOpacity);
}`;
const RadarGL=ScreenCanvas.extend({
  init(){
    const gl=this.gl=this._c.getContext('webgl',{alpha:true,premultipliedAlpha:false,antialias:false,preserveDrawingBuffer:false});
    if(!gl){ toast('WebGL is unavailable in this browser.'); return; }
    const sh=(t,s)=>{ const o=gl.createShader(t); gl.shaderSource(o,s); gl.compileShader(o); if(!gl.getShaderParameter(o,gl.COMPILE_STATUS)) console.error(gl.getShaderInfoLog(o)); return o; };
    const pr=this.pr=gl.createProgram(); gl.attachShader(pr,sh(gl.VERTEX_SHADER,VS)); gl.attachShader(pr,sh(gl.FRAGMENT_SHADER,FS)); gl.linkProgram(pr); gl.useProgram(pr);
    const b=gl.createBuffer(); gl.bindBuffer(gl.ARRAY_BUFFER,b); gl.bufferData(gl.ARRAY_BUFFER,new Float32Array([-1,-1,1,-1,-1,1,1,1]),gl.STATIC_DRAW);
    const loc=gl.getAttribLocation(pr,'p'); gl.enableVertexAttribArray(loc); gl.vertexAttribPointer(loc,2,gl.FLOAT,false,0,0);
    this.u={}; for(const n of ['uData','uLut','uOrigin','uRadM','uSizePx','uWorld','uDpr','uLat0','uFirst','uGate','uNG','uSmooth','uOpacity']) this.u[n]=gl.getUniformLocation(pr,n);
    gl.uniform1i(this.u.uData,0); gl.uniform1i(this.u.uLut,1);
    gl.pixelStorei(gl.UNPACK_ALIGNMENT,1);
    this.luts=new Map(); this.texs=new Map(); this.lru=[];
    gl.enable(gl.BLEND); gl.blendFunc(gl.SRC_ALPHA,gl.ONE_MINUS_SRC_ALPHA);
    this._c.addEventListener('webglcontextlost',e=>{e.preventDefault();});
  },
  _params(f){ const gl=this.gl; gl.texParameteri(gl.TEXTURE_2D,gl.TEXTURE_MIN_FILTER,f); gl.texParameteri(gl.TEXTURE_2D,gl.TEXTURE_MAG_FILTER,f); gl.texParameteri(gl.TEXTURE_2D,gl.TEXTURE_WRAP_S,gl.CLAMP_TO_EDGE); gl.texParameteri(gl.TEXTURE_2D,gl.TEXTURE_WRAP_T,gl.CLAMP_TO_EDGE); },
  lutFor(d){
    const k=d.pal+'|'+JSON.stringify(d.dec)+'|'+lutVer; let t=this.luts.get(k); if(t) return t; const gl=this.gl;
    if(this.luts.size>40){ for(const v of this.luts.values()) gl.deleteTexture(v); this.luts.clear(); }
    t=gl.createTexture(); gl.bindTexture(gl.TEXTURE_2D,t); gl.texImage2D(gl.TEXTURE_2D,0,gl.RGBA,256,1,0,gl.RGBA,gl.UNSIGNED_BYTE,buildLUT(d.dec,d.pal)); this._params(PAL[d.pal].cat?gl.NEAREST:gl.LINEAR);
    this.luts.set(k,t); return t;
  },
  texFor(d){
    let t=this.texs.get(d); if(t) return t; const gl=this.gl; t=gl.createTexture(); gl.bindTexture(gl.TEXTURE_2D,t);
    gl.texImage2D(gl.TEXTURE_2D,0,gl.LUMINANCE,d.nGates,720,0,gl.LUMINANCE,gl.UNSIGNED_BYTE,d.grid); this._params(gl.NEAREST);
    this.texs.set(d,t); this.lru.push(d); while(this.lru.length>Math.max(64,state.nFrames+12)){ const o=this.lru.shift(), ot=this.texs.get(o); if(ot){ gl.deleteTexture(ot); this.texs.delete(o); } }
    return t;
  },
  setData(d){ this.data=d; this.drawNow(); },
  draw(){
    const gl=this.gl; if(!gl) return; const m=this._map;
    gl.viewport(0,0,this._c.width,this._c.height); gl.clearColor(0,0,0,0); gl.clear(gl.COLOR_BUFFER_BIT);
    const d=this.data; if(!d||!d.grid||d.lat==null) return;
    const z=m.getZoom(), world=256*Math.pow(2,z), pb=m.getPixelBounds(), rw=m.project([d.lat,d.lon],z);
    gl.useProgram(this.pr);
    gl.activeTexture(gl.TEXTURE1); gl.bindTexture(gl.TEXTURE_2D,this.lutFor(d));
    gl.activeTexture(gl.TEXTURE0); gl.bindTexture(gl.TEXTURE_2D,this.texFor(d));
    const U=this.u, smooth=state.smooth&&PAL[d.pal].smooth&&d.dec.t!=='tab'&&d.dec.t!=='cat';
    gl.uniform2f(U.uOrigin,pb.min.x-rw.x,pb.min.y-rw.y); gl.uniform2f(U.uRadM,rw.x/world,rw.y/world);
    gl.uniform2f(U.uSizePx,this._c.width,this._c.height); gl.uniform1f(U.uWorld,world); gl.uniform1f(U.uDpr,this.dpr);
    gl.uniform1f(U.uLat0,d.lat*Math.PI/180); gl.uniform1f(U.uFirst,d.firstKm); gl.uniform1f(U.uGate,d.gateKm); gl.uniform1f(U.uNG,d.nGates);
    gl.uniform1f(U.uSmooth,smooth?1:0); gl.uniform1f(U.uOpacity,state.opacity);
    gl.drawArrays(gl.TRIANGLE_STRIP,0,4);
  }
});
const texLRU=[];

/* ===== 2D overlay: GLM, storm tracks, TVS, range ring ===== */
/* lightning bolt sprites, pre-rendered per age class */
const BOLT=[[.60,0],[.16,.58],[.45,.58],[.32,1],[.86,.36],[.55,.36],[.74,0]];
const boltSprites=(()=>{ const dpr=Math.min(window.devicePixelRatio||1,2);
  return [['#ffffff',20,'rgba(255,240,150,.95)'],['#ffd23f',16,null],['#ff8a1f',13,null]].map(([fill,sz,glow])=>{
    const c=document.createElement('canvas'), pad=4; c.width=c.height=Math.ceil((sz+pad*2)*dpr); const x=c.getContext('2d'); x.scale(dpr,dpr);
    x.translate(pad,pad); x.beginPath(); BOLT.forEach(([u,v],i)=>{ const px=u*sz,py=v*sz; i?x.lineTo(px,py):x.moveTo(px,py); }); x.closePath();
    if(glow){ x.shadowColor=glow; x.shadowBlur=6; }
    x.fillStyle=fill; x.fill(); x.shadowBlur=0; x.lineJoin='round'; x.lineWidth=1.2; x.strokeStyle='rgba(0,0,0,.9)'; x.stroke();
    return {c,w:sz+pad*2};
  }); })();
const tornadoSprites=(()=>{ const dpr=Math.min(window.devicePixelRatio||1,2);
  return [['#ff2a3c','#ffffff'],['#ffd600','#1a1208']].map(([fill,band])=>{
    const sz=26,pad=3,c=document.createElement('canvas'); c.width=c.height=Math.ceil((sz+pad*2)*dpr); const x=c.getContext('2d'); x.scale(dpr,dpr); x.translate(pad,pad);
    // funnel: stacked bands narrowing and drifting toward the ground
    const bands=[[13,3,12],[12.4,7.5,9.5],[11.2,12,7],[10.4,16.5,4.8],[11.4,20.5,2.8]];
    x.lineJoin='round'; x.lineCap='round';
    const draw=(lw,col)=>{ x.strokeStyle=col; x.lineWidth=lw; for(const [cx,cy,hw] of bands){ x.beginPath(); x.moveTo(cx-hw,cy); x.lineTo(cx+hw,cy); x.stroke(); } x.beginPath(); x.moveTo(11.4,20.5); x.quadraticCurveTo(12.5,23.5,13.5,24.5); x.stroke(); };
    draw(6.4,'#000'); draw(3.6,fill);
    x.strokeStyle=band; x.globalAlpha=.55; x.lineWidth=1; x.beginPath(); x.moveTo(13-8,2.2); x.lineTo(13+8,2.2); x.stroke(); x.globalAlpha=1;
    return {c,w:sz+pad*2};
  }); })();
function pip(pt,ring){ let ins=false; for(let i=0,j=ring.length-1;i<ring.length;j=i++){ const xi=ring[i][0],yi=ring[i][1],xj=ring[j][0],yj=ring[j][1]; if(((yi>pt[1])!==(yj>pt[1]))&&(pt[0]<(xj-xi)*(pt[1]-yi)/(yj-yi)+xi)) ins=!ins; } return ins; }
function inGeom(g,lat,lon){ if(!g) return false; const p=[lon,lat]; const polys=g.type==='Polygon'?[g.coordinates]:g.type==='MultiPolygon'?g.coordinates:[]; return polys.some(pl=>pip(p,pl[0])); }
function parseMotion(P){
  const s=P&&P.parameters&&P.parameters.eventMotionDescription&&P.parameters.eventMotionDescription[0]; if(!s) return null;
  const m=String(s).match(/^(.+?)\.\.\.[^.]*\.\.\.(\d{1,3})DEG\.\.\.(\d{1,3})KT\.\.\.(.+)$/); if(!m) return null;
  const pts=m[4].trim().split(/\s+/).map(p=>p.split(',').map(Number)).filter(p=>p.length===2&&!isNaN(p[0])&&!isNaN(p[1]));
  return {t:Date.parse(m[1]),from:+m[2],kt:+m[3],pts};
}
const Overlay=ScreenCanvas.extend({
  init(){ this.ctx=this._c.getContext('2d'); },
  draw(){
    const x=this.ctx, m=this._map, dpr=this.dpr; if(!x) return;
    const W=this._c.width/dpr, H=this._c.height/dpr;
    x.setTransform(1,0,0,1,0,0); x.clearRect(0,0,this._c.width,this._c.height); x.setTransform(dpr,0,0,dpr,0,0);
    const P=ll=>m.latLngToContainerPoint(ll);
    const pxPerKm=(lat,lon)=>{ const a=P([lat,lon]), b=P(destLL(lat,lon,10,90)); return Math.abs(b.x-a.x)/10; };
    const fr=curFrame(), d=fr&&fr.data;
    const label=(txt,px,py,bg,fg,border)=>{ x.font='600 10.5px "IBM Plex Mono",monospace'; const tw=x.measureText(txt).width; const w=tw+10,h=16;
      x.fillStyle=bg; x.beginPath(); x.roundRect?x.roundRect(px,py-h/2,w,h,3):x.rect(px,py-h/2,w,h); x.fill();
      if(border){ x.fillStyle=border; x.fillRect(px,py-h/2,2.5,h); }
      x.fillStyle=fg; x.textBaseline='middle'; x.fillText(txt,px+5+(border?1.5:0),py+.5); return w; };
    this.hit=[];
    // range ring
    const ring=d&&d.lat!=null?[d.lat,d.lon,d.firstKm+d.gateKm*d.nGates]:(state.site&&isMrrl(state.site)?[state.site.lat,state.site.lon,state.site.rangeKm]:null);
    if(ring){
      const c=P([ring[0],ring[1]]); const edge=P(destLL(ring[0],ring[1],ring[2],90));
      x.strokeStyle='rgba(255,236,210,.14)'; x.lineWidth=1; x.setLineDash([3,5]); x.beginPath(); x.arc(c.x,c.y,Math.abs(edge.x-c.x),0,Math.PI*2); x.stroke(); x.setLineDash([]);
    }
    if(typeof tools!=='undefined'&&tools.grid) tools.drawGrid(x,m,W,H);
    // range rings + spokes
    if(state.rings&&ring){
      const [k,u]=UNITS[state.units], step=u==='km'?50:u==='nm'?25:25, maxU=ring[2]*k, c0=P([ring[0],ring[1]]);
      x.save(); x.font='500 10px "IBM Plex Mono",monospace'; x.textBaseline='middle'; x.textAlign='center';
      for(let v=step;v<=maxU+1e-6;v+=step){
        const km=v/k, e=P(destLL(ring[0],ring[1],km,90)), r=Math.abs(e.x-c0.x); if(r<18) continue;
        x.strokeStyle='rgba(255,236,210,.28)'; x.lineWidth=1; x.beginPath(); x.arc(c0.x,c0.y,r,0,Math.PI*2); x.stroke();
        const lp=P(destLL(ring[0],ring[1],km,0)); const t=`${v} ${u}`; const tw=x.measureText(t).width+6;
        x.fillStyle='rgba(8,10,14,.75)'; x.fillRect(lp.x-tw/2,lp.y-7,tw,14); x.fillStyle='rgba(255,236,210,.8)'; x.fillText(t,lp.x,lp.y);
      }
      x.strokeStyle='rgba(255,236,210,.14)'; x.setLineDash([2,6]);
      for(let a=0;a<360;a+=30){ const e=P(destLL(ring[0],ring[1],ring[2],a)); x.beginPath(); x.moveTo(c0.x,c0.y); x.lineTo(e.x,e.y); x.stroke(); }
      x.setLineDash([]); x.restore();
    }
    // GLM lightning icons
    if(state.glm && fr){
      const fl=glmFor(fr.t), t1=+fr.t, win=state.glmWin*60000, z=m.getZoom();
      // optional radar-range clip: the current product's range ring
      const site=state.site||{lat:0,lon:0}, cLat=d&&d.lat!=null?d.lat:site.lat, cLon=d&&d.lat!=null?d.lon:site.lon;
      const rKm=d&&d.lat!=null?d.firstKm+d.gateKm*d.nGates:(site.rangeKm||460), kx=111.2*Math.cos(cLat*Math.PI/180), r2=rKm*rKm;
      const inRange=(la,lo)=>{ const dy=(la-cLat)*111.2, dx=(lo-cLon)*kx; return dx*dx+dy*dy<=r2; };
      let n=0; for(const g of fl){ if(!state.glmRange){ n+=g.lat.length; continue; } for(let i=0;i<g.lat.length;i++) if(inRange(g.lat[i],g.lon[i])) n++; }
      const scale=n>4000||z<6?.62:z<7.5?.8:1;
      for(const g of fl.sort((a,b)=>a.t-b.t)){
        const age=Math.max(0,(t1-g.t)/win);
        const sp=boltSprites[age<.2?0:age<.6?1:2], s=sp.w*scale;
        x.globalAlpha=state.glmFade?Math.max(.18,1-age*.85):1;
        for(let i=0;i<g.lat.length;i++){
          if(state.glmRange&&!inRange(g.lat[i],g.lon[i])) continue;
          const p=P([g.lat[i],g.lon[i]]); if(p.x<-20||p.y<-20||p.x>W+20||p.y>H+20) continue; x.drawImage(sp.c,p.x-s/2,p.y-s*.55,s,s);
        }
      }
      x.globalAlpha=1;
      updateGlmStat(n,fr.t);
    } else updateGlmStat(null);
    // NWS warning motion arrows (extrapolated to the frame time)
    if(state.alerts && fr){
      const t=+fr.t;
      for(const s of curAlerts){
        const mo=s.motion; if(!mo||!mo.pts.length||mo.kt<1) continue;
        const col=alertColor(s), toward=(mo.from+180)%360;
        const dtMin=Math.max(-30,Math.min(60,(t-mo.t)/60000));
        const pts=mo.pts.length>2?[mo.pts[0],mo.pts[Math.floor(mo.pts.length/2)],mo.pts[mo.pts.length-1]]:mo.pts;
        for(const q of pts){
          const o=destLL(q[0],q[1],mo.kt*1.852*dtMin/60,toward);
          const a=P(o), lenPx=Math.max(30,Math.min(110,mo.kt*1.852*.5*pxPerKm(o[0],o[1])));
          const r=toward*Math.PI/180, b={x:a.x+Math.sin(r)*lenPx,y:a.y-Math.cos(r)*lenPx};
          const hx=Math.sin(r), hy=-Math.cos(r), nx=-hy, ny=hx, hl=11, hw=7;
          const tip={x:b.x+hx*4,y:b.y+hy*4};
          const path=()=>{ x.beginPath(); x.moveTo(a.x,a.y); x.lineTo(b.x-hx*hl*.6,b.y-hy*hl*.6); };
          const head=()=>{ x.beginPath(); x.moveTo(tip.x,tip.y); x.lineTo(tip.x-hx*hl+nx*hw,tip.y-hy*hl+ny*hw); x.lineTo(tip.x-hx*hl-nx*hw,tip.y-hy*hl-ny*hw); x.closePath(); };
          x.lineCap='round'; x.strokeStyle='rgba(0,0,0,.85)'; x.lineWidth=6; path(); x.stroke(); head(); x.lineWidth=3; x.lineJoin='round'; x.stroke();
          x.strokeStyle=col; x.lineWidth=3; path(); x.stroke(); head(); x.fillStyle=col; x.fill();
          x.fillStyle='#fff'; x.strokeStyle='#000'; x.lineWidth=1.5; x.beginPath(); x.arc(a.x,a.y,3,0,7); x.fill(); x.stroke();
          if(q===pts[0]) label(`${mo.kt} KT`,b.x+(hx>=0?8:-62),b.y+(hy>0?10:-10),'rgba(8,10,14,.82)','#fff',col);
        }
      }
    }
    // storm tracks: SCIT forecast line with a perpendicular tick at each 15-min forecast position
    const cells=fr&&cellsFor(fr);
    if(cells && state.tracks){
      const segs=[], ticks=[], dots=[];
      for(const s of cells.storms){
        const c=P(s.pos); dots.push(c);
        this.hit.push({x:c.x,y:c.y,kind:'cell',s,base:cells.t,src:cells.src});
        if(!s.fcst.length) continue;
        const pts=[s.pos,...s.fcst];
        segs.push(pts.map(P));
        for(let i=1;i<pts.length;i++){
          const brg=bearingDeg(pts[i-1],pts[i]);
          ticks.push([P(destLL(pts[i][0],pts[i][1],1.609344,(brg+270)%360)),P(destLL(pts[i][0],pts[i][1],1.609344,(brg+90)%360))]);
        }
      }
      x.lineCap='square'; x.lineJoin='miter';
      for(const [w,col] of [[4,'#000'],[2,'#fff']]){
        x.lineWidth=w; x.strokeStyle=col; x.beginPath();
        for(const t of ticks){ x.moveTo(t[0].x,t[0].y); x.lineTo(t[1].x,t[1].y); }
        for(const l of segs) l.forEach((p,i)=>i?x.lineTo(p.x,p.y):x.moveTo(p.x,p.y));
        x.stroke();
      }
      x.lineWidth=1; x.strokeStyle='#000'; x.fillStyle='#fff';
      for(const c of dots){ x.beginPath(); x.arc(c.x,c.y,3,0,7); x.fill(); x.stroke(); }
      if(m.getZoom()>=9){
        x.font='600 10px "IBM Plex Mono",monospace'; x.textBaseline='middle'; x.lineWidth=3; x.strokeStyle='rgba(0,0,0,.9)'; x.fillStyle='#fff';
        cells.storms.forEach((s,i)=>{ const c=dots[i]; x.strokeText(s.id,c.x+6,c.y-7); x.fillText(s.id,c.x+6,c.y-7); });
      }
    }
    if(cells && state.meso){
      for(const z of cells.meso){
        const c=P(z.pos), strong=z.type===9||z.type===10||z.type===1||z.type===3, shear=z.type===2||z.type===4;
        const r=Math.max(strong?9:7,z.radiusKm*pxPerKm(z.pos[0],z.pos[1]));
        const col=strong?'#ffd600':shear?'#c9c9c9':'#fff07a';
        x.lineWidth=strong?5:4; x.strokeStyle='#000'; x.beginPath(); x.arc(c.x,c.y,r,0,7); x.stroke();
        x.lineWidth=strong?3:1.6; x.strokeStyle=col; x.beginPath(); x.arc(c.x,c.y,r,0,7); x.stroke();
        if(strong){ for(const a of [0,90,180,270]){ const q=a*Math.PI/180; x.lineWidth=4; x.strokeStyle='#000'; x.beginPath(); x.moveTo(c.x+Math.cos(q)*r,c.y+Math.sin(q)*r); x.lineTo(c.x+Math.cos(q)*(r+6),c.y+Math.sin(q)*(r+6)); x.stroke(); x.lineWidth=2; x.strokeStyle=col; x.stroke(); } }
        this.hit.push({x:c.x,y:c.y-r,kind:'meso',z,base:cells.mesoT,src:cells.src});
        this.hit.push({x:c.x,y:c.y+r,kind:'meso',z,base:cells.mesoT,src:cells.src});
      }
    }
    if(cells && state.hail){
      for(const h of cells.hail){
        if(h.pHail<30&&h.pSvr<30) continue;
        const c=P(h.pos), r=h.pSvr>=50?9:7, sev=h.pSvr>=50;
        x.beginPath(); x.moveTo(c.x,c.y+r*.9); x.lineTo(c.x-r,c.y-r*.8); x.lineTo(c.x+r,c.y-r*.8); x.closePath();
        x.lineWidth=4; x.strokeStyle='#000'; x.stroke(); x.lineWidth=2; x.strokeStyle='#46e07a'; x.stroke();
        if(sev){ x.fillStyle='#46e07a'; x.fill(); } else if(h.pHail>=50){ x.fillStyle='rgba(70,224,122,.25)'; x.fill(); }
        if(h.size>0&&h.size<=6&&m.getZoom()>=8){ const txt=`${h.size}"`; x.font='600 10px "IBM Plex Mono",monospace'; x.textBaseline='middle'; x.lineWidth=3; x.strokeStyle='#000'; x.strokeText(txt,c.x+r+3,c.y); x.fillStyle='#b8ffcf'; x.fillText(txt,c.x+r+3,c.y); }
        this.hit.push({x:c.x,y:c.y,kind:'hail',h,base:cells.hailT,src:cells.src});
      }
    }
    if(typeof drawMetars==='function') drawMetars(x,P,W,H,this.hit,fr);
    if(typeof drawReports==='function') drawReports(x,P,W,H,this.hit,fr);
    if(cells && state.tvs){
      for(const t of cells.tvs){
        const c=P(t.pos), sp=tornadoSprites[t.elevated?1:0], z=m.getZoom(), sz=sp.w*(z<7?.75:1);
        x.drawImage(sp.c,c.x-sz*.48,c.y-sz*.93,sz,sz);
        this.hit.push({x:c.x,y:c.y-sz*.5,kind:'tvs',t,base:cells.tvsT,src:cells.src});
      }
    }
  }
});
function bearingDeg(a,b){ const p1=a[0]*Math.PI/180,p2=b[0]*Math.PI/180,dl=(b[1]-a[1])*Math.PI/180; return (Math.atan2(Math.sin(dl)*Math.cos(p2),Math.cos(p1)*Math.sin(p2)-Math.sin(p1)*Math.cos(p2)*Math.cos(dl))*180/Math.PI+360)%360; }
function destLL(lat,lon,km,brg){ const R=6371,d=km/R,b=brg*Math.PI/180,p1=lat*Math.PI/180,l1=lon*Math.PI/180; const p2=Math.asin(Math.sin(p1)*Math.cos(d)+Math.cos(p1)*Math.sin(d)*Math.cos(b)); const l2=l1+Math.atan2(Math.sin(b)*Math.sin(d)*Math.cos(p1),Math.cos(d)-Math.sin(p1)*Math.sin(p2)); return [p2*180/Math.PI,l2*180/Math.PI]; }

/* ================= state ================= */
const state={
  site:null, src:store.get('src','L3'), prod:store.get('prod','REF'), tilt:0,
  nFrames:Math.min(200,store.get('nFrames',12)), speedIdx:store.get('speedIdx',3), idx:0, playing:false,
  frames:[], smooth:store.get('smooth',false), opacity:store.get('opacity',90)/100,
  alerts:true, tracks:true, tvs:true, hail:store.get('hail',false), meso:store.get('meso',false), glm:store.get('glm',false), glmWin:5, glmFade:store.get('glmFade',true), glmRange:store.get('glmRange',false), gen:0
};
const frameCache=new Map(); // url -> entry
const radar=new RadarGL('radar','radar-canvas'); radar.addTo(map);
const overlay=new Overlay('cells','ovl-canvas'); overlay.addTo(map);
const curFrame=()=>state.frames[state.idx];

/* ================= sites ================= */
let SITES=[]; const siteRenderer=L.canvas({pane:'sites',padding:.3});
const siteLayer=L.layerGroup().addTo(map);
function parseFallback(){ return FALLBACK_SITES.split('|').map(r=>{ const p=r.split(' '); return {id:p[0],lat:+p[1],lon:+p[2],name:p.slice(3).join(' '),type:p[0][0]==='T'&&p[0]!=='TJUA'?'TDWR':'88D'}; }); }
async function loadSites(){
  try{
    const r=await fetch('https://api.weather.gov/radar/stations',{headers:{Accept:'application/geo+json'}});
    if(!r.ok) throw 0; const j=await r.json();
    SITES=j.features.map(f=>({id:f.properties.id,name:f.properties.name,lat:f.geometry.coordinates[1],lon:f.geometry.coordinates[0],type:/TDWR/i.test(f.properties.stationType)?'TDWR':'88D'})).filter(s=>/^[A-Z]{4}$/.test(s.id));
    if(SITES.length<50) throw 0;
  }catch(e){ SITES=parseFallback(); }
  const have=new Set(SITES.map(x=>x.id)); for(const m of mrrlSites().concat(ideamSites())) if(!have.has(m.id)){ SITES.push(m); have.add(m.id); }
  discoverIdeam();
  SITES.sort((a,b)=>a.id.localeCompare(b.id));
  drawSites();
}
function drawSites(){
  siteLayer.clearLayers(); if(!$('#lySites').checked) return;
  const showM=$('#lyMrrl').checked;
  for(const s of SITES){
    const sel=state.site&&s.id===state.site.id, m=isMrrl(s);
    if(m&&!showM&&!sel) continue;
    const mk=L.circleMarker([s.lat,s.lon],{renderer:siteRenderer,pane:'sites',radius:sel?6:(m?3.4:s.type==='TDWR'?3.5:4.5),weight:sel?2:1.2,color:sel?'#fff':'#000',fillColor:sel?'#ffb347':(m?'#c3a6ff':s.type==='TDWR'?'#7fe3d6':'#d9c7a3'),fillOpacity:sel?1:.85});
    mk.bindTooltip(m?`${s.id} · ${s.name}`:s.id,{className:'site-tip',direction:'top',offset:[0,-6]});
    mk.on('click',()=>selectSite(s.id));
    siteLayer.addLayer(mk);
  }
}
function renderPicker(q=''){
  q=q.trim().toUpperCase();
  const list=SITES.filter(s=>!q||s.id.includes(q)||s.name.toUpperCase().includes(q)||(isMrrl(s)&&(s.cc===q||s.region===q||'MRRL'===q))).slice(0,350);
  const esc=t=>String(t).replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
  $('#siteList').innerHTML=list.map(s=>`<button data-id="${s.id}" class="${state.site&&s.id===state.site.id?'sel':''}"><span class="i">${s.id}</span><span class="n">${esc(s.name)}</span><span class="t ${s.type==='TDWR'?'tdwr':isMrrl(s)?'mrrl':''}">${s.type==='TDWR'?'TDWR':isMrrl(s)?s.cc:'88D'}</span></button>`).join('')||'<div style="padding:14px;color:var(--mute)">No match</div>';
}

function selectSite(id,fly=true){
  const s=SITES.find(x=>x.id===id); if(!s) return;
  state.site=s; store.set('site',id);
  if(isMrrl(s)){ if(!famAnywhere(famById(state.prod),s)) state.prod='REF'; state.tilt=0; }
  else if(s.type==='TDWR'){ if(!famById(state.prod).tdwr){ state.prod='REF'; toast(`${famById(state.prod).name} isn't produced by TDWR sites. Switched to reflectivity.`); } if(state.src==='L2'){ state.src='L3'; } }
  else if(!famAvail(famById(state.prod),s,state.src)) state.src=famById(state.prod).l3?'L3':'L2';
  $('#picker').hidden=true;
  if(fly) map.flyTo([s.lat,s.lon],Math.max(map.getZoom(),s.type==='TDWR'?8:isMrrl(s)&&s.rangeKm<100?9:7),{duration:.6});
  cellIndex={nst:[],ntv:[],nhi:[],nmd:[],src:null}; updateCellSourceUI();
  imagery.clear(); imgClip(); if(typeof satSet==='function'&&sat.on) setTimeout(satSet,0); if(typeof wind!=='undefined'){ wind.site=null; wind.warned=false; pool.drop(j=>j.tag==='wind'); }
  updateProductUI(); drawSites(); reload(true);
}

/* ================= product UI ================= */
function updateProductUI(){
  const s=state.site, tdwr=s&&s.type==='TDWR', fam=famById(state.prod);
  const mr=isMrrl(s), raw=mrrlRaw(s);
  $('#siteId').textContent=s?s.id:'----'; $('#siteName').textContent=s?s.name:''; $('#siteType').textContent=mr?`MRRL · ${s.region} · ${SRC_NAME[raw?s.src:(s.src==='geomet'?'geomet':'rv')]}`:tdwr?'TDWR · Terminal Doppler':'WSR-88D · NEXRAD';
  $('#srcL2').disabled=mr||tdwr||!fam.l2; $('#srcL3').disabled=mr||!(tdwr?fam.tdwr:fam.l3);
  $$('[data-src]').forEach(b=>b.setAttribute('aria-pressed',b.dataset.src===state.src));
  $('#prodStrip').innerHTML=CATALOG.map(f=>{ const ok=famAnywhere(f,s); return `<button data-fam="${f.id}" aria-pressed="${f.id===state.prod}" ${ok?'':'disabled'} title="${f.name}${f.l2&&!f.l3?' (Level II)':(!f.l2?' (Level III)':'')}">${f.label}</button>`; }).join('');
  const cur=$('#prodStrip [aria-pressed="true"]'); if(cur) cur.scrollIntoView({block:'nearest',inline:'nearest'});
  const sel=$('#tilt'); let opts;
  if(mr){ opts=raw?(s.src==='dwd'?DWD_SWEEPS.map((w,i)=>[i,`${w[1].toFixed(1)}° · sweep ${w[0]}`]):(s.elevs||[0,1,2,3,4,5]).map((e,i)=>[i,s.elevs?`${e.toFixed(1)}°`:`Tilt ${i+1}`])):[[0,'Composite']]; }
  else if(state.src==='L3'){ const list=tdwr?fam.tdwr:fam.l3; opts=list.map((c,i)=>[i,`${tdwr&&fam.tdwrNames?fam.tdwrNames[i]:(list.length>1?'Tilt '+(i+1):'Product')} · ${c}`]); }
  else opts=[0,1,2,3,4,5].map(i=>[i,`Tilt ${i+1}`]);
  if(state.tilt>=opts.length) state.tilt=0;
  sel.innerHTML=opts.map(o=>`<option value="${o[0]}" ${o[0]===state.tilt?'selected':''}>${o[1]}</option>`).join('');
  sel.disabled=opts.length<2;
  $('#prodName').innerHTML=mr?(raw?`<b>${fam.name}</b> · ${SRC_NAME[s.src]}`:`<b>Precipitation composite</b> · clipped to ${s.rangeKm} km`):`<b>${fam.name}</b> · ${state.src==='L2'?'Level II':'Level III'}`;
  drawLegend(fam.pal);
  $('#legend').classList.toggle('img',mr&&!raw);
  if(mr&&!raw){ $('#legendName').textContent=s.src==='geomet'?'RAIN RATE':'PRECIP'; $('#legendU').textContent=s.src==='geomet'?'mm/h':'';
    $('#legendNote').textContent=s.src==='geomet'?'ECCC 1 km composite · 6 min':'RainViewer Universal Blue · 10 min · native to zoom 7'; }
  $('#lyTracks').disabled=$('#lyTVS').disabled=false;
}
function pickFamily(id){
  const f=famById(id), s=state.site; if(!famAnywhere(f,s)) return;
  state.prod=id; store.set('prod',id);
  if(!famAvail(f,s,state.src)){ const ns=state.src==='L2'?'L3':'L2'; if(famAvail(f,s,ns)){ state.src=ns; store.set('src',ns); } }
  state.tilt=0; updateProductUI(); reload(true);
}

/* ================= frames ================= */
function l3Codes(ctx=state){ const f=famById(ctx.prod); const td=state.site.type==='TDWR'; const list=td?f.tdwr:f.l3; if(!list) return [null,null]; const i=Math.min(ctx.tilt,list.length-1); const c=list[i]; return [c,!td&&f.fb?f.fb[i]:null]; }
function dayPrefixes(){ const n=state.nFrames>100?3:2; return Array.from({length:n},(_,i)=>new Date(Date.now()-(n-1-i)*86400000)); }
async function listRadarFrames(ctx=state){
  const s=state.site, out=[];
  if(isMrrl(s)) return listMrrl(s);
  if(ctx.src==='L2'){
    for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d);
      const keys=await cachedList(S3.l2,`${Y}/${M}/${D}/${s.id}/`,55000).catch(()=>[]);
      for(const k of keys){ const m=k.match(/\/([A-Z]{4})(\d{4})(\d{2})(\d{2})_(\d{2})(\d{2})(\d{2})_V0\d$/); if(m) out.push({t:new Date(Date.UTC(+m[2],m[3]-1,+m[4],+m[5],+m[6],+m[7])),url:`${S3.l2}/${k}`,type:'l2'}); }
    }
    out.sort((a,b)=>a.t-b.t);
    // real-time volumes not yet in the archive
    try{
      const live=await l2LiveVolumes(s.id);
      const lastArch=out.length?+out[out.length-1].t:0;
      for(const v of live) if(+v.t>lastArch+60e3) out.push({t:v.t,url:`live:${s.id}:${v.vol}:${+v.t}`,type:'l2c',urls:v.urls,live:true});
    }catch(e){ console.warn('L2 real-time',e); }
  } else {
    const id3=s.id.slice(1); let [code,fbc]=l3Codes(ctx);
    const get=async c=>{ const r=[]; for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d);
      const keys=await cachedList(S3.l3,`${id3}_${c}_${Y}_${M}_${D}`,55000).catch(()=>[]);
      for(const k of keys){ const m=k.match(/_(\d{4})_(\d{2})_(\d{2})_(\d{2})_(\d{2})_(\d{2})$/); if(m) r.push({t:new Date(Date.UTC(+m[1],m[2]-1,+m[3],+m[4],+m[5],+m[6])),url:`${S3.l3}/${k}`,type:'l3'}); } } return r; };
    let r=await get(code);
    const recent=r.filter(f=>Date.now()-f.t<3*3600e3);
    if(!recent.length && fbc){ const r2=await get(fbc); if(r2.length){ r=r2; code=fbc; } }
    ctx.l3code=code; out.push(...r);
  }
  out.sort((a,b)=>a.t-b.t);
  return out;
}
let cellIndex={nst:[],ntv:[],nhi:[],nmd:[],src:null};
/* ---- MRRL sources ---- */
async function listMrrl(s){
  if(!s.rawBlocked){
    const order=[]; if(s.src==='dwd'||s.src==='fmi'||s.src==='ideam'||s.src==='opera') order.push(s.src);
    if(EU.has(s.cc)&&!order.includes('opera')) order.push('opera');
    if(s.origSrc&&!order.includes(s.origSrc)) order.unshift(s.origSrc);
    const errs=[];
    for(const src of order){
      try{ const r=src==='dwd'?await listDWD(s):src==='fmi'?await listFMI(s):src==='ideam'?await listIDEAM(s):await listOperaSite(s);
        if(r.length){ if(!s.origSrc) s.origSrc=s.src; if(s.src!==src){ s.src=src; s.elevs=null; updateProductUI(); } return r; } errs.push(`${SRC_NAME[src]}: no recent files`); }
      catch(e){ console.warn('MRRL raw',s.id,src,e); errs.push(`${SRC_NAME[src]}: ${/fetch|network|cors/i.test(e.message)?'blocked by the server':e.message}`); }
    }
    if(order.length){ s.rawBlocked=true; toast(`${s.id}: ${errs.join('; ')}. Showing the composite.`,7000); if(state.prod!=='REF') state.prod='REF'; state.tilt=0; updateProductUI(); }
  }
  return s.cc==='CA'?listGeoMet():listRainViewer();
}
/* ---- IDEAM (Colombia) Sigmet/IRIS volumes on AWS ---- */
const IDEAM_BUCKET='https://s3-radaresideam.s3.amazonaws.com';
async function listIDEAM(s){
  const out=[], seen=new Set();
  for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d);
    const keys=await cachedList(IDEAM_BUCKET,`l2_data/${Y}/${M}/${D}/${s.folder}/`,55000);
    for(const k of keys){ const m=k.match(/([A-Z]{3})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})\.RAW[^/]*$/); if(!m||seen.has(k)) continue; seen.add(k);
      out.push({t:new Date(Date.UTC(2000+ +m[2],m[3]-1,+m[4],+m[5],+m[6],+m[7])),url:`${IDEAM_BUCKET}/${k.split('/').map(encodeURIComponent).join('/')}`,type:'iris'}); } }
  return out.sort((a,b)=>a.t-b.t);
}
async function discoverIdeam(){
  try{ const [Y,M,D]=ymdU(new Date()); const ps=await s3prefixes(IDEAM_BUCKET,`l2_data/${Y}/${M}/${D}/`);
    const known=new Set(IDEAM_SITES.map(x=>x[0])); let added=0;
    for(const p of ps){ const folder=p.split('/').filter(Boolean).pop(); if(!folder||known.has(folder)) continue;
      SITES.push({id:'CO'+folder.slice(0,3).toUpperCase()+(SITES.some(x=>x.id==='CO'+folder.slice(0,3).toUpperCase())?'2':''),folder,lat:4.6,lon:-74.1,approx:true,cc:'CO',region:'CO',name:`${folder.replace(/_/g,' ')} (IDEAM, location from first scan)`,type:'MRRL',src:'ideam',rangeKm:240}); added++; }
    if(added){ SITES.sort((a,b)=>a.id.localeCompare(b.id)); drawSites(); }
  }catch(e){ /* bucket listing blocked; the static list still works */ }
}
function ideamLearn(s,d){ if(s.src!=='ideam'||!d||d.lat==null||isNaN(d.lat)) return; if(!s.approx&&Math.abs(s.lat-d.lat)<.01&&Math.abs(s.lon-d.lon)<.01) return;
  s.lat=+d.lat.toFixed(4); s.lon=+d.lon.toFixed(4); s.approx=false; const fx=store.get('ideamFix',{}); fx[s.folder]=[s.lat,s.lon]; store.set('ideamFix',fx); drawSites(); }

/* ---- OPERA single-site volumes (EUMETNET open radar data, CloudFerro S3) ---- */
const OPERA_BUCKET='https://s3.waw3-1.cloudferro.com/openradar-24h';
const operaNodes=new Map();
const norm=t=>String(t||'').normalize('NFD').replace(/[̀-ͯ]/g,'').toLowerCase().replace(/[^a-z ]/g,' ');
async function operaNode(s){
  if(s.opNode) return s.opNode;
  const cc=s.cc, [Y,M,D]=ymdU(new Date());
  let nodes=operaNodes.get(cc);
  if(!nodes){ const ps=await s3prefixes(OPERA_BUCKET,`${Y}/${M}/${D}/${cc}/`); nodes=ps.map(p=>p.split('/').filter(Boolean).pop()).filter(Boolean); operaNodes.set(cc,nodes); }
  if(!nodes.length) throw new Error(`no ${cc} sites in OPERA today`);
  const id=s.id.toLowerCase().replace(/_+$/,''), cl=cc.toLowerCase(), words=norm(s.name.split(',')[0]).split(/\s+/).filter(w=>w.length>2);
  const cands=[cl+id.slice(-3), cl+id.slice(1,4), cl+id.slice(0,3), id, ...words.map(w=>cl+w.slice(0,3)), cl+words.map(w=>w[0]).join('').slice(0,3)];
  let hit=nodes.find(n=>cands.includes(n.toLowerCase()));
  if(!hit&&words.length) hit=nodes.find(n=>words.some(w=>w.startsWith(n.toLowerCase().slice(2,5))));
  if(!hit) throw new Error('no matching OPERA site code');
  return s.opNode=hit;
}
async function listOperaSite(s){
  const node=await operaNode(s), out=[], vel=state.prod==='VEL', q=vel?/VRAD/:/DBZH|TH/;
  for(const d of dayPrefixes().slice(-2)){ const [Y,M,D]=ymdU(d);
    const keys=await cachedList(OPERA_BUCKET,`${Y}/${M}/${D}/${s.cc}/${node}/PVOL/`,55000).catch(()=>[]);
    const byT=new Map();
    for(const k of keys){ if(!/\.h5$/i.test(k)) continue; const m=k.match(/(\d{8})T(\d{4})(\d{2})?/); if(!m) continue; const t=Date.UTC(+m[1].slice(0,4),m[1].slice(4,6)-1,+m[1].slice(6,8),+m[2].slice(0,2),+m[2].slice(2,4));
      const qn=(k.match(/@([A-Z]+[A-Z0-9]*)\.h5$/i)||[])[1]||''; const cur=byT.get(t); const good=!qn||q.test(qn)||/PVOL/i.test(qn);
      if(good&&(!cur||(qn&&q.test(qn)))) byT.set(t,k); }
    for(const [t,k] of byT) out.push({t:new Date(t),url:`${OPERA_BUCKET}/${k.split('/').map(encodeURIComponent).join('/')}`,type:'odim'});
  }
  return out.sort((a,b)=>a.t-b.t);
}
async function listDWD(s,forceVel,ctx=state){
  const vel=forceVel||ctx.prod==='VEL', q=vel?'vradh':'dbzh';
  const base=`https://opendata.dwd.de/weather/radar/sites/sweep_vol_${vel?'v':'z'}/${s.code}/hdf5/filter_polarimetric/`;
  const r=await fetch(base); if(!r.ok) throw new Error('HTTP '+r.status);
  const html=await r.text(), sweep=DWD_SWEEPS[forceVel&&ctx===state?0:Math.min(ctx.tilt,5)][0], out=[], seen=new Set();
  const re=new RegExp(`href="([^"]*_${q}_(\\d\\d)-(\\d{14})\\d*-${s.code}-\\d+-hd5)"`,'g');
  for(const m of html.matchAll(re)){ if(m[2]!==sweep||seen.has(m[3])) continue; seen.add(m[3]); const t=m[3];
    out.push({t:new Date(Date.UTC(+t.slice(0,4),t.slice(4,6)-1,+t.slice(6,8),+t.slice(8,10),+t.slice(10,12),+t.slice(12,14))),url:new URL(m[1],base).href,type:'odim'}); }
  return out.sort((a,b)=>a.t-b.t);
}
const FMI_BUCKET='https://fmi-opendata-radar-volume-hdf5.s3.eu-west-1.amazonaws.com';
async function listFMI(s){
  const out=[], seen=new Set();
  for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d);
    const keys=await cachedList(FMI_BUCKET,`${Y}/${M}/${D}/${s.code}/`,55000);
    for(const k of keys){ const m=k.match(/(\d{12})[^/]*\.h5$/); if(!m||seen.has(m[1])) continue; seen.add(m[1]); const t=m[1];
      out.push({t:new Date(Date.UTC(+t.slice(0,4),t.slice(4,6)-1,+t.slice(6,8),+t.slice(8,10),+t.slice(10,12))),url:`${FMI_BUCKET}/${k}`,type:'odim'}); }
  }
  return out.sort((a,b)=>a.t-b.t);
}
let rvCache=null;
async function listRainViewer(){
  if(!rvCache||Date.now()-rvCache.at>60e3){ const r=await fetch('https://api.rainviewer.com/public/weather-maps.json'); if(!r.ok) throw new Error('RainViewer '+r.status); rvCache={at:Date.now(),j:await r.json()}; }
  const j=rvCache.j; return ((j.radar&&j.radar.past)||[]).map(p=>({t:new Date(p.time*1000),url:`${j.host}${p.path}/256/{z}/{x}/{y}/2/1_1.png`,type:'img',kind:'xyz'}));
}
let gmCache=null;
async function listGeoMet(){
  let times=null;
  try{
    if(!gmCache||Date.now()-gmCache.at>60e3){
      const r=await fetch('https://geo.weather.gc.ca/geomet?service=WMS&version=1.3.0&request=GetCapabilities&layer=RADAR_1KM_RRAI'); if(!r.ok) throw 0;
      const m=(await r.text()).match(/<Dimension[^>]*name="time"[^>]*>([^<]+)<\/Dimension>/); if(!m) throw 0;
      const [a,b,step]=m[1].trim().split('/'); const st=(+(step.match(/(\d+)M/)||[0,6])[1])*60e3;
      const out=[]; for(let t=Date.parse(b);t>=Date.parse(a);t-=st) out.unshift(t); gmCache={at:Date.now(),times:out};
    }
    times=gmCache.times;
  }catch(e){ // capabilities blocked: 6-minute steps ending ~8 min ago
    const end=Math.floor((Date.now()-8*60e3)/360e3)*360e3; times=[]; for(let k=29;k>=0;k--) times.push(end-k*360e3);
  }
  return times.map(t=>({t:new Date(t),url:`geomet:${t}`,type:'img',kind:'wms',time:new Date(t).toISOString().replace(/\.\d+Z$/,'Z')}));
}

/* ---- composite imagery, one tile layer per frame, clipped to the radar's range ---- */
const imgPane=map.createPane('imagery'); imgPane.style.zIndex=349; imgPane.style.pointerEvents='none';
const imagery={
  layers:new Map(), cur:null,
  ensure(f,onload){
    let lyr=this.layers.get(f.url); if(lyr) return lyr;
    const common={pane:'imagery',opacity:0,keepBuffer:2,updateWhenZooming:false,updateWhenIdle:true,crossOrigin:'anonymous'};
    lyr=f.kind==='wms'
      ? L.tileLayer.wms('https://geo.weather.gc.ca/geomet',Object.assign({layers:'RADAR_1KM_RRAI',format:'image/png',transparent:true,version:'1.3.0',time:f.time,tileSize:512,attribution:'Radar &copy; ECCC'},common))
      : L.tileLayer(f.url,Object.assign({maxNativeZoom:7,maxZoom:19,attribution:'Radar &copy; RainViewer'},common));
    lyr.once('load',()=>onload&&onload());
    lyr.addTo(map); this.layers.set(f.url,lyr); return lyr;
  },
  show(f){ for(const [u,l] of this.layers) l.setOpacity(f&&u===f.url?state.opacity:0); this.cur=f; },
  prune(keep){ for(const [u,l] of this.layers) if(!keep.has(u)){ map.removeLayer(l); this.layers.delete(u); } },
  clear(){ for(const l of this.layers.values()) map.removeLayer(l); this.layers.clear(); this.cur=null;
    for(const e of frameCache.values()) if(e.type==='img'){ e.status='idle'; } }
};
function imgClip(e){
  const s=state.site; if(!isMrrl(s)){ imgPane.style.clipPath=''; return; }
  const anim=e&&e.center&&e.zoom!=null;
  const proj=anim?(ll=>map._latLngToNewLayerPoint(L.latLng(ll),e.zoom,e.center)):(ll=>map.latLngToLayerPoint(L.latLng(ll)));
  const c=proj([s.lat,s.lon]), edge=proj(destLL(s.lat,s.lon,s.rangeKm,90)), r=Math.abs(edge.x-c.x);
  imgPane.style.transition=anim?'clip-path .25s cubic-bezier(0,0,.25,1)':'none';
  imgPane.style.clipPath=`circle(${r.toFixed(1)}px at ${c.x.toFixed(1)}px ${c.y.toFixed(1)}px)`;
}
map.on('zoomanim',imgClip); map.on('zoom zoomend viewreset moveend',()=>imgClip());

function kmBetween(a,b){ const R=6371,p1=a.lat*Math.PI/180,p2=b.lat*Math.PI/180,dp=p2-p1,dl=(b.lon-a.lon)*Math.PI/180; return 2*R*Math.asin(Math.sqrt(Math.sin(dp/2)**2+Math.cos(p1)*Math.cos(p2)*Math.sin(dl/2)**2)); }
// TDWRs don't produce SCIT/TVS products, so use the closest WSR-88D that has recent ones
function cellCandidates(s){
  if(isMrrl(s)) return s.cc==='US'?SITES.filter(x=>x.type==='88D').map(x=>({x,d:kmBetween(s,x)})).filter(o=>o.d<=230).sort((a,b)=>a.d-b.d).slice(0,2).map(o=>Object.assign({km:Math.round(o.d)},o.x)):[];
  if(s.type!=='TDWR') return [s];
  // the TDWR's own SCIT/TVS products first, nearest WSR-88D only as a fallback
  return [s].concat(SITES.filter(x=>x.type==='88D').map(x=>({x,d:kmBetween(s,x)})).filter(o=>o.d<=300).sort((a,b)=>a.d-b.d).slice(0,3).map(o=>Object.assign({km:Math.round(o.d)},o.x)));
}
async function listCellsFor(site){
  const id3=site.id.slice(1), out={nst:[],ntv:[],nhi:[],nmd:[],src:site};
  for(const p of ['NST','NTV','NHI','NMD']){ const arr=[];
    for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d);
      const keys=await cachedList(S3.l3,`${id3}_${p}_${Y}_${M}_${D}`,55000).catch(()=>[]);
      for(const k of keys){ const m=k.match(/_(\d{4})_(\d{2})_(\d{2})_(\d{2})_(\d{2})_(\d{2})$/); if(m) arr.push({t:+new Date(Date.UTC(+m[1],m[2]-1,+m[3],+m[4],+m[5],+m[6])),url:`${S3.l3}/${k}`}); }
    }
    arr.sort((a,b)=>a.t-b.t); out[p.toLowerCase()]=arr;
  }
  return out;
}
async function listCells(){
  const s=state.site; if(!s){ cellIndex={nst:[],ntv:[],nhi:[],nmd:[],src:null}; return; }
  let next={nst:[],ntv:[],nhi:[],nmd:[],src:null};
  for(const c of cellCandidates(s)){
    const r=await listCellsFor(c);
    const recent=r.nst.length&&Date.now()-r.nst[r.nst.length-1].t<2*3600e3;
    if(recent||s.type!=='TDWR'){ next=r; break; }
  }
  if(state.site===s){ cellIndex=next; updateCellSourceUI();
    if(state.tracks&&state.frames.length){ const t0=+state.frames[0].t-15*60e3; for(const e of next.nst) if(e.t>=t0) loadCellFile(e,6); } }
}
function updateCellSourceUI(){
  const s=state.site, src=cellIndex.src, via=s&&src&&src.id!==s.id;
  $('#trkSrc').textContent=via?`via ${src.id}`:'NST'; $('#tvsSrc').textContent=via?`via ${src.id}`:'NTV'; $('#hailSrc').textContent=via?`via ${src.id}`:'NHI'; $('#mesoSrc').textContent=via?`via ${src.id}`:'NMD';
  $('#lyTracks').parentElement.title=$('#lyTVS').parentElement.title=via?`${s.id} has no recent storm tracking products, so these come from ${src.id} (${src.km} km away).`:(s&&s.type==='TDWR'&&!src?'No recent storm tracking products from this TDWR or a WSR-88D within 300 km':'');
}
function nearestBefore(arr,t,maxAge){ let best=null; for(const e of arr){ if(e.t<=t+90e3) best=e; else break; } return best&&t-best.t<maxAge?best:null; }
const cellCache=new Map();
function loadCellFile(e,prio){
  let c=cellCache.get(e.url); if(c) return c;
  c={status:'loading',data:null}; cellCache.set(e.url,c);
  pool.run('cells',e.url,{},prio,'cells').then(d=>{c.status='ok';c.data=d; overlay.redraw();}).catch(()=>{c.status='err';});
  return c;
}
function cellsFor(fr){
  if(!state.site||(!state.tracks&&!state.tvs&&!state.hail&&!state.meso)) return null;
  const t=+fr.t; const a=nearestBefore(cellIndex.nst,t,12*60e3), b=nearestBefore(cellIndex.ntv,t,12*60e3);
  const h=state.hail&&nearestBefore(cellIndex.nhi,t,12*60e3), mz=state.meso&&nearestBefore(cellIndex.nmd,t,12*60e3);
  const A=a&&loadCellFile(a,40), B=b&&loadCellFile(b,40), Hh=h&&loadCellFile(h,35), Mz=mz&&loadCellFile(mz,35);
  const tOf=(F,e)=>F&&F.data&&F.data.time?F.data.time:(F&&F.data&&F.data.vol>1e12?F.data.vol:(e?e.t:null));
  return {storms:A&&A.data?A.data.storms:[], tvs:B&&B.data?B.data.tvs:[], hail:Hh&&Hh.data?Hh.data.hail:[], meso:Mz&&Mz.data?Mz.data.meso:[], t:tOf(A,a), tvsT:tOf(B,b), hailT:tOf(Hh,h), mesoT:tOf(Mz,mz), src:cellIndex.src};
}

/* ---- Level II real-time chunks: 999 round-robin volume folders per site ---- */
const rtHint={}, rtDirs={};
function chunkInfo(k){ const m=k.match(/\/(\d{4})(\d{2})(\d{2})-(\d{2})(\d{2})(\d{2})-(\d{3})-([SIE])$/); return m?{t:Date.UTC(+m[1],m[2]-1,+m[3],+m[4],+m[5],+m[6]),n:+m[7],type:m[8],k}:null; }
async function rtVolume(site,v,ttl){
  const keys=await cachedList(S3.l2rt,`${site}/${v}/`,ttl).catch(()=>[]);
  const ch=keys.map(chunkInfo).filter(Boolean).sort((a,b)=>a.n-b.n);
  if(!ch.length||ch[0].type!=='S') return null;
  return {vol:v,t:ch[0].t,urls:ch.map(c=>`${S3.l2rt}/${c.k}`),done:ch[ch.length-1].type==='E'};
}
async function l2LiveVolumes(site){
  let c=rtDirs[site];
  if(!c||Date.now()-c.at>3600e3){ c=rtDirs[site]={at:Date.now(),p:s3prefixes(S3.l2rt,site+'/').then(ps=>ps.map(x=>+x.split('/')[1]).filter(n=>n>0).sort((a,b)=>a-b))}; c.p.catch(()=>{ delete rtDirs[site]; }); }
  const list=await c.p;
  if(!list.length) return [];
  const n=list.length, idx=v=>list.indexOf(v);
  const T=async(i,ttl)=>{ const r=await rtVolume(site,list[((i%n)+n)%n],ttl); return r?r.t:-Infinity; };
  let L=-1; const h=rtHint[site];
  if(h&&h.at>Date.now()-3*3600e3&&idx(h.vol)>=0){ L=idx(h.vol); }
  else{ // binary search the rotation point (last folder whose start time >= the first folder's)
    const t0=await T(0,600e3); let lo=0,hi=n-1;
    while(lo<hi){ const mid=Math.ceil((lo+hi)/2); if(await T(mid,600e3)>=t0) lo=mid; else hi=mid-1; }
    L=lo;
  }
  // walk forward with fresh listings while newer volumes exist
  let tL=await T(L,15e3);
  for(let k=0;k<6;k++){ const tn=await T(L+1,15e3); if(tn>tL){ L=(L+1)%n; tL=tn; } else break; }
  rtHint[site]={vol:list[L],at:Date.now()};
  const out=[]; for(const i of [L-1,L]){ const v=await rtVolume(site,list[((i%n)+n)%n],15e3); if(v) out.push(v); }
  return out.filter(v=>Date.now()-v.t<2*3600e3);
}
async function reload(reset){
  const gen=++state.gen;
  if(reset){ pool.drop(j=>j.tag==='radar'||j.tag==='cells'||j.tag==='glm'); state.frames=[]; radar.setData(null); imagery.clear(); renderScrub(); setStatus('Listing files…'); }
  let list;
  try{ list=await listRadarFrames(); }catch(e){ setStatus('Could not list files: '+e.message); return; }
  if(gen!==state.gen) return;
  if(!list.length){ setStatus(`<b>No ${isMrrl(state.site)?'recent data':state.src==='L2'?'Level II files':(state.l3code||'Level III')+' files'}</b> for ${state.site.id}`); state.frames=[]; renderScrub(); radar.setData(null); overlay.redraw(); return; }
  const atEnd=!state.frames.length||state.idx>=state.frames.length-1;
  const prevT=curFrame()&&+curFrame().t;
  const want=list.slice(-state.nFrames);
  const ck=f=>f.type!=='l3'?`${f.url}|${state.prod}|${state.tilt}`:f.url;
  state.frames=want.map(f=>{ let e=frameCache.get(ck(f)); if(!e){ e={t:f.t,url:f.url,type:f.type,status:'idle',data:null,live:!!f.live,kind:f.kind,time:f.time}; frameCache.set(ck(f),e); } if(f.urls) e.urls=f.urls; return e; });
  if(state.frames.some(f=>f.type==='img')) imagery.prune(new Set(state.frames.map(f=>f.url)));
  state.idx=atEnd?state.frames.length-1:Math.max(0,state.frames.findIndex(f=>+f.t>=prevT));
  renderScrub(); loadFrames(); listCells().then(()=>overlay.redraw()); loadAlerts(); if(state.glm) prefetchGLM();
  if(typeof loadReports==='function') loadReports(); if(typeof wind!=='undefined') wind.refreshBg(); if(typeof loadMetars==='function') loadMetars(false);
  if(typeof split!=='undefined') split.reload();
  showFrame(state.idx);
}
function loadFrames(){
  const tilt=state.tilt, fam=famById(state.prod), block=fam.l2, pal=fam.pal;
  const n=state.frames.length;
  state.frames.forEach((f,i)=>{
    if(f.status==='ok'||f.status==='loading') return;
    f.status='loading';
    if(f.type==='img'){ imagery.ensure(f,()=>{ f.status='ok'; renderScrub(); setLoadStatus(); }); if(curFrame()===f) imagery.show(f); return; }
    const prio=100+i+(i===n-1?100:0);
    const s=state.site, odim=f.type==='odim';
    const iris=f.type==='iris';
    const opts=iris?{prod:state.prod,tilt}:odim?{quants:state.prod==='VEL'?['VRADH','VRAD','VRADDH','VRADV']:['DBZH','TH','DBZ','DBZV'],tilt:s.src==='dwd'?0:tilt,vel:state.prod==='VEL'}:{block,tilt,urls:f.urls};
    pool.run(odim?'odim':f.type,f.url,opts,prio,'radar').then(d=>{
      if((odim||iris)&&s.src!=='dwd'&&d.elevs&&!s.elevs){ s.elevs=d.elevs.slice(0,10); updateProductUI(); }
      if(iris) ideamLearn(s,d);
      d.pal=pal; f.data=d; f.status='ok'; bytesLoaded+=d.bytes||0;
      if(curFrame()===f || (!state.playing && !(curFrame()&&curFrame().status==='ok'))) showFrame(state.idx);
      renderScrub(); setLoadStatus();
    }).catch(e=>{ f.status=(e.message==='cancelled'||e.message==='incomplete')?'idle':'err'; f.pending=e.message==='incomplete'; f.err=e.message; renderScrub(); setLoadStatus(); });
  });
  setLoadStatus();
}
let bytesLoaded=0;
function setLoadStatus(){
  const n=state.frames.length, ok=state.frames.filter(f=>f.status==='ok').length, er=state.frames.filter(f=>f.status==='err').length, pend=state.frames.some(f=>f.pending&&f.status!=='ok');
  const s=state.site, lab=isMrrl(s)?`MRRL · ${SRC_NAME[mrrlRaw(s)?s.src:(s.src==='geomet'?'geomet':'rv')]}`:state.src==='L2'?`Level II · ${famById(state.prod).l2.trim()}`:`Level III · ${state.l3code||''}`;
  setStatus(`<b>${lab}</b> · ${ok}/${n} frames${er?` · ${er} failed`:''}${pend?' · live scan in progress':''} · ${(bytesLoaded/1048576).toFixed(1)} MB`);
}
function setStatus(h){ $('#status').innerHTML=h; }

/* ================= scrubber / playback ================= */
function renderScrub(){
  const el=$('#scrub'); const n=state.frames.length;
  if(el.children.length!==n){ el.innerHTML='<i></i>'.repeat(n); el.style.gap=n>120?'0px':n>60?'1px':'2px'; }
  state.frames.forEach((f,i)=>{ el.children[i].className=(f.status==='ok'?'ok':f.status==='err'?'err':(f.status==='loading'||f.pending)?'loading':'')+(i===state.idx?' cur':''); el.children[i].title=zTime(f.t); });
}
function showFrame(i){
  const n=state.frames.length; if(!n){ $('#fTimeZ').textContent='--:--Z'; $('#fTimeL').textContent='no frames'; return; }
  state.idx=(i+n)%n; const f=curFrame();
  let d=f.data; if(!d&&f.type!=='img'){ for(let k=state.idx;k>=0;k--) if(state.frames[k].data){ d=state.frames[k].data; break; } }
  radar.setData(d); imagery.show(f.type==='img'?f:null);
  overlay.redraw();
  $('#fTimeZ').textContent=zTime(f.t);
  const ageMin=Math.round((Date.now()-f.t)/60000);
  $('#fTimeL').textContent=`${lTime(f.t)} · ${ageMin<1?'now':ageMin+' min ago'}${d&&d.elev!=null?' · '+d.elev.toFixed(1)+'°':''}`;
  $('#elevTxt').textContent=d&&d.elev!=null?`${d.elev.toFixed(1)}° · ${d.gateKm*1000|0} m gates`:f.type==='img'?`composite · ${state.site.rangeKm} km`:'--';
  const last=state.frames[n-1], age=(Date.now()-last.t)/60000, chip=$('#ageChip');
  chip.textContent=age<1?'LIVE':`${Math.round(age)} MIN`; chip.className=age<12?'':age<30?'stale':'old';
  renderScrub(); updateAlerts(f.t); if(state.glm) ensureGLM(f.t,60); updateReadout();
  if(typeof wind!=='undefined') wind.onFrame(f);
  if(typeof split!=='undefined') split.show(state.idx);
  if(typeof mosaicOnFrame==='function') mosaicOnFrame();
  if(typeof opera!=='undefined'&&opera.on&&!state.playing) operaShow(false);
  if(typeof tools!=='undefined'&&tools.refreshPins){ tools.refreshPins(); if(side.kind==='cells'||side.kind==='warnings') side.render(); }
}
let playT=0;
function setPlaying(p){
  state.playing=p; clearTimeout(playT);
  $('#playIco').innerHTML=p?'<path d="M6 4h4v16H6zM14 4h4v16h-4z"/>':'<path d="M7 4v16l13-8z"/>';
  if(p) tick();
}
let rockDir=1;
function tick(){
  if(!state.playing) return;
  const n=state.frames.length; if(!n){ playT=setTimeout(tick,500); return; }
  const ok=k=>state.frames[k].status==='ok';
  let j=state.idx;
  if(state.playMode==='rock'&&n>1){
    for(let k=0;k<n;k++){ let nj=j+rockDir; if(nj>=n||nj<0){ rockDir=-rockDir; nj=j+rockDir; } j=nj; if(ok(j)) break; }
  } else { for(let k=0;k<n;k++){ j=(j+1)%n; if(ok(j)) break; } }
  showFrame(j);
  const base=480/SPEEDS[state.speedIdx], end=j===n-1||(state.playMode==='rock'&&j===0);
  playT=setTimeout(tick,end?Math.max(state.dwell*1000,base):base);
}
(function(){
  const el=$('#scrub'); let drag=false;
  const at=e=>{ const r=el.getBoundingClientRect(); const n=state.frames.length; return Math.max(0,Math.min(n-1,Math.floor((e.clientX-r.left)/r.width*n))); };
  el.addEventListener('pointerdown',e=>{ drag=true; el.setPointerCapture(e.pointerId); setPlaying(false); showFrame(at(e)); });
  el.addEventListener('pointermove',e=>{ if(drag) showFrame(at(e)); });
  el.addEventListener('pointerup',()=>drag=false);
})();

/* ================= readout ================= */
let lastPt=null;
function sample(d,ll){
  if(!d) return null;
  const R=6371, la=ll.lat*Math.PI/180, l0=d.lat*Math.PI/180;
  let dl=(ll.lng-d.lon); dl=((dl+540)%360)-180; dl*=Math.PI/180;
  const x=R*dl*Math.cos((la+l0)/2), y=R*(la-l0), r=Math.hypot(x,y); let az=Math.atan2(x,y)*180/Math.PI; if(az<0) az+=360;
  const g=Math.floor((r-d.firstKm)/d.gateKm); if(g<0||g>=d.nGates) return {r,az,b:-1};
  return {r,az,b:d.grid[(Math.floor(az/0.5)%720)*d.nGates+g]};
}
function updateReadout(){
  const ro=$('#readout'); const f=curFrame(); if(!lastPt||!f){ ro.hidden=true; return; }
  const d=f.data||radar.data, s=sample(d,lastPt);
  if(!s){ const eta=tools.track&&tools.eta(lastPt); if(eta){ ro.hidden=false; $('#roV').textContent=''; $('#roK').textContent=eta; } else ro.hidden=true; return; }
  ro.hidden=false;
  $('#roV').textContent=s.b<0?'out of range':fmtValue(d,s.b);
  let k=`${Math.round(s.az)}° · ${fmtDist(s.r)}`;
  if(d.elev!=null&&!d.raster&&s.b>=0){ const ft=beamHeightKm(s.r,d.elev)*3280.84; k+=` · beam ${ft>=10000?(ft/1000).toFixed(1)+'k':Math.round(ft/100)*100} ft`; }
  if(s.b>=0) k+=tools.srvText(d,s.b,lastPt);
  const eta=tools.eta(lastPt); if(eta) k+=` · ${eta}`;
  $('#roK').textContent=k;
}
map.on('mousemove',e=>{ lastPt=e.latlng; updateReadout(); });
map.on('mouseout',()=>{ lastPt=null; $('#readout').hidden=true; });
map.on('click',e=>{
  if(tools.active){ tools.onClick(e); return; }
  lastPt=e.latlng; updateReadout();
  const p=e.containerPoint; let best=null,bd=14;
  for(const h of overlay.hit||[]){ const dd=Math.hypot(h.x-p.x,h.y-p.y); if(dd<bd){bd=dd;best=h;} }
  if(!best) return;
  let html;
  const compass=d=>['N','NNE','NE','ENE','E','ESE','SE','SSE','S','SSW','SW','WSW','W','WNW','NW','NNW'][Math.round(((d%360)+360)%360/22.5)%16];
  const flip=d=>(d+180)%360;
  const at=best.base?lTime(new Date(best.base)):zTime(curFrame().t);
  const fromTxt=best.src&&state.site&&best.src.id!==state.site.id?` · from ${best.src.id}`:'';
  if(best.kind==='metar'){ const o=best.o, F=c=>c==null?'—':`${Math.round(c*9/5+32)}°F / ${Math.round(c)}°C`;
    const rows=[['Observed',`${new Date(o.t).toLocaleString([],{month:'short',day:'numeric',hour:'numeric',minute:'2-digit'})} (${zTime(new Date(o.t))})`],['Temp',F(o.temp)],['Dewpoint',F(o.dewp)],
      ['Wind',o.wspd==null?'—':o.wspd===0?'Calm':`${o.wdir==='VRB'?'Variable':o.wdir+'°'} at ${o.wspd} kt${o.wgst?`, gusts ${o.wgst} kt`:''}`],['Visibility',o.visib!=null?`${o.visib} mi`:'—'],
      ['Sky',o.sky||'—'],['Altimeter',o.altim?`${(o.altim*0.02953).toFixed(2)} inHg`:'—'],['Category',`<b style="color:${FLTCAT[o.cat]||'#ccc'}">${o.cat||'—'}</b>`]];
    if(o.wx) rows.splice(3,0,['Weather',o.wx]);
    html=`<div class="pop"><span class="tag" style="background:${FLTCAT[o.cat]||'#9aa0a8'};color:#0b0b0b">METAR · ${o.id}</span><h4>${(o.name||o.id).replace(/[<>&]/g,'')}</h4><div class="when">${siteRef()?`${state.site.id} ${fromSite([o.lat,o.lon])}`:''}</div>${dl(rows)}${o.raw?`<div style="margin-top:8px;font:500 11px/1.4 var(--mono);color:var(--fg);max-width:300px;word-break:break-word">${o.raw.replace(/[<>&]/g,'')}</div>`:''}</div>`;
  } else if(best.kind==='lsr'||best.kind==='sn'){ const r=best.r, c=rptCat(r.type);
    const rows=[['Time',`${new Date(r.t).toLocaleString([],{month:'short',day:'numeric',hour:'numeric',minute:'2-digit'})} (${zTime(new Date(r.t))})`]];
    if(r.mag!=null&&r.mag!==''&&+r.mag!==0) rows.push(['Magnitude',`<b>${r.mag}${r.unit?' '+r.unit:''}</b>`]);
    if(r.city) rows.push(['Location',`${r.city}${r.county?', '+r.county+' Co.':''}${r.st?', '+r.st:''}`]);
    if(r.src) rows.push(['Source',r.src]); if(r.by) rows.push(['Reported by',r.by]); if(r.wfo) rows.push(['Office',r.wfo]);
    if(siteRef()) rows.push([state.site.id,fromSite([r.lat,r.lon])]);
    const esc=t=>String(t).replace(/[<>&]/g,ch=>({'<':'&lt;','>':'&gt;','&':'&amp;'}[ch]));
    html=`<div class="pop"><span class="tag" style="background:${c[1]};color:${c[3]||'#0b0b0b'}">${best.kind==='sn'?'SPOTTER':'LSR'} · ${esc(r.type).toUpperCase()}</span><h4>${esc(r.type)}</h4><div class="when">${best.kind==='sn'?'SpotterNetwork':r.fromLsr?'Local storm report (spotter source)':'NWS local storm report'}</div><dl>${rows.map(q=>`<dt>${q[0]}</dt><dd>${esc(q[1]).replace(/&lt;(\/?b)&gt;/g,'<$1>')}</dd>`).join('')}</dl>${r.remark?`<div style="margin-top:8px;font-size:12px;color:var(--fg);max-width:290px">${esc(r.remark)}</div>`:''}</div>`;
  } else if(best.kind==='hail'){ const h=best.h;
    html=`<div class="pop"><span class="tag" style="background:#46e07a;color:#062010">HAIL${h.id?' · '+h.id:''}</span><h4>${h.pSvr>=50?'Severe hail likely':h.pHail>=50?'Hail likely':'Hail possible'}</h4><div class="when">${at} · NHI${fromTxt}</div><dl><dt>Prob. of hail</dt><dd><b>${h.pHail}%</b></dd><dt>Prob. severe</dt><dd><b>${h.pSvr}%</b></dd>${h.size>0?`<dt>Max size</dt><dd>${h.size} in</dd>`:''}</dl></div>`;
  } else if(best.kind==='meso'){ const z=best.z;
    const names={1:'Mesocyclone (extrapolated)',2:'3D correlated shear (extrapolated)',3:'Mesocyclone',4:'3D correlated shear',9:'Circulation · strength rank ≥5, low-based',10:'Circulation · strength rank ≥5',11:'Circulation · strength rank <5'};
    html=`<div class="pop"><span class="tag" style="background:#ffd600;color:#1a1208">MESOCYCLONE</span><h4>${names[z.type]||'Circulation (type '+z.type+')'}</h4><div class="when">${at} · NMD${fromTxt}</div><dl><dt>Radius</dt><dd>${z.radiusKm.toFixed(1)} km</dd><dt>Position</dt><dd>${z.pos[0].toFixed(3)}, ${z.pos[1].toFixed(3)}</dd></dl></div>`;
  } else if(best.kind==='cell'){ const s=best.s;
    const rows=[];
    if(!s.isNew&&s.dir!=null) rows.push(['Motion',`<b>${compass(flip(s.dir))}</b> at <b>${Math.round(s.kt*1.150779)}</b> mph (${s.kt} kt)`]);
    else rows.push(['Motion','New cell']);
    if(s.dbzm!=null) rows.push(['Max refl',`<b>${s.dbzm} dBZ</b>`]);
    if(s.hgt!=null) rows.push(['Height of max',`<b>${s.hgt} kft</b>`]);
    if(s.fcst.length) rows.push(['Forecast',`${s.fcst.length*15} min`]);
    if(s.err!=null) rows.push(['Fcst error',`${s.err} / ${s.mean??'—'} nm (err / mean)`]);
    html=`<div class="pop"><span class="tag" style="background:#fff;color:#0b0b0b">STORM TRACK</span><h4>Cell ${s.id} at ${at}</h4><div class="when">SCIT · NST${best.src&&state.site&&best.src.id!==state.site.id?` from ${best.src.id}`:''}</div><dl>${rows.map(r=>`<dt>${r[0]}</dt><dd>${r[1]}</dd>`).join('')}</dl>${trendSVG(cellTrend(s.id))}${!s.isNew&&s.dir!=null&&s.kt?'<button id="trkCell" class="tbtn" style="width:auto;height:auto;margin-top:8px;padding:4px 8px;font:600 10px var(--mono);letter-spacing:.06em">TRACK THIS CELL</button>':''}</div>`;
  } else { const t=best.t;
    const rows=t.az!=null?[['Cell',`<b>${t.cell}</b>`],['Az/Range',`${t.az}° / ${t.ran} nm`],['Avg ΔV',`<b>${t.avgdv} kt</b>`],['Low-level ΔV',`<b>${t.lldv} kt</b>`],['Max ΔV',`<b>${t.mxdv} kt</b> at ${t.mxdvhgt} kft`],['Depth',`${t.depth} kft`],['Base / Top',`${t.base} / ${t.top} kft`],['Max shear',`${t.maxshear} E-3/s at ${t.maxshearhgt} kft`]]:[['Position',`${t.pos[0].toFixed(3)}, ${t.pos[1].toFixed(3)}`]];
    html=`<div class="pop"><span class="tag" style="background:${t.elevated?'#ffd600':'#ff2a3c'};color:${t.elevated?'#1a1208':'#fff'}">${t.elevated?'ETVS':'TVS'}</span><h4>Tornado vortex signature${t.cell?' · '+t.cell:''}</h4><div class="when">${at}${best.src&&state.site&&best.src.id!==state.site.id?` · from ${best.src.id}`:''}</div><dl>${rows.map(r=>`<dt>${r[0]}</dt><dd>${r[1]}</dd>`).join('')}</dl></div>`; }
  L.popup({autoPanPadding:[40,200]}).setLatLng(map.containerPointToLatLng([best.x,best.y])).setContent(html).openOn(map);
  if(best.kind==='cell') setTimeout(()=>{ const b=document.getElementById('trkCell'); if(b) b.onclick=()=>{ trackFromCell(best.s,best.base); map.closePopup(); }; },0);
});

/* ================= alerts (VTEC-synced) ================= */
const ALERT_STYLE={TO:'#ff2e3e',SV:'#ffb020',FF:'#2bd86b',MA:'#ff8a3d',EW:'#ff3fa4',SQ:'#c86bff',DS:'#e8c9a0',SPS:'#78c8ff'};
const alertLayer=L.layerGroup().addTo(map);
let curAlerts=[];
let alertSegs=[], alertKey='';
function parseVTEC(v){ const m=v.match(/\/[OTEX]\.([A-Z]{3})\.([A-Z]{4})\.([A-Z]{2})\.([A-Z])\.(\d{4})\.(\d{6}T\d{4}Z)-(\d{6}T\d{4}Z)\//); if(!m) return null;
  const pt=s=>s.startsWith('000000')?null:Date.UTC(2000+ +s.slice(0,2),+s.slice(2,4)-1,+s.slice(4,6),+s.slice(7,9),+s.slice(9,11));
  return {act:m[1],office:m[2],ph:m[3],sig:m[4],etn:m[5],b:pt(m[6]),e:pt(m[7])}; }
let alertsLoadedAt=0;
async function loadAlerts(){
  if(!state.alerts||!state.frames.length) return;
  if(Date.now()-alertsLoadedAt<55000 && alertSegs.length) { updateAlerts(curFrame().t); return; }
  alertsLoadedAt=Date.now();
  const start=new Date(+state.frames[0].t-4*3600e3).toISOString().replace(/\.\d+Z$/,'Z');
  const feats=new Map();
  const grab=async u=>{ for(let p=0;p<5&&u;p++){ const r=await fetch(u,{headers:{Accept:'application/geo+json'}}); if(!r.ok) break; const j=await r.json(); for(const f of j.features||[]) if(f.geometry) feats.set(f.id,f); u=j.pagination&&j.pagination.next&&(j.features||[]).length?j.pagination.next:null; } };
  try{ await Promise.all([grab(`https://api.weather.gov/alerts?start=${start}&status=actual&limit=500`),grab('https://api.weather.gov/alerts/active?status=actual')]); }catch(e){}
  const byKey=new Map(), segs=[];
  for(const f of feats.values()){
    const P=f.properties, sent=Date.parse(P.sent);
    const vt=((P.parameters&&P.parameters.VTEC)||[]).map(parseVTEC).filter(Boolean).filter(v=>v.sig==='W');
    const end=Date.parse(P.ends||P.expires);
    if(vt.length){
      const live=vt.find(v=>!['CAN','EXP','UPG'].includes(v.act)), v=live||vt[0];
      const key=`${v.office}.${v.ph}.${v.sig}.${v.etn}`;
      const seg={key,ph:v.ph,f,start:Math.max(sent,v.b||0)||sent,sentT:sent,end:live?(v.e||end):sent,dead:!live};
      (byKey.get(key)||byKey.set(key,[]).get(key)).push(seg);
    } else if(P.event==='Special Weather Statement'){ segs.push({key:f.id,ph:'SPS',f,start:Date.parse(P.onset||P.effective||P.sent),end,dead:false}); }
  }
  for(const arr of byKey.values()){ arr.sort((a,b)=>a.sentT-b.sentT); arr.forEach((s,i)=>{ if(arr[i+1]) s.end=Math.min(s.end,arr[i+1].sentT); if(!s.dead) segs.push(s); }); }
  for(const s of segs) s.motion=parseMotion(s.f.properties);
  alertSegs=segs; alertKey=''; updateAlerts(curFrame()&&curFrame().t);
  if(typeof alarm!=='undefined') alarm.check(segs);
}
function alertColor(s){ const P=s.f.properties.parameters||{}; if(s.ph==='TO'&&/CATASTROPHIC/i.test(P.tornadoDamageThreat||'')) return '#d22bff'; if(s.ph==='SV'&&/DESTRUCTIVE/i.test(P.thunderstormDamageThreat||'')) return '#ff7300'; if(s.ph==='FF'&&/CATASTROPHIC/i.test(P.flashFloodDamageThreat||'')) return '#00ffb3'; return ALERT_STYLE[s.ph]||'#cccccc'; }
function updateAlerts(t){
  if(!t) return; t=+t;
  const act=state.alerts?alertSegs.filter(s=>s.start<=t&&t<s.end):[];
  curAlerts=act; overlay.redraw();
  const key=act.map(s=>s.f.id).sort().join(','); if(key===alertKey) return; alertKey=key;
  alertLayer.clearLayers(); if(typeof split!=='undefined') split.syncAlerts(act);
  const order={SPS:0,MA:1,FF:2,SV:3,SQ:4,EW:5,TO:6};
  act.sort((a,b)=>(order[a.ph]||0)-(order[b.ph]||0));
  for(const s of act){
    const col=alertColor(s), P=s.f.properties;
    const casing=L.geoJSON(s.f,{pane:'alerts',interactive:false,style:{color:'#000',weight:5,opacity:.75,fill:false}});
    const poly=L.geoJSON(s.f,{pane:'alerts',bubblingMouseEvents:false,style:{color:col,weight:2.4,opacity:1,fillColor:col,fillOpacity:.08,dashArray:s.ph==='SPS'?'5 4':null}});
    const prm=P.parameters||{}; const rows=[];
    if(prm.tornadoDetection) rows.push(['Tornado',prm.tornadoDetection[0]]);
    if(prm.tornadoDamageThreat) rows.push(['Damage threat',prm.tornadoDamageThreat[0]]);
    if(prm.thunderstormDamageThreat) rows.push(['Damage threat',prm.thunderstormDamageThreat[0]]);
    if(prm.maxHailSize) rows.push(['Hail',prm.maxHailSize[0]+' in']);
    if(prm.maxWindGust) rows.push(['Wind',prm.maxWindGust[0]]);
    if(prm.flashFloodDetection) rows.push(['Flood',prm.flashFloodDetection[0]]);
    if(s.motion) rows.push(['Motion',`toward ${['N','NNE','NE','ENE','E','ESE','SE','SSE','S','SSW','SW','WSW','W','WNW','NW','NNW'][Math.round(((s.motion.from+180)%360)/22.5)%16]} (${(s.motion.from+180)%360}°) at ${s.motion.kt} kt`]);
    rows.push(['Issued',lTime(new Date(s.sentT||s.start))]); rows.push(['Until',lTime(new Date(Date.parse(P.ends||P.expires)))]);
    poly.bindPopup(`<div class="pop"><span class="tag" style="background:${col};color:#0b0b0b">${(P.event||'').toUpperCase()}</span><h4>${P.areaDesc?P.areaDesc.split(';').slice(0,3).join(', '):''}</h4><div class="when">${P.senderName||''}</div><dl>${rows.map(r=>`<dt>${r[0]}</dt><dd>${r[1]}</dd>`).join('')}</dl></div>`,{maxWidth:300});
    alertLayer.addLayer(casing); alertLayer.addLayer(poly);
  }
  const counts={}; for(const s of act) counts[s.ph]=(counts[s.ph]||0)+1;
  const names={TO:'TOR',SV:'SVR',FF:'FFW',MA:'SMW',EW:'EWW',SQ:'SQW',DS:'DSW',SPS:'SPS'};
  const chip=$('#alertChip'); const ent=Object.entries(counts).sort((a,b)=>(order[b[0]]||0)-(order[a[0]]||0));
  chip.hidden=!ent.length; chip.innerHTML=ent.map(([k,v])=>`<span><i style="background:${ALERT_STYLE[k]||'#ccc'}"></i>${v} ${names[k]||k}</span>`).join('');
}

/* ================= GLM ================= */
const glmFiles=new Map(); // url -> {t, status, lat, lon}
const glmHours=new Map();
function glmBase(){ return state.site&&state.site.lon<-109?{b:S3.g18,s:'G18'}:{b:S3.g19,s:'G19'}; }
async function glmList(t0,t1){
  const {b}=glmBase(); const out=[];
  for(let h=Math.floor(t0/3600e3);h<=Math.floor(t1/3600e3);h++){
    const d=new Date(h*3600e3), Y=d.getUTCFullYear(), doy=pad(Math.floor((d-Date.UTC(Y,0,1))/864e5)+1).padStart(3,'0');
    const pre=`GLM-L2-LCFA/${Y}/${doy}/${pad(d.getUTCHours())}/`;
    const fresh=Date.now()-h*3600e3<2*3600e3?50000:3600e3*6;
    const keys=await cachedList(b,pre,fresh).catch(()=>[]);
    for(const k of keys){ const m=k.match(/_s(\d{4})(\d{3})(\d{2})(\d{2})(\d{2})/); if(!m) continue; const t=Date.UTC(+m[1],0,1)+(m[2]-1)*864e5+(+m[3])*3600e3+(+m[4])*60e3+(+m[5])*1e3; out.push({t,url:`${b}/${k}`}); }
  }
  return out;
}
function glmBBox(){ const s=state.site; return [s.lat-6,s.lon-7.5,s.lat+6,s.lon+7.5]; }
async function ensureGLM(t,prio){
  t=+t; const win=state.glmWin*60e3; const list=await glmList(t-win,t);
  const bbox=glmBBox(), site=state.site&&state.site.id;
  let started=0;
  for(const e of list){ if(e.t>t||e.t<=t-win) continue;
    if(glmFiles.has(e.url)) continue;
    const g={t:e.t,status:'loading',lat:[],lon:[],site}; glmFiles.set(e.url,g); started++;
    pool.run('glm',e.url,{bbox},prio,'glm').then(r=>{ g.status='ok'; g.lat=r.lat; g.lon=r.lon; overlay.redraw(); })
      .catch(err=>{ g.status='err'; if(/HDF5/.test(err.message)&&!glmWarned){ glmWarned=true; toast('Lightning decoder could not load from the CDN.'); } if(err.message==='cancelled') glmFiles.delete(e.url); });
  }
}
let glmWarned=false;
function updateGlmStat(n,t){
  const el=$('#glmStat'); if(n==null){ el.hidden=true; return; } el.hidden=false;
  t=+t; const win=state.glmWin*60e3; let pending=false;
  for(const g of glmFiles.values()) if(g.status==='loading'&&g.site===(state.site&&state.site.id)&&g.t<=t&&g.t>t-win){ pending=true; break; }
  $('#glmN').textContent=n.toLocaleString();
  $('#glmRate').textContent=`${Math.round(n/state.glmWin).toLocaleString()}/min`;
  $('#glmScope').textContent=`${state.glmWin} min${state.glmRange?' · in range':''}`;
  el.classList.toggle('loading',pending); el.title=pending?'Still loading lightning files for this window':'GLM flashes in the lightning window';
}
function glmFor(t){ t=+t; const win=state.glmWin*60e3, out=[]; const sid=state.site&&state.site.id;
  for(const g of glmFiles.values()) if(g.status==='ok'&&g.site===sid&&g.t<=t&&g.t>t-win) out.push(g); return out; }
function prefetchGLM(){ const fr=state.frames; if(!fr.length) return; const t1=+fr[fr.length-1].t;
  fr.forEach((f,i)=>{ if(t1-f.t<=30*60e3) ensureGLM(f.t,20+i); }); }

/* ================= controls ================= */
$('#siteBtn').onclick=()=>{ const p=$('#picker'); p.hidden=false; renderPicker(''); $('#siteSearch').value=''; $('#siteSearch').focus(); };
$('#pickClose').onclick=()=>$('#picker').hidden=true;
$('#siteSearch').oninput=e=>renderPicker(e.target.value);
$('#siteSearch').onkeydown=e=>{ if(e.key==='Enter'){ const b=$('#siteList button'); if(b) selectSite(b.dataset.id); } if(e.key==='Escape') $('#picker').hidden=true; };
$('#siteList').onclick=e=>{ const b=e.target.closest('button[data-id]'); if(b) selectSite(b.dataset.id); };
$$('[data-src]').forEach(b=>b.onclick=()=>{ if(b.disabled) return; if(!famAvail(famById(state.prod),state.site,b.dataset.src)) return; state.src=b.dataset.src; store.set('src',state.src); state.tilt=0; updateProductUI(); reload(true); });
$('#prodStrip').onclick=e=>{ const b=e.target.closest('button[data-fam]'); if(b&&!b.disabled) pickFamily(b.dataset.fam); };
$('#tilt').onchange=e=>{ state.tilt=+e.target.value; reload(true); };
$('#layersBtn').onclick=()=>{ const l=$('#layers'); l.hidden=!l.hidden; $('#layersBtn').setAttribute('aria-expanded',!l.hidden); if(!l.hidden){ $('#toolsPanel').hidden=true; $('#allToolsBtn').setAttribute('aria-pressed',false); } };
$('#lyAlerts').onchange=e=>{ state.alerts=e.target.checked; alertKey='__'; overlay.redraw(); if(state.alerts) loadAlerts(); updateAlerts(curFrame()&&curFrame().t); if(!state.alerts) $('#alertChip').hidden=true; };
$('#lyTracks').onchange=e=>{ state.tracks=e.target.checked; overlay.redraw(); };
$('#lyTVS').onchange=e=>{ state.tvs=e.target.checked; overlay.redraw(); };
$('#lyHail').checked=state.hail; $('#lyMeso').checked=state.meso;
$('#lyHail').onchange=e=>{ state.hail=e.target.checked; store.set('hail',state.hail); overlay.redraw(); };
$('#lyMeso').onchange=e=>{ state.meso=e.target.checked; store.set('meso',state.meso); overlay.redraw(); };
$('#lyGLM').checked=state.glm;
$('#lyGLM').onchange=e=>{ state.glm=e.target.checked; store.set('glm',state.glm); if(state.glm){ prefetchGLM(); curFrame()&&ensureGLM(curFrame().t,60);} else pool.drop(j=>j.tag==='glm'); overlay.redraw(); };
$('#lySites').onchange=drawSites;
$('#lyMrrl').checked=store.get('mrrl',true); $('#lyMrrl').onchange=e=>{ store.set('mrrl',e.target.checked); drawSites(); };
$('#lyGlmFade').checked=state.glmFade; $('#lyGlmRange').checked=state.glmRange;
$('#lyGlmFade').onchange=e=>{ state.glmFade=e.target.checked; store.set('glmFade',state.glmFade); overlay.redraw(); };
$('#lyGlmRange').onchange=e=>{ state.glmRange=e.target.checked; store.set('glmRange',state.glmRange); overlay.redraw(); };
$('#lyLabels').onchange=e=>{ if(!refLayer) return; e.target.checked?refLayer.addTo(map):map.removeLayer(refLayer); };
$('#lySmooth').checked=state.smooth;
$('#lySmooth').onchange=e=>{ state.smooth=e.target.checked; store.set('smooth',state.smooth); radar.drawNow(); };
$('#opacity').value=Math.round(state.opacity*100); $('#opacityOut').textContent=$('#opacity').value+'%';
$('#opacity').oninput=e=>{ state.opacity=e.target.value/100; $('#opacityOut').textContent=e.target.value+'%'; store.set('opacity',+e.target.value); radar.drawNow(); if(imagery.cur) imagery.show(imagery.cur); };
$('#glmWin').oninput=e=>{ state.glmWin=+e.target.value; $('#glmWinOut').textContent=e.target.value+' m'; if(state.glm&&curFrame()) ensureGLM(curFrame().t,60); overlay.redraw(); };
$$('.basemaps button').forEach(b=>b.onclick=()=>setBasemap(b.dataset.bm));
$('#frames').value=state.nFrames; $('#framesOut').textContent=state.nFrames;
let frT=0; $('#frames').oninput=e=>{ state.nFrames=+e.target.value; $('#framesOut').textContent=state.nFrames; store.set('nFrames',state.nFrames); clearTimeout(frT); frT=setTimeout(()=>reload(false),250); };
const fmtSpeed=v=>(v<1?v.toFixed(2).replace(/0$/,''):v)+'×';
$('#speed').value=state.speedIdx; $('#speedOut').textContent=fmtSpeed(SPEEDS[state.speedIdx]);
$('#speed').oninput=e=>{ state.speedIdx=+e.target.value; $('#speedOut').textContent=fmtSpeed(SPEEDS[state.speedIdx]); store.set('speedIdx',state.speedIdx); };
$('#playBtn').onclick=()=>setPlaying(!state.playing);
$('#prevBtn').onclick=()=>{ setPlaying(false); showFrame(state.idx-1); };
$('#nextBtn').onclick=()=>{ setPlaying(false); showFrame(state.idx+1); };
document.addEventListener('keydown',e=>{
  if(e.target.matches('input,select,textarea')) return;
  if(e.key===' '){ e.preventDefault(); setPlaying(!state.playing); }
  else if(e.key==='ArrowLeft'){ setPlaying(false); showFrame(state.idx-1); }
  else if(e.key==='ArrowRight'){ setPlaying(false); showFrame(state.idx+1); }
  else if(e.key==='Home'){ setPlaying(false); showFrame(0); }
  else if(e.key==='End'){ setPlaying(false); showFrame(state.frames.length-1); }
  else if(e.key.toLowerCase()==='s'){ e.preventDefault(); $('#siteBtn').click(); }
  else if(e.key==='Escape'){ if(tools.active) tools.set(tools.active); else if(side.kind) side.close(); else tools.hint(null); }
  else if(e.key==='Enter'&&tools.active==='measure'){ tools.finishMeasure(); }
  else if(e.key==='?'){ runTool('help'); }
  else if(e.key==='/'){ e.preventDefault(); runTool('search'); }
  else if(!e.ctrlKey&&!e.metaKey&&!e.altKey&&{i:'inspect',a:'area',o:'radius',e:'stats',k:'vrot',h:'beam',l:'radial',x:'threat',q:'srv',g:'location',c:'cells',w:'warnings',b:'bookmarks',d:'draw',n:'note',u:'grid',z:'snapshot',f:'fullscreen',j:'jump'}[e.key.toLowerCase()]){ e.preventDefault(); runTool({i:'inspect',a:'area',o:'radius',e:'stats',k:'vrot',h:'beam',l:'radial',x:'threat',q:'srv',g:'location',c:'cells',w:'warnings',b:'bookmarks',d:'draw',n:'note',u:'grid',z:'snapshot',f:'fullscreen',j:'jump'}[e.key.toLowerCase()]); }
  else if(e.key.toLowerCase()==='m'){ tools.set('measure'); }
  else if(e.key.toLowerCase()==='t'){ tools.set('track'); }
  else if(e.key.toLowerCase()==='p'){ tools.set('pin'); }
  else if(e.key.toLowerCase()==='r'){ $('#ringsBtn').click(); }
  else if(e.key.toLowerCase()==='v'&&!e.shiftKey){ const t=state.prod==='VEL'?'REF':'VEL'; if(famAnywhere(famById(t),state.site)) pickFamily(t); }
  else if(e.key==='['||e.key===']'){ const av=CATALOG.filter(f=>famAnywhere(f,state.site)); if(av.length<2) return; const i=av.findIndex(f=>f.id===state.prod); pickFamily(av[(i+(e.key===']'?1:-1)+av.length)%av.length].id); }
});
setInterval(()=>{ const d=new Date(); $('#clock').textContent=`${pad(d.getUTCHours())}:${pad(d.getUTCMinutes())}:${pad(d.getUTCSeconds())}Z`; },1000);
let lastRefresh=Date.now();
setInterval(()=>{ if(!state.site||document.hidden) return; const every=state.src==='L2'?30e3:60e3; if(Date.now()-lastRefresh>=every){ lastRefresh=Date.now(); reload(false); } },5000);
document.addEventListener('visibilitychange',()=>{ if(!document.hidden&&state.site) reload(false); });

/* ================= tools ================= */
const UNITS={mi:[0.621371,'mi'],km:[1,'km'],nm:[0.539957,'nm']};
state.units=store.get('units','mi'); state.rings=store.get('rings',false);
const fmtDist=km=>{ const [k,u]=UNITS[state.units]; const v=km*k; return `${v<10?v.toFixed(1):Math.round(v).toLocaleString()} ${u}`; };
const compass16=d=>['N','NNE','NE','ENE','E','ESE','SE','SSE','S','SSW','SW','WSW','W','WNW','NW','NNW'][Math.round(((d%360)+360)%360/22.5)%16];
const llArr=ll=>[ll.lat,ll.lng];
const distKm=(a,b)=>kmBetween({lat:a[0],lon:a[1]},{lat:b[0],lon:b[1]});
map.createPane('tools').style.zIndex=480;
const toolLayer=L.layerGroup().addTo(map);
const tools={
  active:null, measure:{pts:[],group:null,live:null}, track:null, trackGroup:null, trackPick:null, pins:store.get('pins',[]), pinGroup:L.layerGroup().addTo(map),
  set(t){
    if(this.active==='measure') this.finishMeasure();
    if(this.active==='area') this.finishArea();
    if(this.active==='track'&&this.trackPick&&!this.track){ this.clearTrackPick(); }
    this.active=this.active===t?null:t;
    $$('[data-tool]').forEach(b=>b.setAttribute('aria-pressed',b.dataset.tool===this.active));
    map.getContainer().classList.toggle('tool-on',!!this.active);
    if(this.active) map.doubleClickZoom.disable(); else map.doubleClickZoom.enable();
    if(this.active==='measure') this.hint('Click to add points. <b>Double-click</b> or Done to finish.');
    else if(this.active==='track') this.hint(this.track?'Click a storm to start a new track.':'Click the storm\'s leading edge on this frame.');
    else if(this.active==='pin') this.hint('Click to drop a pin showing the radar value there. Click a pin to remove it.');
    else if(this.active&&this.hints&&this.hints[this.active]) this.hint(this.hints[this.active]);
    else this.hint(null);
    if(this.clearTmp){ if(this.active!=='area') this.clearTmp(); }
    if(this.active==='draw') map.dragging.disable(); else map.dragging.enable();
    if(typeof syncToolPanel==='function') syncToolPanel();
  },
  hint(h){ $('#toolHint').hidden=!h; if(h) $('#toolHintTxt').innerHTML=h; },
  onClick(e){
    const ll=llArr(e.latlng);
    if(this.active==='measure') this.addMeasure(ll);
    else if(this.active==='track') this.trackClick(ll);
    else if(this.active==='pin') this.addPin(ll);
    else if(this.active==='draw') return;
    else this.extClick(ll);
  },
  /* ---- measure ---- */
  addMeasure(ll){
    const M=this.measure;
    if(!M.group){ M.group=L.layerGroup().addTo(toolLayer); }
    M.pts.push(ll);
    L.circleMarker(ll,{pane:'tools',radius:4,color:'#000',weight:1.5,fillColor:'#ffb347',fillOpacity:1,interactive:false}).addTo(M.group);
    if(M.pts.length>1){
      const a=M.pts[M.pts.length-2], b=ll, km=distKm(a,b), brg=bearingDeg(a,b);
      L.polyline([a,b],{pane:'tools',color:'#000',weight:5,opacity:.7,interactive:false}).addTo(M.group);
      L.polyline([a,b],{pane:'tools',color:'#ffb347',weight:2.5,interactive:false}).addTo(M.group);
      const mid=[(a[0]+b[0])/2,(a[1]+b[1])/2];
      L.tooltip({permanent:true,direction:'center',className:'tool-label',pane:'tools'}).setLatLng(mid).setContent(`${fmtDist(km)} · ${Math.round(brg)}° ${compass16(brg)}`).addTo(M.group);
    }
    this.updateMeasureTotal();
  },
  updateMeasureTotal(){
    const M=this.measure; if(M.total){ M.group.removeLayer(M.total); M.total=null; }
    if(M.pts.length<3) return;
    let km=0; for(let i=1;i<M.pts.length;i++) km+=distKm(M.pts[i-1],M.pts[i]);
    M.total=L.tooltip({permanent:true,direction:'right',offset:[10,0],className:'tool-label',pane:'tools'}).setLatLng(M.pts[M.pts.length-1]).setContent(`Total ${fmtDist(km)}`).addTo(M.group);
  },
  measureMove(e){
    const M=this.measure; if(this.active!=='measure'||!M.pts.length) return;
    const a=M.pts[M.pts.length-1], b=llArr(e.latlng);
    if(!M.live){ M.live=L.polyline([a,b],{pane:'tools',color:'#ffb347',weight:2,dashArray:'4 5',interactive:false}).addTo(toolLayer);
      M.liveTip=L.tooltip({permanent:true,direction:'right',offset:[12,0],className:'tool-label',pane:'tools'}).addTo(map); }
    M.live.setLatLngs([a,b]); const km=distKm(a,b), brg=bearingDeg(a,b);
    M.liveTip.setLatLng(b).setContent(`${fmtDist(km)} · ${Math.round(brg)}°`);
  },
  finishMeasure(){ const M=this.measure; if(M.live){ toolLayer.removeLayer(M.live); map.removeLayer(M.liveTip); M.live=M.liveTip=null; } if(M.pts.length<2&&M.group){ toolLayer.removeLayer(M.group); } M.pts=[]; M.group=null; M.total=null; },
  /* ---- storm motion / ETA ---- */
  clearTrackPick(){ if(this.trackPick&&this.trackPick.mk) toolLayer.removeLayer(this.trackPick.mk); this.trackPick=null; },
  trackClick(ll){
    const f=curFrame(); if(!f){ toast('Load radar frames first.'); return; }
    if(!this.trackPick||this.track){
      this.clearTrack(); this.trackPick={ll,t:+f.t};
      this.trackPick.mk=L.circleMarker(ll,{pane:'tools',radius:5,color:'#000',weight:2,fillColor:'#5cc8ff',fillOpacity:1,interactive:false}).addTo(toolLayer)
        .bindTooltip(`Start · ${zTime(f.t)}`,{permanent:true,direction:'top',offset:[0,-6],className:'tool-label trk'});
      this.hint(`Start set at <b>${zTime(f.t)}</b>. Step to a later frame (→) and click the same point on the storm.`);
      return;
    }
    const p=this.trackPick, dt=(+f.t-p.t)/60000;
    if(Math.abs(dt)<1){ toast('Pick the second point on a different frame, at least a minute apart.'); return; }
    const [a,b,ta,tb]=dt>0?[p.ll,ll,p.t,+f.t]:[ll,p.ll,+f.t,p.t];
    const km=distKm(a,b), mins=Math.abs(dt), kmh=km/(mins/60);
    if(kmh>200){ toast(`That's ${Math.round(kmh*0.539957)} kt. Check that you clicked the same storm feature.`); }
    this.clearTrackPick();
    this.track={a,b,ta,tb,brg:bearingDeg(a,b),kmh};
    this.drawTrack(); this.hint(`Moving <b>${compass16(this.track.brg)} (${Math.round(this.track.brg)}°) at ${Math.round(kmh*0.621371)} mph</b> · ${Math.round(kmh*0.539957)} kt. Hover the map for ETA.`);
  },
  drawTrack(){
    const T=this.track; if(this.trackGroup) toolLayer.removeLayer(this.trackGroup); if(!T) return;
    const g=this.trackGroup=L.layerGroup().addTo(toolLayer);
    const steps=6, stepMin=10, endKm=T.kmh*steps*stepMin/60, end=destLL(T.b[0],T.b[1],endKm,T.brg);
    const line=(pts,o)=>{ L.polyline(pts,Object.assign({pane:'tools',color:'#000',weight:6,opacity:.75,interactive:false},o.casing||{})).addTo(g); L.polyline(pts,Object.assign({pane:'tools',weight:3,interactive:false},o)).addTo(g); };
    line([T.a,T.b],{color:'#5cc8ff',dashArray:'5 5'});
    line([T.b,end],{color:'#5cc8ff'});
    // cone: widen 2 km per 10 min either side
    const left=[],right=[]; for(let i=0;i<=steps;i++){ const c=destLL(T.b[0],T.b[1],T.kmh*i*stepMin/60,T.brg), w=1+i*2; left.push(destLL(c[0],c[1],w,T.brg-90)); right.unshift(destLL(c[0],c[1],w,T.brg+90)); }
    L.polygon(left.concat(right),{pane:'tools',color:'#5cc8ff',weight:1,opacity:.5,fillColor:'#5cc8ff',fillOpacity:.08,interactive:false}).addTo(g);
    for(const q of [T.a,T.b]) L.circleMarker(q,{pane:'tools',radius:4.5,color:'#000',weight:2,fillColor:'#5cc8ff',fillOpacity:1,interactive:false}).addTo(g);
    for(let i=1;i<=steps;i++){
      const c=destLL(T.b[0],T.b[1],T.kmh*i*stepMin/60,T.brg), w=1+i*2, l=destLL(c[0],c[1],w,T.brg-90), r=destLL(c[0],c[1],w,T.brg+90);
      line([l,r],{color:'#5cc8ff',weight:2,casing:{weight:4.5}});
      const tt=new Date(T.tb+i*stepMin*60e3);
      L.tooltip({permanent:true,direction:'right',offset:[8,0],className:'tool-label trk',pane:'tools'}).setLatLng(r).setContent(tt.toLocaleTimeString([],{hour:'numeric',minute:'2-digit'})).addTo(g);
    }
  },
  eta(ll){
    const T=this.track; if(!T) return null;
    const p=[ll.lat,ll.lng], d=distKm(T.b,p), brg=bearingDeg(T.b,p), dθ=(brg-T.brg)*Math.PI/180;
    const along=d*Math.cos(dθ), cross=Math.abs(d*Math.sin(dθ));
    if(along<-2||cross>Math.max(15,along*.35)) return null;
    const t=T.tb+along/T.kmh*3600e3, inMin=Math.round((t-Date.now())/60000);
    return `ETA ${new Date(t).toLocaleTimeString([],{hour:'numeric',minute:'2-digit'})}${inMin>0?` (${inMin} min)`:inMin>-5?' (now)':' (passed)'}`;
  },
  clearTrack(){ this.clearTrackPick(); if(this.trackGroup){ toolLayer.removeLayer(this.trackGroup); this.trackGroup=null; } this.track=null; },
  /* ---- pins ---- */
  pinLabel(ll){
    const q=probe(ll); let txt=q?q.txt:'—';
    const e=this.eta(L.latLng(ll[0],ll[1])); if(e) txt+=`<br><span style="font-weight:500">${e}</span>`;
    return txt;
  },
  refreshPins(){
    (this.pinMarkers||[]).forEach(({mk,ll})=>{ const q=probe(ll); mk.setStyle({fillColor:valColor(q&&q.d,q?q.v:NaN)}); mk.setTooltipContent(this.pinLabel(ll)); });
    if(this.refreshLoc) this.refreshLoc();
  },
  drawPins(){
    this.pinGroup.clearLayers(); this.pinMarkers=[];
    this.pins.forEach((ll,i)=>{
      const q=probe(ll);
      const mk=L.circleMarker(ll,{pane:'tools',radius:6,color:'#000',weight:2,fillColor:valColor(q&&q.d,q?q.v:NaN),fillOpacity:1,bubblingMouseEvents:false}).addTo(this.pinGroup);
      this.pinMarkers.push({mk,ll});
      mk.bindTooltip(this.pinLabel(ll),{permanent:true,direction:'right',offset:[9,0],className:'tool-label val'});
      mk.on('click',ev=>{ if(this.active!=='pin') return; L.DomEvent.stop(ev); this.pins.splice(i,1); store.set('pins',this.pins); this.drawPins(); });
    });
  },
  addPin(ll){ if(this.pins.length>=50){ toast('Up to 50 pins.'); return; } this.pins.push([+ll[0].toFixed(5),+ll[1].toFixed(5)]); store.set('pins',this.pins); this.drawPins(); },
  clearAll(){ this.finishMeasure(); if(this.clearTmp) this.clearTmp(); if(this.loc){ navigator.geolocation.clearWatch(this.loc.watch); this.loc=null; } this.notes=[]; toolLayer.clearLayers(); this.trackGroup=null; this.track=null; this.trackPick=null; this.pins=[]; store.set('pins',[]); this.drawPins(); if(this.active) this.set(this.active); }
};
/* ---- shared helpers for analysis tools ---- */
function probe(ll){ const f=curFrame(); const d=f&&(f.data||(f.type!=='img'?radar.data:null)); if(!d) return null; const p=sample(d,L.latLng(ll[0],ll[1])); if(!p||p.b<0) return {d,p,v:NaN,txt:p?'out of range':'—'}; const T=decodeTable(d.dec); return {d,p,v:T.rf[p.b]?NaN:T.v[p.b],txt:fmtValue(d,p.b)}; }
function valColor(d,v){ if(!d||isNaN(v)) return '#9aa0a8'; const P_=PAL[d.pal]; if(P_.cat){ const h=hcaClass(v); return h?h[3]:'#9aa0a8'; } const c=palColor(P_,v); return c[3]<.2?'#9aa0a8':`rgb(${c[0]|0},${c[1]|0},${c[2]|0})`; }
function localXY(ll,o){ return [(ll[1]-o[1])*111.32*Math.cos(o[0]*Math.PI/180),(ll[0]-o[0])*110.57]; }
const sqUnit=()=>({mi:[0.386102,'sq mi'],km:[1,'km²'],nm:[0.291553,'sq nm']})[state.units];
const fmtArea=km2=>{ const [k,u]=sqUnit(), v=km2*k; return `${v<10?v.toFixed(1):Math.round(v).toLocaleString()} ${u}`; };
const VCP_ANGLES=[0.5,0.9,1.3,1.8,2.4,3.1,4.0,5.1,6.4,8.0,10.0,12.5,15.6,19.5];
const popup=(ll,html)=>L.popup({autoPanPadding:[40,200],maxWidth:320}).setLatLng(ll).setContent(`<div class="pop">${html}</div>`).openOn(map);
const dl=rows=>`<dl>${rows.map(r=>`<dt>${r[0]}</dt><dd>${r[1]}</dd>`).join('')}</dl>`;
const ftK=km=>{ const ft=km*3280.84; return ft>=10000?(ft/1000).toFixed(1)+'k ft':Math.round(ft/100)*100+' ft'; };
const siteRef=()=>{ const s=state.site; return s?[s.lat,s.lon]:null; };
const fromSite=ll=>{ const o=siteRef(); if(!o) return ''; const km=distKm(o,ll); return `${Math.round(bearingDeg(o,ll))}° · ${fmtDist(km)}`; };

Object.assign(tools,{
  tmp:[], tmpGroup:null, srv:store.get('srv',false), grid:store.get('grid',false), loc:null, notes:[],
  hints:{
    inspect:'Click anywhere for a full readout.',
    area:'Click to outline an area. <b>Double-click</b> to close it.',
    radius:'Click a center, then click the edge.',
    stats:'Click a center, then click the edge to get echo statistics inside.',
    vrot:'Velocity product: click the strongest inbound gate, then the strongest outbound gate.',
    beam:'Click a point to see beam heights for every VCP tilt.',
    radial:'Click a point to draw the radial from the radar.',
    threat:'Click a location to see which storms are heading toward it.',
    draw:'Drag on the map to draw. Map panning is paused while drawing.',
    note:'Click where the note should go.'
  },
  clearTmp(){ if(this.tmpGroup){ toolLayer.removeLayer(this.tmpGroup); this.tmpGroup=null; } this.tmp=[]; },
  tmpG(){ return this.tmpGroup||(this.tmpGroup=L.layerGroup().addTo(toolLayer)); },
  dot(ll,col='#ffb347',g=toolLayer){ return L.circleMarker(ll,{pane:'tools',radius:4,color:'#000',weight:1.5,fillColor:col,fillOpacity:1,interactive:false}).addTo(g); },
  line(pts,col,g=toolLayer,dash){ L.polyline(pts,{pane:'tools',color:'#000',weight:5,opacity:.7,interactive:false}).addTo(g); return L.polyline(pts,{pane:'tools',color:col,weight:2.5,dashArray:dash,interactive:false}).addTo(g); },
  label(ll,html,cls='',dir='right',g=toolLayer){ return L.tooltip({permanent:true,direction:dir,offset:dir==='right'?[8,0]:[0,0],className:'tool-label '+cls,pane:'tools'}).setLatLng(ll).setContent(html).addTo(g); },
  extClick(ll){
    const a=this.active;
    if(a==='inspect'){ const q=probe(ll), f=curFrame(); const rows=[];
      if(q){ rows.push(['Value',`<b>${q.txt}</b>`]); if(q.p&&q.p.b>=0&&q.d.elev!=null&&!q.d.raster) rows.push(['Beam',`${ftK(beamHeightKm(q.p.r,q.d.elev))} · ${q.d.elev.toFixed(1)}°`]); }
      if(siteRef()) rows.push([state.site.id,fromSite(ll)]);
      const e=this.eta(L.latLng(ll[0],ll[1])); if(e) rows.push(['Storm',e]);
      const c=`${ll[0].toFixed(4)}, ${ll[1].toFixed(4)}`; rows.push(['Lat/Lon',`<span id="inspLL">${c}</span> <button class="tbtn" style="width:auto;height:auto;padding:1px 6px;font:600 10px var(--mono);display:inline" id="inspCopy">COPY</button>`]);
      popup(ll,`<span class="tag" style="background:#ffb347;color:#1a1208">INSPECT</span><h4>${famById(state.prod).name}</h4><div class="when">${f?lTime(f.t):''}</div>${dl(rows)}`);
      setTimeout(()=>{ const b=document.getElementById('inspCopy'); if(b) b.onclick=()=>{ navigator.clipboard&&navigator.clipboard.writeText(c).then(()=>b.textContent='COPIED').catch(()=>{ const r=document.createRange(); r.selectNode(document.getElementById('inspLL')); getSelection().removeAllRanges(); getSelection().addRange(r); }); }; },0);
    }
    else if(a==='area'){ this.tmp.push(ll); this.dot(ll,'#ffb347',this.tmpG()); if(this.tmp.length>1) this.line(this.tmp.slice(-2),'#ffb347',this.tmpG()); }
    else if(a==='radius'||a==='stats'){
      if(!this.tmp.length){ this.clearTmp(); this.tmp=[ll]; this.dot(ll,'#ffb347',this.tmpG()); this.hint(a==='radius'?'Now click the edge.':'Now click the edge of the area to analyze.'); return; }
      const c=this.tmp[0], km=distKm(c,ll); this.clearTmp(); if(km<0.2) return;
      const g=L.layerGroup().addTo(toolLayer);
      L.circle(c,{pane:'tools',radius:km*1000,color:'#000',weight:5,opacity:.6,fill:false,interactive:false}).addTo(g);
      L.circle(c,{pane:'tools',radius:km*1000,color:a==='stats'?'#5cc8ff':'#ffb347',weight:2.5,fillOpacity:.06,interactive:false}).addTo(g);
      this.dot(c,a==='stats'?'#5cc8ff':'#ffb347',g);
      if(a==='radius'){ this.line([c,ll],'#ffb347',g,'4 5'); this.label(ll,`r ${fmtDist(km)} · ${fmtArea(Math.PI*km*km)}`); }
      else this.echoStats(c,km,g);
      this.hint(this.hints[a]);
    }
    else if(a==='vrot'){
      const q=probe(ll); if(!q||q.d.pal!=='VEL'){ toast('Switch to a velocity product (VEL or SRV) first.'); return; }
      if(isNaN(q.v)){ toast('No velocity at that point.'); return; }
      this.tmp.push({ll,v:q.v}); this.dot(ll,q.v<0?'#2bd86b':'#ff3b4a',this.tmpG());
      if(this.tmp.length<2){ this.hint(`First gate <b>${q.txt}</b>. Now click the opposite gate.`); return; }
      const [p1,p2]=this.tmp; this.tmp=[]; const grp=this.tmpGroup; this.tmpGroup=null;
      const km=distKm(p1.ll,p2.ll), dv=Math.abs(p1.v-p2.v), opp=Math.sign(p1.v)!==Math.sign(p2.v);
      const vrot=opp?(Math.abs(p1.v)+Math.abs(p2.v))/2:dv/2, shear=(dv/1.943844)/(km*1000);
      const mid=[(p1.ll[0]+p2.ll[0])/2,(p1.ll[1]+p2.ll[1])/2], rng=siteRef()?distKm(siteRef(),mid):null;
      const cat=vrot>=50?'very strong':vrot>=35?'strong':vrot>=20?'moderate':'weak';
      this.line([p1.ll,p2.ll],'#ffd600',grp);
      this.label(mid,`Vrot ${Math.round(vrot)} kt · ${cat}`,'val');
      popup(mid,`<span class="tag" style="background:#ffd600;color:#1a1208">COUPLET</span><h4>Vrot ${Math.round(vrot)} kt</h4><div class="when">${opp?'Inbound/outbound pair':'Same-sign pair (shear only)'}</div>${dl([['Gates',`${p1.v>0?'+':''}${Math.round(p1.v)} / ${p2.v>0?'+':''}${Math.round(p2.v)} kt`],['ΔV',`${Math.round(dv)} kt`],['Separation',fmtDist(km)],['Shear',`${(shear*1000).toFixed(1)} ×10⁻³ s⁻¹`],['Range',rng!=null?fmtDist(rng):'—'],['Rough class',`${cat} (weak &lt;20, moderate 20–34, strong 35–49, very strong ≥50 kt)`]])}`);
      this.hint(this.hints.vrot);
    }
    else if(a==='beam'){
      const o=siteRef(); if(!o){ toast('Pick a radar first.'); return; } const r=distKm(o,ll), bw=0.475;
      const rows=VCP_ANGLES.map(e=>`<tr><td>${e.toFixed(1)}°</td><td>${ftK(beamHeightKm(r,e))}</td><td>${ftK(Math.max(0,beamHeightKm(r,e-bw)))} – ${ftK(beamHeightKm(r,e+bw))}</td></tr>`).join('');
      popup(ll,`<span class="tag" style="background:#ffb347;color:#1a1208">BEAM HEIGHT</span><h4>${fmtDist(r)} from ${state.site.id}</h4><div class="when">Above radar level · 4/3 earth · 0.95° beam</div><table style="width:100%;font:500 11.5px var(--mono);border-collapse:collapse"><tr style="color:var(--mute)"><td>Tilt</td><td>Center</td><td>Bottom – top</td></tr>${rows}</table>`);
    }
    else if(a==='radial'){ const o=siteRef(); if(!o) return; const g=L.layerGroup().addTo(toolLayer); this.line([o,ll],'#c3a6ff',g); this.dot(ll,'#c3a6ff',g);
      const q=probe(ll); const beam=q&&q.d.elev!=null&&!q.d.raster?` · beam ${ftK(beamHeightKm(distKm(o,ll),q.d.elev))}`:''; this.label(ll,`${Math.round(bearingDeg(o,ll))}° · ${fmtDist(distKm(o,ll))}${beam}`,'',undefined,g); }
    else if(a==='threat') this.threatCheck(ll);
    else if(a==='note'){
      this.hint(`<input id="noteIn" placeholder="Note text, then Enter" maxlength="80">`);
      const inp=document.getElementById('noteIn'); inp.focus();
      inp.onkeydown=ev=>{ if(ev.key==='Enter'&&inp.value.trim()){ this.notes.push(this.label(ll,inp.value.trim().replace(/[<>&]/g,c=>({'<':'&lt;','>':'&gt;','&':'&amp;'}[c])),'note','top')); this.hint(this.hints.note); } else if(ev.key==='Escape') this.hint(this.hints.note); ev.stopPropagation(); };
    }
  },
  finishArea(){
    if(this.tmp.length>=3){ const o=this.tmp[0], xy=this.tmp.map(p=>localXY(p,o)); let a2=0,per=0;
      for(let i=0;i<xy.length;i++){ const j=(i+1)%xy.length; a2+=xy[i][0]*xy[j][1]-xy[j][0]*xy[i][1]; per+=distKm(this.tmp[i],this.tmp[j]); }
      const g=L.layerGroup().addTo(toolLayer);
      L.polygon(this.tmp,{pane:'tools',color:'#ffb347',weight:2.5,fillOpacity:.1,interactive:false}).addTo(g);
      const c=this.tmp.reduce((s,p)=>[s[0]+p[0]/this.tmp.length,s[1]+p[1]/this.tmp.length],[0,0]);
      this.label(c,`${fmtArea(Math.abs(a2)/2)} · perimeter ${fmtDist(per)}`,'','center',g);
    }
    this.clearTmp();
  },
  echoStats(c,km,g){
    const f=curFrame(), d=f&&(f.data||radar.data); if(!d){ toast('No radar data in this frame.'); return; }
    const T=decodeTable(d.dec), step=Math.max(km/90,d.gateKm,0.25), kx=111.32*Math.cos(c[0]*Math.PI/180);
    let n=0,valid=0,sum=0,mx=-Infinity,mn=Infinity,mxAt=null; const vel=d.pal==='VEL'; const th=vel?[]:(d.pal==='REF'?[30,40,50,60]:[]); const cnt=th.map(()=>0);
    for(let y=-km;y<=km;y+=step) for(let x=-km;x<=km;x+=step){ if(x*x+y*y>km*km) continue; n++;
      const ll=L.latLng(c[0]+y/110.57,c[1]+x/kx), p=sample(d,ll); if(!p||p.b<=0||T.rf[p.b]) continue; const v=T.v[p.b]; if(isNaN(v)) continue;
      valid++; sum+=v; if(v>mx){mx=v;mxAt=[ll.lat,ll.lng];} if(v<mn) mn=v; th.forEach((t,i)=>{ if(v>=t) cnt[i]++; }); }
    const P_=PAL[d.pal], fmt=v=>P_.cat?(hcaClass(v)||[0,0,'—'])[2]:P_.fmt(v), A=Math.PI*km*km;
    const rows=[['Area',fmtArea(A)],['Echo coverage',`${n?Math.round(valid/n*100):0}%`]];
    if(valid){ if(vel){ rows.push(['Max outbound',fmt(mx)],['Max inbound',fmt(mn)]); } else rows.push(['Max',`<b>${fmt(mx)}</b>`],['Mean',P_.cat?'—':fmt(sum/valid)]);
      th.forEach((t,i)=>rows.push([`≥ ${t} dBZ`,`${fmtArea(A*cnt[i]/n)} (${Math.round(cnt[i]/n*100)}%)`])); }
    if(mxAt&&!vel) this.dot(mxAt,'#ff3b4a',g);
    popup(c,`<span class="tag" style="background:#5cc8ff;color:#06141d">ECHO STATS</span><h4>${famById(state.prod).name}</h4><div class="when">${lTime(f.t)} · radius ${fmtDist(km)}</div>${dl(rows)}`);
  },
  threatCheck(ll){
    const f=curFrame(), cells=f&&cellsFor(f); this.dot(ll,'#fff',this.tmpG());
    const out=[];
    for(const s of (cells&&cells.storms)||[]){ if(s.dir==null||!s.kt) continue;
      const d=distKm(s.pos,ll); if(d>250) continue; const to=(s.dir+180)%360, dθ=(bearingDeg(s.pos,ll)-to)*Math.PI/180;
      const along=d*Math.cos(dθ), cross=Math.abs(d*Math.sin(dθ)), kmh=s.kt*1.852;
      if(along<=0||cross>Math.max(12,along*.25)) continue;
      const t=+((cells.t)||+f.t)+along/kmh*3600e3; const tvs=(cells.tvs||[]).some(v=>distKm(v.pos,s.pos)<8);
      out.push({s,d,t,cross,tvs}); }
    out.sort((a,b)=>a.t-b.t);
    const rows=out.slice(0,8).map(o=>`<tr><td><b>${o.s.id}</b>${o.tvs?' <span style="color:#ff3b4a">TVS</span>':''}</td><td>${new Date(o.t).toLocaleTimeString([],{hour:'numeric',minute:'2-digit'})}</td><td>${fmtDist(o.d)}</td><td>${fmtDist(o.cross)} off</td><td>${o.s.dbzm!=null?o.s.dbzm+' dBZ':''}</td></tr>`).join('');
    popup(ll,`<span class="tag" style="background:#ff3b4a;color:#fff">THREAT CHECK</span><h4>${out.length?`${out.length} storm${out.length>1?'s':''} approaching`:'No storms heading here'}</h4><div class="when">SCIT motion · ${f?lTime(f.t):''}</div>${out.length?`<table style="width:100%;font:500 11.5px var(--mono);border-collapse:collapse"><tr style="color:var(--mute)"><td>Cell</td><td>Arrives</td><td>Now</td><td>Track</td><td>Max</td></tr>${rows}</table>`:'<div style="color:var(--mute);font-size:12px">'+(cells&&cells.storms.length?'Tracked cells are not moving toward this point.':'No storm tracks loaded for this radar.')+'</div>'}`);
  },
  /* freehand draw */
  initDraw(){
    const el=map.getContainer(); let pts=null, ln=null;
    el.addEventListener('pointerdown',ev=>{ if(this.active!=='draw'||ev.button>0) return; if(ev.target.closest('.panel')) return; pts=[]; ln=L.polyline([],{pane:'tools',color:'#fff',weight:2.5,interactive:false}).addTo(toolLayer); el.setPointerCapture(ev.pointerId); ev.preventDefault(); });
    el.addEventListener('pointermove',ev=>{ if(!pts) return; const r=el.getBoundingClientRect(); const ll=map.containerPointToLatLng([ev.clientX-r.left,ev.clientY-r.top]); pts.push(ll); ln.setLatLngs(pts); });
    const end=()=>{ if(!pts) return; if(pts.length<2) toolLayer.removeLayer(ln); else { const c=L.polyline(pts,{pane:'tools',color:'#000',weight:5,opacity:.6,interactive:false}); toolLayer.addLayer(c); c.bringToBack&&0; ln.bringToFront(); } pts=null; ln=null; };
    el.addEventListener('pointerup',end); el.addEventListener('pointercancel',end);
  },
  /* my location */
  toggleLocation(){
    if(this.loc){ navigator.geolocation.clearWatch(this.loc.watch); toolLayer.removeLayer(this.loc.g); this.loc=null; setToolPressed('location',false); return; }
    if(!navigator.geolocation){ toast('Location is not available in this browser.'); return; }
    const g=L.layerGroup().addTo(toolLayer); this.loc={g,first:true}; setToolPressed('location',true); toast('Finding your location…',2500);
    this.loc.watch=navigator.geolocation.watchPosition(p=>{ if(!this.loc) return; const ll=[p.coords.latitude,p.coords.longitude];
      if(!this.loc.mk){ this.loc.acc=L.circle(ll,{pane:'tools',radius:p.coords.accuracy,color:'#4ea3ff',weight:1,fillOpacity:.1,interactive:false}).addTo(g);
        this.loc.mk=L.circleMarker(ll,{pane:'tools',radius:7,color:'#fff',weight:3,fillColor:'#2f86ff',fillOpacity:1,interactive:false}).addTo(g);
        this.loc.tip=L.tooltip({permanent:true,direction:'right',offset:[10,0],className:'tool-label pin',pane:'tools'}).setLatLng(ll).addTo(g); }
      else { this.loc.mk.setLatLng(ll); this.loc.acc.setLatLng(ll).setRadius(p.coords.accuracy); this.loc.tip.setLatLng(ll); }
      this.loc.ll=ll; this.refreshLoc(); if(this.loc.first){ this.loc.first=false; map.flyTo(ll,Math.max(map.getZoom(),9),{duration:.6}); } },
      err=>{ toast(err.code===1?'Location permission was denied.':'Could not get your location.'); if(this.loc){ toolLayer.removeLayer(this.loc.g); this.loc=null; setToolPressed('location',false); } },{enableHighAccuracy:true,maximumAge:15000});
  },
  refreshLoc(){ if(!this.loc||!this.loc.ll) return; const q=probe(this.loc.ll), e=this.eta(L.latLng(this.loc.ll[0],this.loc.ll[1]));
    this.loc.tip.setContent(`You · ${q?q.txt:'—'}${siteRef()?`<br>${state.site.id} ${fromSite(this.loc.ll)}`:''}${e?`<br>${e}`:''}`); },
  /* lat/lon grid (drawn in the overlay) */
  drawGrid(x,m,W,H){
    const b=m.getBounds(), span=Math.max(b.getEast()-b.getWest(),b.getNorth()-b.getSouth()), st=[20,10,5,2,1,.5,.25,.1].find(v=>span/v>=4&&span/v<=14)||.1;
    x.save(); x.strokeStyle='rgba(255,236,210,.16)'; x.lineWidth=1; x.font='500 10px "IBM Plex Mono",monospace'; x.fillStyle='rgba(255,236,210,.65)';
    const fmtD=(v,p,n)=>`${Math.abs(v).toFixed(st<1?(st<.5?2:1):0)}°${v>=0?p:n}`;
    for(let lo=Math.ceil(b.getWest()/st)*st;lo<=b.getEast();lo+=st){ const p=m.latLngToContainerPoint([b.getCenter().lat,lo]); x.beginPath(); x.moveTo(p.x,0); x.lineTo(p.x,H); x.stroke(); x.fillText(fmtD(lo,'E','W'),p.x+3,H-170); }
    for(let la=Math.ceil(b.getSouth()/st)*st;la<=b.getNorth();la+=st){ const p=m.latLngToContainerPoint([la,b.getCenter().lng]); x.beginPath(); x.moveTo(0,p.y); x.lineTo(W,p.y); x.stroke(); x.fillText(fmtD(la,'N','S'),W-70,p.y-3); }
    x.restore();
  },
  /* SRV from the drawn storm track */
  srvText(d,b,ll){ const T=this.track; if(!this.srv||!T||!d||d.pal!=='VEL'||!siteRef()) return '';
    const tbl=decodeTable(d.dec); if(tbl.rf[b]||isNaN(tbl.v[b])) return ''; const az=bearingDeg(siteRef(),[ll.lat,ll.lng]);
    const comp=T.kmh/1.852*Math.cos((T.brg-az)*Math.PI/180), srv=tbl.v[b]-comp; return ` · SRV ${srv>0?'+':''}${Math.round(srv)} kt`; }
});
function setToolPressed(id,on){ const b=document.querySelector(`#toolsPanel [data-x="${id}"]`); if(b) b.setAttribute('aria-pressed',!!on); }

/* ---- side panels: cells, warnings, bookmarks, shortcuts ---- */
const side={ kind:null,
  open(k){ if(this.kind===k){ this.close(); return; } this.kind=k; $('#sidePanel').hidden=false; ['cells','warnings','bookmarks','help'].forEach(x=>setToolPressed(x,x===k)); this.render(); },
  close(){ const k=this.kind; this.kind=null; $('#sidePanel').hidden=true; if(k) setToolPressed(k,false); },
  render(){
    const k=this.kind; if(!k) return; const body=$('#sideBody');
    if(k==='cells'){ $('#sideTitle').textContent='Storm cells'; const f=curFrame(), c=f&&cellsFor(f), o=siteRef();
      const list=((c&&c.storms)||[]).map(s=>({s,tvs:(c.tvs||[]).some(v=>distKm(v.pos,s.pos)<8),meso:(c.meso||[]).some(z=>distKm(z.pos,s.pos)<8),hail:(c.hail||[]).filter(h=>distKm(h.pos,s.pos)<6).sort((a,b)=>b.pSvr-a.pSvr)[0]})).sort((a,b)=>(b.tvs-a.tvs)||((b.s.dbzm||0)-(a.s.dbzm||0)));
      body.innerHTML=list.length?`<table><tr><th>ID</th><th>Max</th><th>Motion</th><th>Hail</th><th>From ${o?state.site.id:''}</th></tr>${list.map(({s,tvs,meso,hail})=>`<tr data-ll="${s.pos[0]},${s.pos[1]}"><td><b>${s.id}</b>${tvs?' <span style="color:#ff3b4a">TVS</span>':meso?' <span style="color:#ffd600">MESO</span>':''}</td><td>${s.dbzm!=null?s.dbzm+' dBZ':'—'}</td><td>${s.dir!=null?`${compass16(s.dir+180)} ${Math.round(s.kt*1.150779)} mph`:'new'}</td><td>${hail?`${hail.pHail}/${hail.pSvr}%`:'—'}</td><td>${o?fromSite(s.pos):''}</td></tr>`).join('')}</table>`:`<div class="empty">${c?'No storm cells in the latest SCIT product.':'No storm tracking data for this radar or frame.'}</div>`; }
    else if(k==='warnings'){ $('#sideTitle').textContent='Active warnings'; const act=(curAlerts||[]).slice().sort((a,b)=>(['TO','SV','FF','EW','SQ','MA','DS','SPS'].indexOf(a.ph))-(['TO','SV','FF','EW','SQ','MA','DS','SPS'].indexOf(b.ph)));
      body.innerHTML=act.length?act.map((a,i)=>{ const P=a.f.properties; return `<div class="row" data-i="${i}"><span class="sw" style="background:${alertColor(a)}"></span><div class="m"><div><b>${P.event}</b></div><div><small>${(P.areaDesc||'').split(';').slice(0,3).join(', ')}</small></div></div><small>until ${new Date(Date.parse(P.ends||P.expires)).toLocaleTimeString([],{hour:'numeric',minute:'2-digit'})}</small></div>`; }).join(''):'<div class="empty">No warnings in effect at this frame time.</div>';
      body.onclick=e=>{ const r=e.target.closest('.row'); if(!r) return; const a=act[+r.dataset.i]; try{ map.fitBounds(L.geoJSON(a.f).getBounds(),{padding:[60,60],maxZoom:10}); }catch(_){} }; return; }
    else if(k==='bookmarks'){ $('#sideTitle').textContent='Bookmarks'; const bm=store.get('bookmarks',[]);
      body.innerHTML=`<div class="act"><input id="bmName" placeholder="Name (optional)" maxlength="40"><button id="bmSave">Save view</button></div>`+(bm.length?bm.map((b,i)=>`<div class="row" data-i="${i}"><div class="m"><div><b>${b.name.replace(/[<>&]/g,'')}</b></div><div><small>${b.site} · ${famById(b.prod).label} · z${b.z.toFixed(1)}</small></div></div><button class="del" data-del="${i}" title="Delete">✕</button></div>`).join(''):'<div class="empty">Save the current radar, product and map view to come back to it later.</div>');
      $('#bmSave').onclick=()=>{ const c=map.getCenter(); bm.unshift({name:$('#bmName').value.trim()||`${state.site?state.site.id:''} ${new Date().toLocaleString([],{month:'short',day:'numeric',hour:'numeric',minute:'2-digit'})}`,site:state.site&&state.site.id,prod:state.prod,tilt:state.tilt,lat:c.lat,lon:c.lng,z:map.getZoom()}); store.set('bookmarks',bm.slice(0,30)); this.render(); };
      body.onclick=e=>{ const d=e.target.closest('[data-del]'); if(d){ bm.splice(+d.dataset.del,1); store.set('bookmarks',bm); this.render(); return; } const r=e.target.closest('.row'); if(!r) return; const b=bm[+r.dataset.i];
        if(b.site&&(!state.site||state.site.id!==b.site)){ state.prod=b.prod; selectSite(b.site,false); } else if(b.prod!==state.prod) pickFamily(b.prod);
        map.setView([b.lat,b.lon],b.z); }; return; }
    else if(k==='help'){ $('#sideTitle').textContent='Shortcuts'; const K=[['Space','Play / pause'],['← →','Step frames'],['Home / End','First / last frame'],['S','Change radar'],['V','Reflectivity ⇄ velocity'],['[ ]','Previous / next product'],['M','Measure'],['T','Storm motion / ETA'],['P','Pin'],['R','Range rings'],['I','Inspect'],['A','Area'],['O','Radius'],['E','Echo stats'],['K','Velocity couplet (Vrot)'],['H','Beam heights'],['L','Radial line'],['X','Threat check'],['Q','SRV readout'],['G','My location'],['/','Search place'],['C','Storm cells'],['W','Warnings'],['B','Bookmarks'],['D','Draw'],['N','Note'],['U','Lat/lon grid'],['Z','Snapshot'],['F','Fullscreen'],['J','Jump to time'],['2','Split screen'],['Y','Point history'],['\\','Cross section (Level II)'],['Shift+V','Record loop video'],['.','Copy view link'],['?','This list'],['Esc','Stop the current tool']];
      body.innerHTML=`<table>${K.map(k=>`<tr><td style="width:90px"><kbd>${k[0]}</kbd></td><td style="font-family:var(--ui)">${k[1]}</td></tr>`).join('')}</table>`; }
    body.onclick=e=>{ const r=e.target.closest('tr[data-ll]'); if(r){ const [a,b]=r.dataset.ll.split(',').map(Number); map.panTo([a,b]); } };
  }
};
$('#sideClose').onclick=()=>side.close();

/* ---- search / go to ---- */
function openSearch(){
  tools.hint(`<input id="searchIn" placeholder="Place, address or lat, lon" autocomplete="off"><span id="searchRes"></span>`);
  const inp=document.getElementById('searchIn'); inp.focus();
  inp.onkeydown=async ev=>{ ev.stopPropagation(); if(ev.key==='Escape'){ tools.hint(null); return; } if(ev.key!=='Enter') return;
    const q=inp.value.trim(); const m=q.match(/^\s*(-?\d+(?:\.\d+)?)\s*[, ]\s*(-?\d+(?:\.\d+)?)\s*$/);
    const go=(ll,name)=>{ map.flyTo(ll,Math.max(map.getZoom(),10),{duration:.7}); const g=L.layerGroup().addTo(toolLayer); tools.dot(ll,'#fff',g); tools.label(ll,name.replace(/[<>&]/g,''),'pin','right',g); tools.hint(null); };
    if(m){ go([+m[1],+m[2]],`${(+m[1]).toFixed(4)}, ${(+m[2]).toFixed(4)}`); return; }
    $('#searchRes').textContent='Searching…';
    try{ const r=await fetch(`https://nominatim.openstreetmap.org/search?format=jsonv2&limit=1&q=${encodeURIComponent(q)}`,{headers:{'Accept-Language':navigator.language||'en'}}); const j=await r.json();
      if(!j.length){ $('#searchRes').textContent='No match'; return; } go([+j[0].lat,+j[0].lon],j[0].display_name.split(',').slice(0,2).join(',')); }
    catch(e){ $('#searchRes').textContent='Search unavailable'; } };
}

/* ---- jump to time ---- */
function openJump(){
  if(!state.frames.length){ toast('No frames loaded.'); return; }
  const f=curFrame(), t=new Date(f.t), v=`${String(t.getHours()).padStart(2,'0')}:${String(t.getMinutes()).padStart(2,'0')}`;
  tools.hint(`Jump to <input id="jumpIn" type="time" value="${v}" style="width:120px">`); const inp=document.getElementById('jumpIn'); inp.focus();
  const go=()=>{ const [h,mn]=inp.value.split(':').map(Number); if(isNaN(h)) return; let best=0,bd=Infinity;
    state.frames.forEach((fr,i)=>{ const d=new Date(fr.t), want=new Date(d); want.setHours(h,mn,0,0); let dd=Math.abs(d-want); if(dd>12*3600e3) dd=86400e3-dd; if(dd<bd){bd=dd;best=i;} });
    setPlaying(false); showFrame(best); };
  inp.onchange=go; inp.onkeydown=ev=>{ ev.stopPropagation(); if(ev.key==='Enter'){ go(); tools.hint(null); } if(ev.key==='Escape') tools.hint(null); };
}

/* ---- snapshot (PNG) ---- */
function mapPanes(el){ return [...el.querySelectorAll('.leaflet-pane')].filter(p=>!p.classList.contains('leaflet-map-pane')&&!/marker|shadow|tooltip|popup/.test(p.className)).sort((a,b)=>(+getComputedStyle(a).zIndex||0)-(+getComputedStyle(b).zIndex||0)); }
async function svgSnapshots(el){ const out=new Map();
  await Promise.all([...el.querySelectorAll('svg.leaflet-zoom-animated')].map(svg=>new Promise(res=>{ const c=svg.cloneNode(true); c.setAttribute('xmlns','http://www.w3.org/2000/svg'); const r=svg.getBoundingClientRect(); c.setAttribute('width',r.width); c.setAttribute('height',r.height);
    const im=new Image(); im.onload=()=>{ out.set(svg,{im,r}); res(); }; im.onerror=res; im.src='data:image/svg+xml;charset=utf-8,'+encodeURIComponent(new XMLSerializer().serializeToString(c)); })));
  return out; }
// draws the live map (tiles, radar, overlays, SVG vectors) into a 2D context sized to the map container
function drawMapInto(x,withTiles,svgImgs){
  const el=map.getContainer(), R0=el.getBoundingClientRect();
  x.fillStyle=getComputedStyle(el).backgroundColor||'#0b0d10'; x.fillRect(0,0,R0.width,R0.height);
  radar.drawNow();
  for(const p of mapPanes(el)){ const clip=p===imgPane&&isMrrl(state.site);
    if(clip){ const s=state.site, c=map.latLngToContainerPoint([s.lat,s.lon]), e=map.latLngToContainerPoint(destLL(s.lat,s.lon,s.rangeKm,90)); x.save(); x.beginPath(); x.arc(c.x,c.y,Math.abs(e.x-c.x),0,7); x.clip(); }
    for(const n of p.querySelectorAll('img.leaflet-tile-loaded,img.leaflet-image-layer,canvas,svg')){
      if(n.tagName==='IMG'&&!withTiles) continue; const st=getComputedStyle(n); if(st.visibility==='hidden'||st.display==='none') continue;
      const lay=n.closest('.leaflet-layer'), op=+(lay?getComputedStyle(lay).opacity:1)*+st.opacity; if(op<=0) continue;
      x.globalAlpha=op;
      if(n.tagName==='svg'){ const sv=svgImgs&&svgImgs.get(n); if(sv) x.drawImage(sv.im,sv.r.left-R0.left,sv.r.top-R0.top,sv.r.width,sv.r.height); }
      else { const r=n.getBoundingClientRect(); if(r.width&&r.height) try{ x.drawImage(n,r.left-R0.left,r.top-R0.top,r.width,r.height); }catch(e){} }
    }
    x.globalAlpha=1; if(clip) x.restore(); }
  const f=curFrame(), txt=`${state.site?state.site.id:''} · ${famById(state.prod).name} · ${f?lTime(f.t)+' ('+zTime(f.t)+')':''}`;
  x.font='600 13px "IBM Plex Mono",monospace'; const w=x.measureText(txt).width+20; x.fillStyle='rgba(8,10,14,.85)'; x.fillRect(10,R0.height-34,w,24); x.fillStyle='#ffb347'; x.fillText(txt,20,R0.height-17);
}
function mapCanvas(withTiles,svgImgs,dpr){ const R0=map.getContainer().getBoundingClientRect(); dpr=dpr||Math.min(window.devicePixelRatio||1,2);
  const cv=document.createElement('canvas'); cv.width=Math.round(R0.width*dpr); cv.height=Math.round(R0.height*dpr); const x=cv.getContext('2d'); x.scale(dpr,dpr); drawMapInto(x,withTiles,svgImgs); return cv; }
function tilesAllowed(svgImgs){ try{ mapCanvas(true,svgImgs,0.25).toDataURL(); return true; }catch(e){ return false; } }
function download(href,name){ const a=document.createElement('a'); a.download=name; a.href=href; document.body.appendChild(a); a.click(); a.remove(); }
async function snapshot(){
  const svg=await svgSnapshots(map.getContainer()), ok=tilesAllowed(svg), f=curFrame();
  download(mapCanvas(ok,svg).toDataURL('image/png'),`${state.site?state.site.id:'radar'}_${state.prod}_${f?new Date(f.t).toISOString().slice(0,16).replace(/[:T-]/g,''):'now'}.png`);
  toast('Snapshot saved'+(ok?'':' (basemap left out: tile server does not allow it)'),3500);
}

/* ---- loop video (WebM/MP4 via MediaRecorder) ---- */
const rec={busy:false,cancel:false};
async function recordLoop(){
  if(rec.busy){ rec.cancel=true; return; }
  if(!window.MediaRecorder){ toast('Video recording is not supported in this browser.'); return; }
  const frames=state.frames.map((f,i)=>i).filter(i=>state.frames[i].status==='ok'||state.frames[i].type==='img');
  if(frames.length<2){ toast('Load at least two frames first.'); return; }
  const mime=['video/webm;codecs=vp9','video/webm;codecs=vp8','video/webm','video/mp4'].find(m=>MediaRecorder.isTypeSupported(m));
  if(!mime){ toast('No supported video format in this browser.'); return; }
  rec.busy=true; rec.cancel=false; setPlaying(false); const keep=state.idx;
  const R0=map.getContainer().getBoundingClientRect(), dpr=1.5, cv=document.createElement('canvas'); cv.width=Math.round(R0.width*dpr)&~1; cv.height=Math.round(R0.height*dpr)&~1;
  const x=cv.getContext('2d'); x.scale(dpr,dpr);
  const svg0=await svgSnapshots(map.getContainer()), withTiles=tilesAllowed(svg0);
  const stream=cv.captureStream(0), track=stream.getVideoTracks()[0], chunks=[];
  const mr=new MediaRecorder(stream,{mimeType:mime,videoBitsPerSecond:8e6}); mr.ondataavailable=e=>e.data.size&&chunks.push(e.data);
  const stopped=new Promise(r=>mr.onstop=r); mr.start();
  const per=Math.max(60,480/SPEEDS[state.speedIdx]), sleep=ms=>new Promise(r=>setTimeout(r,ms)), raf=()=>new Promise(r=>requestAnimationFrame(()=>requestAnimationFrame(r)));
  try{
    for(let k=0;k<frames.length&&!rec.cancel;k++){
      showFrame(frames[k]); tools.hint(`Recording frame <b>${k+1}/${frames.length}</b> <button id="recStop">STOP</button>`); const b=document.getElementById('recStop'); if(b) b.onclick=()=>rec.cancel=true;
      await raf(); const svg=await svgSnapshots(map.getContainer()); drawMapInto(x,withTiles,svg); track.requestFrame&&track.requestFrame();
      await sleep(k===frames.length-1?Math.max(per,state.dwell*1000):per);
    }
  } finally {
    mr.stop(); await stopped; rec.busy=false; tools.hint(null); showFrame(keep);
  }
  if(rec.cancel&&!chunks.length) return;
  const blob=new Blob(chunks,{type:mime.split(';')[0]}), ext=mime.includes('mp4')?'mp4':'webm';
  download(URL.createObjectURL(blob),`${state.site?state.site.id:'radar'}_${state.prod}_loop.${ext}`);
  toast(`Loop video saved (${(blob.size/1048576).toFixed(1)} MB)${withTiles?'':', basemap left out'}`,4000);
}

/* ---- tools panel ---- */
const TOOL_DEFS=[
  ['Measure',[['measure','Distance','M','mode'],['area','Area','A','mode'],['radius','Radius','O','mode'],['radial','Radial line','L','mode']]],
  ['Analyze',[['inspect','Inspect','I','mode'],['stats','Echo stats','E','mode'],['vrot','Vrot couplet','K','mode'],['beam','Beam heights','H','mode'],['srv','SRV readout','Q','toggle'],['threat','Threat check','X','mode'],['track','Storm ETA','T','mode']]],
  ['Navigate',[['location','My location','G','toggle'],['search','Search place','/','action'],['bookmarks','Bookmarks','B','panel'],['jump','Jump to time','J','action']]],
  ['Annotate',[['pin','Value pin','P','mode'],['draw','Draw','D','mode'],['note','Note','N','mode']]],
  ['Panels',[['cells','Storm cells','C','panel'],['warnings','Warnings','W','panel']]],
  ['View',[['rings','Range rings','R','toggle'],['grid','Lat/lon grid','U','toggle'],['snapshot','Snapshot','Z','action'],['fullscreen','Fullscreen','F','action'],['help','Shortcuts','?','panel']]]
];
$('#toolsPanel').innerHTML=TOOL_DEFS.map(([g,items])=>`<h3>${g}</h3><div class="tg">${items.map(([id,lab,k])=>`<button data-x="${id}" aria-pressed="false"><span>${lab}</span><kbd>${k}</kbd></button>`).join('')}</div>`).join('');
function runTool(id){
  const def=TOOL_DEFS.flatMap(g=>g[1]).find(d=>d[0]===id); if(!def) return; const kind=def[3];
  if(kind==='mode'){ tools.set(id); syncToolPanel(); }
  else if(kind==='panel') side.open(id);
  else if(id==='srv'){ tools.srv=!tools.srv; store.set('srv',tools.srv); setToolPressed('srv',tools.srv); if(tools.srv&&!tools.track) toast('Draw a storm motion track (T) first; SRV uses its motion.',4500); updateReadout(); }
  else if(id==='location') tools.toggleLocation();
  else if(id==='rings') $('#ringsBtn').click();
  else if(id==='grid'){ tools.grid=!tools.grid; store.set('grid',tools.grid); setToolPressed('grid',tools.grid); overlay.redraw(); }
  else if(id==='search') openSearch();
  else if(id==='jump') openJump();
  else if(id==='snapshot') snapshot();
  else if(id==='fullscreen'){ const d=document; (d.fullscreenElement?d.exitFullscreen():d.documentElement.requestFullscreen()).catch(()=>toast('Fullscreen is not available here.')); }
}
function syncToolPanel(){ TOOL_DEFS.flatMap(g=>g[1]).forEach(([id,,,k])=>{ if(k==='mode') setToolPressed(id,tools.active===id); }); setToolPressed('rings',state.rings); setToolPressed('grid',tools.grid); setToolPressed('srv',tools.srv); setToolPressed('location',!!tools.loc); }
$('#toolsPanel').onclick=e=>{ const b=e.target.closest('[data-x]'); if(b) runTool(b.dataset.x); };
$('#allToolsBtn').onclick=()=>{ const p=$('#toolsPanel'); p.hidden=!p.hidden; $('#allToolsBtn').setAttribute('aria-pressed',!p.hidden); if(!p.hidden){ $('#layers').hidden=true; $('#layersBtn').setAttribute('aria-expanded',false); syncToolPanel(); } };
tools.initDraw();

// 4/3-earth beam height (km) for slant range r (km) at elevation e (deg), antenna height h0 (km)
function beamHeightKm(r,e,h0=0){ const ke=4/3*6371, t=e*Math.PI/180; return Math.sqrt(r*r+ke*ke+2*r*ke*Math.sin(t))-ke+h0; }
$$('[data-tool]').forEach(b=>b.onclick=()=>tools.set(b.dataset.tool));
$('#toolDone').onclick=()=>{ if(tools.active) tools.set(tools.active); };
$('#clearTools').onclick=()=>tools.clearAll();
$('#ringsBtn').setAttribute('aria-pressed',state.rings);
$('#ringsBtn').onclick=()=>{ state.rings=!state.rings; store.set('rings',state.rings); $('#ringsBtn').setAttribute('aria-pressed',state.rings); overlay.redraw(); };
$('#units').value=state.units;
$('#units').onchange=e=>{ state.units=e.target.value; store.set('units',state.units); overlay.redraw(); tools.drawPins(); tools.drawTrack(); updateReadout(); };
map.on('mousemove',e=>tools.measureMove(e));
map.on('dblclick',e=>{ if(tools.active==='area'){ L.DomEvent.stop(e); tools.finishArea(); tools.hint('Area saved. Click to outline another, or Done.'); return; } if(tools.active==='measure'){ L.DomEvent.stop(e); tools.finishMeasure(); tools.hint('Measurement saved. Click to start another, or Done.'); } });
tools.drawPins();

/* ================= storm reports: IEM LSR + SpotterNetwork ================= */
const RPT_WIN=[15,30,60,120,240,360,720,1440];
state.lsr=store.get('lsr',false); state.sn=store.get('sn',false); state.rptWinIdx=store.get('rptWinIdx',3); state.wind=store.get('wind',false);
const reports={lsr:[],sn:[],lsrAt:0,snAt:0,snBlocked:false,snWarned:false};
// [letter, fill, stroke, text color]
function rptCat(t){ t=String(t||'').toUpperCase();
  if(/TORNADO|WATERSPOUT/.test(t)) return ['T','#ff2e3e','#000','#fff'];
  if(/FUNNEL|WALL CLOUD|ROTAT/.test(t)) return ['F','#ff9f1c','#000','#1a1208'];
  if(/HAIL/.test(t)) return ['H','#2bd86b','#000','#062010'];
  if(/DMG|DAMAGE/.test(t)) return ['D','#2f6bd8','#000','#fff'];
  if(/WND|WIND|GUST/.test(t)) return ['G','#4ea3ff','#000','#06141d'];
  if(/FLOOD|RAIN|WATER/.test(t)) return ['W','#10b39a','#000','#04201b'];
  if(/SNOW|BLIZZARD|ICE|SLEET|FREEZ/.test(t)) return ['S','#9ad7ff','#000','#06141d'];
  if(/LIGHTNING/.test(t)) return ['L','#ffd23f','#000','#1a1208'];
  return ['•','#9aa0a8','#000','#0b0b0b']; }
async function loadReports(force){
  const now=Date.now(), first=state.frames.length?+state.frames[0].t:now-3600e3, winMs=RPT_WIN[state.rptWinIdx]*60e3;
  const sts=new Date(Math.max(first-winMs,now-72*3600e3)).toISOString().slice(0,19)+'Z', ets=new Date(now+5*60e3).toISOString().slice(0,19)+'Z';
  const needLsr=state.lsr||(state.sn&&reports.snBlocked);
  if(needLsr&&(force||now-reports.lsrAt>60e3)){ reports.lsrAt=now;
    try{ const r=await fetch(`https://mesonet.agron.iastate.edu/geojson/lsr.py?sts=${sts}&ets=${ets}`); if(!r.ok) throw new Error('HTTP '+r.status); const j=await r.json();
      reports.lsr=(j.features||[]).filter(f=>f.geometry&&f.geometry.coordinates).map(f=>{ const p=f.properties||{}, v=String(p.valid||p.utc_valid||'');
        return {lon:+f.geometry.coordinates[0],lat:+f.geometry.coordinates[1],t:Date.parse(/Z$|[+-]\d\d:?\d\d$/.test(v)?v:v+'Z'),type:p.typetext||p.type||'Report',mag:p.magnitude,unit:p.unit,city:p.city,county:p.county,st:p.state||p.st,src:p.source,remark:p.remark,wfo:p.wfo}; }).filter(r=>!isNaN(r.t));
    }catch(e){ console.warn('LSR',e); if(force) toast('Storm reports (IEM) could not be loaded.'); }
  }
  if(state.sn&&!reports.snBlocked&&(force||now-reports.snAt>60e3)){ reports.snAt=now;
    try{ const r=await fetch('https://www.spotternetwork.org/feeds/reports.txt'); if(!r.ok) throw new Error('HTTP '+r.status); reports.sn=parseSNReports(await r.text()); }
    catch(e){ console.warn('SpotterNetwork',e); reports.snBlocked=true; $('#snSrc').textContent='spotter LSRs';
      if(!reports.snWarned){ reports.snWarned=true; toast('SpotterNetwork blocks browser requests, so spotter and chaser reports come from NWS LSRs instead.',6000); }
      return loadReports(true); }
  }
  overlay.redraw();
}
function parseSNReports(txt){
  const out=[];
  for(const m of txt.matchAll(/Object:\s*(-?[\d.]+)\s*,\s*(-?[\d.]+)([\s\S]*?)End:/gi)){
    const body=m[3], q=(body.match(/(?:Icon|Text):[^"\n]*"([^"]*)"/i)||[])[1]||''; const t=q.replace(/\\n/g,'\n');
    const tm=t.match(/(\d{4}-\d{2}-\d{2})[ T](\d{2}:\d{2}(?::\d{2})?)\s*(UTC|Z|GMT)?/i);
    const typeLine=(t.match(/(Tornado|Funnel Cloud|Rotating Wall Cloud|Wall Cloud|Hail|Wind|Flood(?:ing)?|Flash Flood|Snow|Freezing Rain|Damage|Other)[^\n]*/i)||['Report'])[0].trim();
    out.push({lat:+m[1],lon:+m[2],t:tm?Date.parse(`${tm[1]}T${tm[2]}Z`):Date.now(),type:typeLine.slice(0,48),by:((t.match(/Reported By:\s*([^\n]+)/i)||[])[1]||'').trim(),
      mag:((t.match(/Size:\s*([\d.]+)/i)||t.match(/(\d+)\s*mph/i)||[])[1])||'',unit:/Size:/i.test(t)?'in':/mph/i.test(t)?'mph':'',remark:((t.match(/(?:Notes?|Comments?):\s*([^\n]+)/i)||[])[1]||'').trim()});
  }
  return out;
}
function drawReports(x,P,W,H,hit,fr){
  if(!fr||(!state.lsr&&!state.sn)) return;
  const t1=+fr.t+60e3, win=RPT_WIN[state.rptWinIdx]*60e3, z=map.getZoom();
  const sets=[];
  if(state.lsr) sets.push(['lsr',reports.lsr]);
  if(state.sn) sets.push(['sn',reports.snBlocked?reports.lsr.filter(r=>/SPOTTER|CHASER|AMATEUR|SKYWARN|RADIO|STORM CHASER/i.test(r.src||'')).map(r=>Object.assign({},r,{fromLsr:true})):reports.sn]);
  x.save(); x.font='700 10px "IBM Plex Mono",monospace'; x.textAlign='center'; x.textBaseline='middle';
  for(const [kind,list] of sets) for(const r of list){
    if(r.t>t1||r.t<t1-win) continue;
    const p=P([r.lat,r.lon]); if(p.x<-20||p.y<-20||p.x>W+20||p.y>H+20) continue;
    if(kind==='lsr'&&state.sn&&reports.snBlocked&&/SPOTTER|CHASER|AMATEUR|SKYWARN|RADIO/i.test(r.src||'')) continue; // drawn once as spotter
    const c=rptCat(r.type), age=(t1-r.t)/win; x.globalAlpha=Math.max(.45,1-age*.55);
    const s=z<6?6:7.5;
    x.beginPath();
    if(kind==='sn'){ x.moveTo(p.x,p.y-s-1.5); x.lineTo(p.x+s+1.5,p.y); x.lineTo(p.x,p.y+s+1.5); x.lineTo(p.x-s-1.5,p.y); x.closePath(); }
    else if(x.roundRect) x.roundRect(p.x-s,p.y-s,2*s,2*s,3); else x.rect(p.x-s,p.y-s,2*s,2*s);
    x.fillStyle=c[1]; x.fill(); x.lineWidth=kind==='sn'?2:1.5; x.strokeStyle=kind==='sn'?'#fff':'#000'; x.stroke();
    x.fillStyle=c[3]; x.fillText(c[0],p.x,p.y+.5);
    if(z>=8&&r.mag!=null&&r.mag!==''&&+r.mag>0){ const lab=`${r.mag}${r.unit==='in'||/HAIL/i.test(r.type)?'"':r.unit&&/MPH/i.test(r.unit)?'':''}`; x.textAlign='left'; x.lineWidth=3; x.strokeStyle='rgba(0,0,0,.9)'; x.strokeText(lab,p.x+s+3,p.y); x.fillStyle='#fff'; x.fillText(lab,p.x+s+3,p.y); x.textAlign='center'; }
    hit.push({x:p.x,y:p.y,kind,r});
  }
  x.restore();
}

/* ================= wind particles from radar velocity =================
   Radial component: the observed Doppler velocity at each gate.
   Cross-beam component: a VAD fit (u,v) from the same scan, so particles can move across the beam. */
map.createPane('wind').style.zIndex=445; map.getPane('wind').style.pointerEvents='none';
function velMS(d){ if(d._ms) return d._ms; const T=decodeTable(d.dec), o=new Float32Array(256); for(let c=0;c<256;c++) o[c]=T.rf[c]?NaN:T.v[c]/1.943844; return d._ms=o; }
function computeVAD(d){
  if(d._vad!==undefined) return d._vad; const ms=velMS(d), nG=d.nGates;
  const rmin=Math.max(10,d.firstKm+d.gateKm), rmax=Math.min(90,d.firstKm+d.gateKm*nG);
  const g0=Math.max(0,Math.ceil((rmin-d.firstKm)/d.gateKm)), g1=Math.floor((rmax-d.firstKm)/d.gateKm), gs=Math.max(1,Math.round((g1-g0)/50));
  const A=[0,0,0,0,0,0,0,0,0], B=[0,0,0], sec=new Uint8Array(36); let n=0;
  for(let row=0;row<720;row+=2){ const a=(row+.5)*.5*Math.PI/180, sn=Math.sin(a), cs=Math.cos(a), base=row*nG;
    for(let g=g0;g<g1;g+=gs){ const v=ms[d.grid[base+g]]; if(isNaN(v)) continue; const X=[sn,cs,1];
      for(let i=0;i<3;i++){ B[i]+=X[i]*v; for(let j=0;j<3;j++) A[i*3+j]+=X[i]*X[j]; } n++; sec[(row/20)|0]=1; } }
  let cover=0; for(const q of sec) cover+=q;
  if(n<150||cover<20) return d._vad=null;
  const det=m=>m[0]*(m[4]*m[8]-m[5]*m[7])-m[1]*(m[3]*m[8]-m[5]*m[6])+m[2]*(m[3]*m[7]-m[4]*m[6]); const D=det(A); if(Math.abs(D)<1e-9) return d._vad=null;
  const col=(k)=>{ const m=A.slice(); for(let i=0;i<3;i++) m[i*3+k]=B[i]; return det(m)/D; };
  const ce=Math.cos((d.elev||0.5)*Math.PI/180); const u=col(0)/ce, v=col(1)/ce;
  return d._vad=(Math.hypot(u,v)<90?[u,v]:null);
}
const WIND_COLS=['rgba(200,215,230,.75)','rgba(150,230,255,.85)','rgba(120,255,170,.9)','rgba(255,235,90,.95)','rgba(255,150,60,.95)','rgba(255,70,90,1)'];
const WindLayer=ScreenCanvas.extend({
  init(){ this.ctx=this._c.getContext('2d'); this.parts=[]; this.run=false; },
  draw(){ if(!this.ctx) return; this.ctx.setTransform(1,0,0,1,0,0); this.ctx.clearRect(0,0,this._c.width,this._c.height); this.parts.length=0; },
  start(){ if(this.run) return; this.run=true; const loop=()=>{ if(!this.run) return; if(!document.hidden&&!this._map._animatingZoom) this.step(); this.raf2=requestAnimationFrame(loop); }; this.raf2=requestAnimationFrame(loop); },
  stop(){ this.run=false; cancelAnimationFrame(this.raf2); this.draw(); },
  step(){
    const d=wind.field(), x=this.ctx, dpr=this.dpr, W=this._c.width/dpr, H=this._c.height/dpr;
    x.setTransform(1,0,0,1,0,0); x.globalCompositeOperation='destination-in'; x.fillStyle='rgba(0,0,0,.92)'; x.fillRect(0,0,this._c.width,this._c.height); x.globalCompositeOperation='source-over';
    if(!d){ this.parts.length=0; return; }
    x.setTransform(dpr,0,0,dpr,0,0);
    const m=this._map, c0=m.latLngToContainerPoint([d.lat,d.lon]), pk=Math.abs(m.latLngToContainerPoint(destLL(d.lat,d.lon,20,90)).x-c0.x)/20;
    const ms=velMS(d), vad=computeVAD(d)||[0,0], nG=d.nGates, rMax=d.firstKm+d.gateKm*nG, grid=d.grid;
    const N=Math.min(4200,Math.round(W*H/260)), k=0.075*(1+Math.max(0,m.getZoom()-7)*0.15);
    const sampleV=(px,py)=>{ const dx=(px-c0.x)/pk, dy=(c0.y-py)/pk, r=Math.hypot(dx,dy); if(r<d.firstKm||r>=rMax) return null;
      let az=Math.atan2(dx,dy); if(az<0) az+=2*Math.PI; const row=((az*180/Math.PI/0.5)|0)%720, g=((r-d.firstKm)/d.gateKm)|0; const vr=ms[grid[row*nG+g]]; if(isNaN(vr)) return null;
      const sa=Math.sin(az), ca=Math.cos(az), vt=vad[0]*ca-vad[1]*sa; return [vr*sa+vt*ca, vr*ca-vt*sa]; };
    const spawn=p=>{ for(let i=0;i<8;i++){ p.x=Math.random()*W; p.y=Math.random()*H; if(sampleV(p.x,p.y)) break; } p.age=0; p.max=40+Math.random()*70; };
    while(this.parts.length<N){ const p={}; spawn(p); p.age=Math.random()*p.max; this.parts.push(p); }
    if(this.parts.length>N) this.parts.length=N;
    const paths=WIND_COLS.map(()=>new Path2D());
    for(const p of this.parts){
      const v=sampleV(p.x,p.y); if(!v||++p.age>p.max){ spawn(p); continue; }
      const spd=Math.hypot(v[0],v[1]), nx=p.x+v[0]*k, ny=p.y-v[1]*k;
      const b=spd<5?0:spd<10?1:spd<17?2:spd<25?3:spd<35?4:5; paths[b].moveTo(p.x,p.y); paths[b].lineTo(nx,ny); p.x=nx; p.y=ny;
      if(nx<0||ny<0||nx>W||ny>H) spawn(p);
    }
    x.lineWidth=1.3; x.lineCap='round'; paths.forEach((pa,i)=>{ x.strokeStyle=WIND_COLS[i]; x.stroke(pa); });
  }
});
const windLayer=new WindLayer('wind','ovl-canvas');
const wind={warned:false, index:[], indexKey:'', cache:new Map(), map:new Map(), site:null,
  // velocity grid for the current frame; falls back to the nearest earlier loaded frame so particles never blank out mid-loop
  field(){
    const f=curFrame(); if(!f) return null;
    if(f.data&&f.data.pal==='VEL'&&f.data.lat!=null) return f.data;
    if(this.site!==(state.site&&state.site.id)) return null;
    const i0=state.idx;
    for(let k=0;k<state.frames.length;k++){ const fr=state.frames[(i0-k+state.frames.length)%state.frames.length], key=this.map.get(fr.url), c=key&&this.cache.get(key); if(c&&c.d) return c.d; if(k>3&&this.lastD) return this.lastD; }
    return this.lastD||null;
  },
  async buildIndex(){
    const s=state.site; if(!s) return [];
    if(isMrrl(s)){
      if(s.src==='ideam'&&!s.rawBlocked) return state.frames.filter(f=>f.type==='iris').map(f=>({t:+f.t,key:f.url+'|VEL',job:['iris',f.url,{prod:'VEL',tilt:0}]}));
      if(s.src==='opera'&&!s.rawBlocked){ const sv=state.prod; state.prod='VEL'; try{ const v=await listOperaSite(s); return v.map(f=>({t:+f.t,key:f.url+'|vel',job:['odim',f.url,{quants:['VRADH','VRAD'],tilt:0,vel:true}]})); } finally { state.prod=sv; } }
      if(s.src==='fmi'&&!s.rawBlocked) return state.frames.filter(f=>f.type==='odim').map(f=>({t:+f.t,key:f.url+'|vel',job:['odim',f.url,{quants:['VRADH','VRAD','VRADDH','VRADV'],tilt:0,vel:true}]}));
      if(s.src==='dwd'&&!s.rawBlocked){ const v=await listDWD(s,true); return v.map(f=>({t:+f.t,key:f.url,job:['odim',f.url,{quants:['VRADH','VRAD'],tilt:0,vel:true}]})); }
      return [];
    }
    if(state.src==='L2'&&s.type!=='TDWR') return state.frames.filter(f=>f.type==='l2'||f.type==='l2c').map(f=>({t:+f.t,key:f.url+'|DVEL',job:[f.type,f.url,{block:'DVEL',tilt:0,urls:f.urls}]}));
    const id3=s.id.slice(1), codes=s.type==='TDWR'?['TV0']:['N0G','N0U'];
    for(const code of codes){ const out=[];
      for(const d of dayPrefixes()){ const [Y,M,D]=ymdU(d); const keys=await cachedList(S3.l3,`${id3}_${code}_${Y}_${M}_${D}`,55000).catch(()=>[]);
        for(const k of keys){ const m=k.match(/_(\d{4})_(\d{2})_(\d{2})_(\d{2})_(\d{2})_(\d{2})$/); if(m) out.push({t:Date.UTC(+m[1],m[2]-1,+m[3],+m[4],+m[5],+m[6]),key:k,job:['l3',`${S3.l3}/${k}`,{}]}); } }
      if(out.length&&Date.now()-Math.max(...out.map(o=>o.t))<3*3600e3) return out.sort((a,b)=>a.t-b.t);
    }
    return [];
  },
  async refreshBg(){
    if(!state.wind||!state.site||!state.frames.length) return;
    const s=state.site; if(this.site!==s.id){ this.site=s.id; this.cache.clear(); this.map.clear(); this.lastD=null; this.index=[]; }
    if(state.frames.every(f=>f.data&&f.data.pal==='VEL')) return;
    try{ this.index=await this.buildIndex(); }catch(e){ console.warn('wind index',e); this.index=[]; }
    if(!this.index.length){ if(!this.warned){ this.warned=true; toast(isMrrl(s)?'Wind particles need radar velocity, which this site does not provide.':'No velocity scans found for wind particles at this radar.',5000); } return; }
    // match each radar frame to the closest velocity scan within 8 minutes
    this.map.clear(); const n=state.frames.length;
    state.frames.forEach((f,i)=>{ let best=null,bd=8*60e3; for(const v of this.index){ const dd=Math.abs(v.t-f.t); if(dd<=bd){ bd=dd; best=v; } } if(best){ this.map.set(f.url,best.key); this.load(best,20+i+(i===n-1?40:0)); } });
    if(!this.map.size&&this.index.length){ const v=this.index[this.index.length-1]; state.frames.forEach(f=>this.map.set(f.url,v.key)); this.load(v,80); }
    const keep=new Set(this.map.values()); for(const k of this.cache.keys()) if(!keep.has(k)) this.cache.delete(k);
  },
  load(v,prio){
    let c=this.cache.get(v.key); if(c) return c; c={status:'loading',d:null}; this.cache.set(v.key,c);
    pool.run(v.job[0],v.job[1],v.job[2],prio,'wind').then(d=>{ d.pal='VEL'; c.d=d; c.status='ok'; this.lastD=d; }).catch(e=>{ c.status='err'; if(e.message==='cancelled') this.cache.delete(v.key); });
    return c;
  },
  onFrame(f){ if(!state.wind||!f) return; const key=this.map.get(f.url); if(!key) return; const c=this.cache.get(key); if(!c){ const v=this.index.find(x=>x.key===key); if(v) this.load(v,85); } else if(c.status==='loading') pool.bump(j=>j.tag==='wind'&&j.url&&key.includes(j.url.split('/').pop()),85); },
  set(on){ state.wind=on; store.set('wind',on); if(on){ windLayer.addTo(map); windLayer.start(); this.refreshBg(); } else { windLayer.stop(); map.removeLayer(windLayer); pool.drop(j=>j.tag==='wind'); } }
};

$('#lyLSR').checked=state.lsr; $('#lySN').checked=state.sn; $('#lyWind').checked=state.wind;
$('#rptWin').value=state.rptWinIdx;
const fmtWin=m=>m<60?m+' min':(m/60)+' h'; $('#rptWinOut').textContent=fmtWin(RPT_WIN[state.rptWinIdx]);
$('#lyLSR').onchange=e=>{ state.lsr=e.target.checked; store.set('lsr',state.lsr); if(state.lsr) loadReports(true); else overlay.redraw(); };
$('#lySN').onchange=e=>{ state.sn=e.target.checked; store.set('sn',state.sn); if(state.sn) loadReports(true); else overlay.redraw(); };
$('#rptWin').oninput=e=>{ state.rptWinIdx=+e.target.value; store.set('rptWinIdx',state.rptWinIdx); $('#rptWinOut').textContent=fmtWin(RPT_WIN[state.rptWinIdx]); overlay.redraw(); clearTimeout(rptT); rptT=setTimeout(()=>loadReports(true),400); };
let rptT=0;
$('#lyWind').onchange=e=>wind.set(e.target.checked);
if(state.wind) wind.set(true);
setInterval(()=>{ if(!document.hidden&&(state.lsr||state.sn)) loadReports(); },60e3);

/* ================= METARs (aviationweather.gov Data API) ================= */
state.metar=store.get('metar',false);
const FLTCAT={VFR:'#2bd86b',MVFR:'#4ea3ff',IFR:'#ff3b4a',LIFR:'#d36bff'};
const metars={st:new Map(),bbox:null,at:0,hours:0,busy:false};
function fltCat(vis,ceil){ if((ceil!=null&&ceil<500)||(vis!=null&&vis<1)) return 'LIFR'; if((ceil!=null&&ceil<1000)||(vis!=null&&vis<3)) return 'IFR'; if((ceil!=null&&ceil<=3000)||(vis!=null&&vis<=5)) return 'MVFR'; return 'VFR'; }
const IEM='https://mesonet.agron.iastate.edu';
const COVER_RANK={CLR:0,SKC:0,NSC:0,FEW:1,SCT:2,BKN:3,OVC:4,VV:5};
function iemOb(r){
  const num=v=>v==null||v===''||v==='M'||v==='T'?null:+v;
  const layers=[1,2,3,4].map(i=>({c:r['skyc'+i],b:num(r['skyl'+i])})).filter(l=>l.c&&l.c!=='M'&&l.c.trim());
  let cover=null,best=-1; for(const l of layers){ const k=COVER_RANK[l.c.trim()]; if(k!=null&&k>best){ best=k; cover=l.c.trim(); } }
  const ceilL=layers.filter(l=>/BKN|OVC|VV/.test(l.c)).map(l=>l.b).filter(x=>x!=null);
  const vis=num(r.vsby), tF=num(r.tmpf), dF=num(r.dwpf), alt=num(r.alti), valid=String(r.utc_valid||r.valid||'');
  const t=Date.parse(/Z$|[+-]\d\d:?\d\d$/.test(valid)?valid:valid.replace(' ','T')+'Z');
  return {id:r.station,name:r.name||r.station,lat:+r.lat,lon:+(r.lon??r.lng),t,
    temp:tF==null?null:(tF-32)*5/9,dewp:dF==null?null:(dF-32)*5/9,wdir:num(r.drct),wspd:num(r.sknt),wgst:num(r.gust),visib:vis,altim:alt?alt/0.02953:null,
    wx:(r.wxcodes&&r.wxcodes!=='M')?String(r.wxcodes).trim():'',raw:r.raw||r.metar||'',cover,sky:layers.map(l=>`${l.c.trim()}${l.b!=null?' '+l.b.toLocaleString()+' ft':''}`).join(', ')||cover||'',
    cat:fltCat(vis,ceilL.length?Math.min(...ceilL):null)};
}
function parseCSV(txt){ const lines=txt.replace(/\r/g,'').split('\n').filter(l=>l&&!l.startsWith('#')); if(!lines.length) return []; const h=lines[0].split(',');
  return lines.slice(1).map(l=>{ const c=[]; let cur='',q=false; for(const ch of l){ if(ch==='"') q=!q; else if(ch===','&&!q){ c.push(cur); cur=''; } else cur+=ch; } c.push(cur); const o={}; h.forEach((k,i)=>o[k.trim()]=c[i]); return o; }); }
let iemCurAt=0, iemCur=[];
async function loadMetars(force){
  if(!state.metar||metars.busy) return;
  const vb=map.getBounds(), first=state.frames.length?+state.frames[0].t:Date.now()-3600e3, spanMin=(Date.now()-first)/60000;
  const stale=Date.now()-metars.at>5*60e3, moved=!metars.bbox||!metars.bbox.contains(vb);
  if(!force&&!stale&&!moved&&first>=metars.from) return;
  metars.busy=true;
  try{
    // latest obs for every ASOS/AWOS in the US and Canada (IEM API v1)
    if(force||Date.now()-iemCurAt>4*60e3||!iemCur.length){
      const got=await Promise.all(['US','CA'].map(c=>fetch(`${IEM}/api/1/currents.json?networkclass=ASOS&country=${c}`).then(r=>r.ok?r.json():{data:[]}).catch(()=>({data:[]}))));
      iemCur=got.flatMap(j=>j.data||[]).map(iemOb).filter(o=>!isNaN(o.lat)&&!isNaN(o.t)); iemCurAt=Date.now();
      if(!iemCur.length) throw new Error('no current obs');
    }
    const b=vb.pad(.3), st=new Map();
    const inView=iemCur.filter(o=>b.contains([o.lat,o.lon]));
    for(const o of inView) st.set(o.id,[o]);
    // history for the loop: IEM ASOS archive (routine + special METARs), nearest stations first
    if(spanMin>70&&inView.length){
      const c=vb.getCenter(), ids=inView.slice().sort((p,q)=>((p.lat-c.lat)**2+(p.lon-c.lng)**2)-((q.lat-c.lat)**2+(q.lon-c.lng)**2)).slice(0,240).map(o=>o.id);
      const s0=new Date(first-60*60e3), e0=new Date(Date.now()+10*60e3);
      const dt=(d,n)=>`year${n}=${d.getUTCFullYear()}&month${n}=${d.getUTCMonth()+1}&day${n}=${d.getUTCDate()}&hour${n}=${d.getUTCHours()}&minute${n}=${d.getUTCMinutes()}`;
      const chunks=[]; for(let i=0;i<ids.length;i+=60) chunks.push(ids.slice(i,i+60));
      await Promise.all(chunks.map(async ch=>{
        const u=`${IEM}/cgi-bin/request/asos.py?${ch.map(i=>'station='+encodeURIComponent(i)).join('&')}&data=tmpf&data=dwpf&data=drct&data=sknt&data=gust&data=vsby&data=alti&data=skyc1&data=skyc2&data=skyc3&data=skyc4&data=skyl1&data=skyl2&data=skyl3&data=skyl4&data=wxcodes&data=metar&${dt(s0,1)}&${dt(e0,2)}&tz=Etc/UTC&format=onlycomma&latlon=yes&missing=M&trace=T&report_type=3&report_type=4`;
        try{ const r=await fetch(u); if(!r.ok) return; for(const row of parseCSV(await r.text())){ const o=iemOb(row); if(isNaN(o.t)||isNaN(o.lat)) continue; const cur=st.get(o.id); if(cur){ const nm=cur[cur.length-1].name; o.name=nm; cur.push(o); } } }catch(e){ console.warn('IEM ASOS history',e); }
      }));
      for(const a of st.values()){ a.sort((x,y)=>x.t-y.t); for(let i=a.length-1;i>0;i--) if(Math.abs(a[i].t-a[i-1].t)<60e3) a.splice(i-1,1); }
    }
    metars.st=st; metars.bbox=b; metars.at=Date.now(); metars.from=first; overlay.redraw();
  }catch(e){ console.warn('METAR',e); if(force) toast('METARs could not be loaded from the Iowa Environmental Mesonet.'); }
  finally{ metars.busy=false; }
}
const COVER_FRAC={SKC:0,CLR:0,CAVOK:0,NSC:0,FEW:.25,SCT:.5,BKN:.75,OVC:1,OVX:1,VV:1};
function drawBarb(x,cx,cy,dir,spd,col,lw=1.6){
  if(spd==null) return;
  if(spd<3){ x.beginPath(); x.arc(cx,cy,8.5,0,7); x.strokeStyle=col; x.lineWidth=lw*.8; x.stroke(); return; }
  if(dir==='VRB'||dir==null) return;
  const a=(+dir)*Math.PI/180, ux=Math.sin(a), uy=-Math.cos(a), L0=7, L1=30, px=-uy, py=ux; // barbs on the clockwise side
  const ex=cx+ux*L1, ey=cy+uy*L1; x.strokeStyle=col; x.fillStyle=col; x.lineWidth=lw; x.lineCap='round'; x.lineJoin='round';
  x.beginPath(); x.moveTo(cx+ux*L0,cy+uy*L0); x.lineTo(ex,ey); x.stroke();
  let s5=Math.round(spd/5)*5, pos=0; const step=4.2, bl=11;
  while(s5>=50){ const bx=ex-ux*pos, by=ey-uy*pos; x.beginPath(); x.moveTo(bx,by); x.lineTo(bx+px*bl-ux*2.5,by+py*bl-uy*2.5); x.lineTo(bx-ux*step*1.3,by-uy*step*1.3); x.closePath(); x.fill(); pos+=step*1.6; s5-=50; }
  while(s5>=10){ const bx=ex-ux*pos, by=ey-uy*pos; x.beginPath(); x.moveTo(bx,by); x.lineTo(bx+px*bl+ux*3,by+py*bl+uy*3); x.stroke(); pos+=step; s5-=10; }
  if(s5>=5){ if(pos===0) pos=step; const bx=ex-ux*pos, by=ey-uy*pos; x.beginPath(); x.moveTo(bx,by); x.lineTo(bx+px*bl*.55+ux*1.6,by+py*bl*.55+uy*1.6); x.stroke(); }
}
function drawMetars(x,P,W,H,hit,fr){
  if(!state.metar||!fr||!metars.st.size) return;
  const t=+fr.t, z=map.getZoom(), full=z>=7, cell=full?(z>=9?58:72):16, occ=new Set(), C=state.units==='mi';
  const tc=c=>c==null?'':String(Math.round(C?c*9/5+32:c));
  x.save(); x.textBaseline='middle'; x.lineJoin='round';
  for(const a of metars.st.values()){
    let o=null; for(let i=a.length-1;i>=0;i--){ if(a[i].t<=t+5*60e3){ o=a[i]; break; } } if(!o||t-o.t>90*60e3) continue;
    const p=P([o.lat,o.lon]); if(p.x<-30||p.y<-30||p.x>W+30||p.y>H+30) continue;
    const k=`${Math.floor(p.x/cell)},${Math.floor(p.y/cell)}`; if(occ.has(k)) continue; occ.add(k);
    const col=FLTCAT[o.cat]||'#c8cdd4';
    if(!full){ x.beginPath(); x.arc(p.x,p.y,3.6,0,7); x.fillStyle=col; x.fill(); x.lineWidth=1; x.strokeStyle='#000'; x.stroke(); hit.push({x:p.x,y:p.y,kind:'metar',o}); continue; }
    // wind barb (black casing, then white)
    drawBarb(x,p.x,p.y,o.wdir,o.wspd,'rgba(0,0,0,.8)',3.6); drawBarb(x,p.x,p.y,o.wdir,o.wspd,'#f2f4f7',1.6);
    // sky cover circle in flight-category color
    const fr_=COVER_FRAC[o.cover]!=null?COVER_FRAC[o.cover]:0, r=5.5;
    x.beginPath(); x.arc(p.x,p.y,r,0,7); x.fillStyle='rgba(8,10,14,.9)'; x.fill();
    if(fr_>0){ x.beginPath(); x.moveTo(p.x,p.y); x.arc(p.x,p.y,r,-Math.PI/2,-Math.PI/2+fr_*Math.PI*2); x.closePath(); x.fillStyle=col; x.fill(); }
    x.beginPath(); x.arc(p.x,p.y,r,0,7); x.lineWidth=1.6; x.strokeStyle=col; x.stroke();
    x.font='600 11px "IBM Plex Mono",monospace'; x.lineWidth=3; x.lineJoin='round'; x.strokeStyle='rgba(0,0,0,.9)'; x.textAlign='right';
    const txt=(s,xx,yy,c)=>{ if(!s) return; x.strokeText(s,xx,yy); x.fillStyle=c; x.fillText(s,xx,yy); };
    txt(tc(o.temp),p.x-9,p.y-9,'#ff7a7a'); txt(tc(o.dewp),p.x-9,p.y+10,'#5ee08a');
    if(o.wx){ x.font='600 10px "IBM Plex Mono",monospace'; txt(o.wx.split(' ')[0].slice(0,6),p.x-9,p.y+.5,'#ffd23f'); }
    x.textAlign='left'; x.font='600 10px "IBM Plex Mono",monospace';
    if(o.wgst) txt(`G${o.wgst}`,p.x+9,p.y-9,'#ffb347');
    if(z>=8.5) txt(o.id,p.x+9,p.y+10,'rgba(236,230,218,.75)');
    hit.push({x:p.x,y:p.y,kind:'metar',o});
  }
  x.restore();
}
$('#lyMetar').checked=state.metar;
$('#lyMetar').onchange=e=>{ state.metar=e.target.checked; store.set('metar',state.metar); if(state.metar) loadMetars(true); else overlay.redraw(); };
let metT=0; map.on('moveend',()=>{ if(!state.metar) return; clearTimeout(metT); metT=setTimeout(()=>loadMetars(false),700); });
setInterval(()=>{ if(state.metar&&!document.hidden) loadMetars(false); },60e3);

/* ================= split screen (pane B: second product, synced) ================= */
const split={on:false,map:null,radar:null,ctx:{prod:'VEL',tilt:0,src:'L3',l3code:''},frames:[],cache:new Map(),alerts:null,base:null,ref:null,xh:[],syncing:false,gen:0,
  init(){
    const m2=this.map=L.map('map2',{zoomControl:false,attributionControl:false,zoomSnap:0.25,wheelPxPerZoomLevel:90,fadeAnimation:false,worldCopyJump:true});
    [['radar',350],['alerts',420],['labels',450],['tools',480]].forEach(([n,z])=>{ const p=m2.createPane(n); p.style.zIndex=z; if(n!=='alerts') p.style.pointerEvents='none'; });
    m2.setView(map.getCenter(),map.getZoom(),{animate:false});
    this.radar=new RadarGL('radar','radar-canvas'); this.radar.addTo(m2);
    this.alerts=L.layerGroup().addTo(m2);
    const sync=(a,b)=>{ if(this.syncing) return; this.syncing=true; b.setView(a.getCenter(),a.getZoom(),{animate:false}); this.syncing=false; };
    map.on('move zoomend',()=>this.on&&sync(map,m2)); m2.on('move zoomend',()=>this.on&&sync(m2,map));
    const icon=L.divIcon({className:'xhair',html:'<i></i><b></b>',iconSize:[0,0]});
    this.xh=[L.marker([0,0],{icon,interactive:false,pane:'tools'}),L.marker([0,0],{icon,interactive:false,pane:'tools'})];
    const show=(ll,which)=>{ const mk=this.xh[which], tgt=which?m2:map; mk.setLatLng(ll); if(!tgt.hasLayer(mk)) mk.addTo(tgt); this.readout(ll); };
    map.on('mousemove',e=>this.on&&show(e.latlng,1)); m2.on('mousemove',e=>{ if(!this.on) return; show(e.latlng,0); lastPt=e.latlng; updateReadout(); });
    map.on('mouseout',()=>{ if(m2.hasLayer(this.xh[1])) m2.removeLayer(this.xh[1]); }); m2.on('mouseout',()=>{ if(map.hasLayer(this.xh[0])) map.removeLayer(this.xh[0]); });
    m2.on('click',e=>map.fire('click',{latlng:e.latlng,containerPoint:map.latLngToContainerPoint(e.latlng),originalEvent:e.originalEvent}));
    this.setBasemap(store.get('bm','dark'));
    $('#pbStrip').onclick=e=>{ const b=e.target.closest('[data-f]'); if(!b||b.disabled) return; this.ctx.prod=b.dataset.f; this.ctx.tilt=0; store.set('splitProd',this.ctx.prod); this.ui(); this.reload(true); };
    $('#pbTilt').onchange=e=>{ this.ctx.tilt=+e.target.value; this.reload(true); };
  },
  setBasemap(k){ if(!this.map) return; const b=BASEMAPS[k]; if(this.base) this.map.removeLayer(this.base); if(this.ref) this.map.removeLayer(this.ref);
    this.base=L.tileLayer(b.base,{maxNativeZoom:b.max,maxZoom:19,keepBuffer:3,updateWhenZooming:false,crossOrigin:'anonymous'}).addTo(this.map);
    this.ref=b.ref?L.tileLayer(b.ref,{pane:'labels',maxNativeZoom:b.max,maxZoom:19,keepBuffer:3,updateWhenZooming:false,crossOrigin:'anonymous'}).addTo(this.map):null; },
  usable(){ const s=state.site; return s&&!(isMrrl(s)&&!mrrlRaw(s)); },
  srcFor(f){ const s=state.site; if(isMrrl(s)) return state.src; return famAvail(f,s,state.src)?state.src:(state.src==='L2'?'L3':'L2'); },
  ui(){
    const s=state.site, fams=CATALOG.filter(f=>famAnywhere(f,s));
    if(!fams.some(f=>f.id===this.ctx.prod)) this.ctx.prod=fams.find(f=>f.id!==state.prod)?.id||'REF';
    const f=famById(this.ctx.prod); this.ctx.src=this.srcFor(f);
    $('#pbStrip').innerHTML=CATALOG.map(c=>`<button data-f="${c.id}" aria-pressed="${c.id===this.ctx.prod}" ${famAnywhere(c,s)?'':'disabled'} title="${c.name}">${c.label}</button>`).join('');
    const td=s&&s.type==='TDWR'; let opts;
    if(isMrrl(s)) opts=s.src==='dwd'?DWD_SWEEPS.map((w,i)=>[i,w[1].toFixed(1)+'°']):[[0,'Tilt 1'],[1,'Tilt 2'],[2,'Tilt 3']];
    else if(this.ctx.src==='L3') opts=((td?f.tdwr:f.l3)||[]).map((c,i)=>[i,c]); else opts=[0,1,2,3,4,5].map(i=>[i,'Tilt '+(i+1)]);
    if(this.ctx.tilt>=opts.length) this.ctx.tilt=0;
    $('#pbTilt').innerHTML=opts.map(o=>`<option value="${o[0]}" ${o[0]===this.ctx.tilt?'selected':''}>${o[1]}</option>`).join(''); $('#pbTilt').disabled=opts.length<2;
    const p=PAL[f.pal], cv=$('#pbLegend'), x=cv.getContext('2d'); x.clearRect(0,0,cv.width,cv.height);
    if(p.cat){ const cls=HCA_CLASSES.filter(k=>k[1]!=='RF'); cls.forEach((k,i)=>{ x.fillStyle=k[3]; x.fillRect(i*cv.width/cls.length,0,cv.width/cls.length-1,cv.height); }); }
    else for(let i=0;i<cv.width;i++){ const c=palColor(p,p.min+(p.max-p.min)*i/(cv.width-1)); x.fillStyle=`rgba(${c[0]|0},${c[1]|0},${c[2]|0},${Math.max(c[3],.25)})`; x.fillRect(i,0,1,cv.height); }
  },
  toggle(force){
    const on=force!=null?force:!this.on;
    if(on&&!this.usable()){ toast('Split view needs a radar with polar products. Composite-only sites are not supported.'); return; }
    this.on=on; document.body.classList.toggle('split',on); $('#map2').hidden=!on; $('#paneB').hidden=!on; setToolPressed('split',on); store.set('split',on);
    if(on){ if(!this.map){ const pref=store.get('splitProd',null); this.ctx.prod=pref&&pref!==state.prod?pref:(state.prod==='REF'?'VEL':'REF'); this.init(); } }
    setTimeout(()=>{ map.invalidateSize(); if(this.map){ this.map.invalidateSize(); this.map.setView(map.getCenter(),map.getZoom(),{animate:false}); } if(on){ this.ui(); this.reload(true); } else pool.drop(j=>j.tag==='split'); },30);
  },
  async reload(reset){
    if(!this.on||!state.site||!state.frames.length) return;
    if(!this.usable()){ this.toggle(false); return; }
    const gen=++this.gen, s=state.site, f=famById(this.ctx.prod); this.ctx.src=this.srcFor(f);
    if(reset){ pool.drop(j=>j.tag==='split'); this.radar.setData(null); }
    let list=[];
    try{
      if(isMrrl(s)){ if(s.src==='dwd') list=await listDWD(s,false,this.ctx); else if(s.src==='opera'&&(this.ctx.prod==='VEL')!==(state.prod==='VEL')){ const sv=state.prod; state.prod=this.ctx.prod; try{ list=await listOperaSite(s); } finally { state.prod=sv; } } else list=state.frames.filter(x=>x.type==='odim'||x.type==='iris').map(x=>({t:x.t,url:x.url,type:x.type})); }
      else list=await listRadarFrames(this.ctx);
    }catch(e){ console.warn('split list',e); }
    if(gen!==this.gen) return;
    if(!list.length){ $('#pbTime').textContent='no data'; this.frames=[]; this.radar.setData(null); return; }
    this.frames=state.frames.map(fr=>{ let best=null,bd=10*60e3; for(const v of list){ const dd=Math.abs(v.t-fr.t); if(dd<bd){bd=dd;best=v;} } return best?this.entry(best):null; });
    const keep=new Set(this.frames.filter(Boolean).map(e=>e.key)); for(const k of this.cache.keys()) if(!keep.has(k)) this.cache.delete(k);
    const n=this.frames.length; this.frames.forEach((e,i)=>e&&this.load(e,50+i+(i===n-1?45:0)+(i===state.idx?40:0)));
    this.show(state.idx);
  },
  entry(v){ const key=`${v.url}|${this.ctx.prod}|${this.ctx.tilt}`; let e=this.cache.get(key); if(!e){ e={t:v.t,url:v.url,type:v.type,urls:v.urls,key,status:'idle',data:null}; this.cache.set(key,e); } return e; },
  load(e,prio){
    if(e.status!=='idle') return; e.status='loading'; const f=famById(this.ctx.prod), s=state.site, odim=e.type==='odim', vel=this.ctx.prod==='VEL';
    const opts=e.type==='iris'?{prod:this.ctx.prod,tilt:this.ctx.tilt}:odim?{quants:vel?['VRADH','VRAD','VRADDH','VRADV']:['DBZH','TH','DBZ','DBZV'],tilt:s.src==='dwd'?0:this.ctx.tilt,vel}:{block:f.l2,tilt:this.ctx.tilt,urls:e.urls};
    pool.run(odim?'odim':e.type,e.url,opts,prio,'split').then(d=>{ d.pal=f.pal; e.data=d; e.status='ok'; if(this.frames[state.idx]===e||!this.radar.data) this.show(state.idx); })
      .catch(err=>{ e.status=err.message==='cancelled'||err.message==='incomplete'?'idle':'err'; });
  },
  show(i){
    if(!this.on||!this.radar) return; const e=this.frames[i]; let d=e&&e.data;
    if(!d) for(let k=i;k>=0;k--){ const q=this.frames[k]; if(q&&q.data){ d=q.data; break; } }
    this.radar.setData(d||null);
    $('#pbTime').textContent=e?`${zTime(new Date(e.t))}${d&&d.elev!=null?' · '+d.elev.toFixed(1)+'°':''}`:'—';
    if(e&&e.status==='idle') this.load(e,95);
  },
  readout(ll){ const d=this.radar&&this.radar.data; if(!d){ $('#pbVal').textContent=''; return; } const p=sample(d,ll); $('#pbVal').textContent=!p||p.b<0?'':fmtValue(d,p.b); },
  syncAlerts(act){ if(!this.alerts) return; this.alerts.clearLayers(); if(!this.on) return;
    for(const a of act){ const col=alertColor(a); L.geoJSON(a.f,{pane:'alerts',interactive:false,style:{color:'#000',weight:5,opacity:.75,fill:false}}).addTo(this.alerts); L.geoJSON(a.f,{pane:'alerts',interactive:false,style:{color:col,weight:2.4,fillColor:col,fillOpacity:.08}}).addTo(this.alerts); } }
};

/* ================= warning alarm (sound + browser notification) ================= */
const alarm={seen:new Set(),primed:false,ctx:null,
  enable(on){ state.alarm=on; store.set('alarm',on); if(on){ this.primed=false; try{ this.ctx=this.ctx||new (window.AudioContext||window.webkitAudioContext)(); this.ctx.resume(); }catch(e){}
      if('Notification' in window&&Notification.permission==='default') Notification.requestPermission().catch(()=>{}); this.check(alertSegs); } },
  beep(tor){ if(!this.ctx) return; const c=this.ctx, t0=c.currentTime; for(let i=0;i<(tor?4:2);i++){ const o=c.createOscillator(), g=c.createGain(); o.type='square'; o.frequency.value=tor?(i%2?660:880):740; g.gain.setValueAtTime(.0001,t0+i*.32); g.gain.exponentialRampToValueAtTime(.18,t0+i*.32+.02); g.gain.exponentialRampToValueAtTime(.0001,t0+i*.32+.28); o.connect(g).connect(c.destination); o.start(t0+i*.32); o.stop(t0+i*.32+.3); } },
  check(segs){
    if(!state.alarm) return; const now=Date.now(), view=map.getBounds().pad(.1), loc=tools.loc&&tools.loc.ll;
    const live=segs.filter(s=>['TO','SV','FF','EW'].includes(s.ph)&&s.end>now&&s.start<=now+60e3);
    const fresh=[];
    for(const s of live){ const k=s.key+'|'+(s.sentT||s.start); if(this.seen.has(k)) continue; this.seen.add(k);
      if(!this.primed) continue;
      let hit=false; try{ const bb=L.geoJSON(s.f).getBounds(); hit=loc?inGeom(s.f.geometry,loc[0],loc[1]):view.intersects(bb); }catch(e){}
      if(hit) fresh.push(s); }
    this.primed=true;
    if(!fresh.length) return;
    const tor=fresh.some(s=>s.ph==='TO'), top=fresh.sort((a,b)=>(a.ph==='TO'?0:1)-(b.ph==='TO'?0:1))[0], P=top.f.properties;
    this.beep(tor); const msg=`${P.event}: ${(P.areaDesc||'').split(';').slice(0,3).join(', ')}`;
    toast(`⚠ ${msg}`,9000);
    try{ if('Notification' in window&&Notification.permission==='granted') new Notification(P.event,{body:(P.areaDesc||'').split(';').slice(0,4).join(', '),tag:top.key}); }catch(e){}
  }
};

/* ================= SPC Day 1 categorical outlook ================= */
const spc={layer:null,at:0,
  async load(force){
    if(!state.spc) return; if(!force&&Date.now()-this.at<15*60e3&&this.layer) return;
    try{ const r=await fetch('https://www.spc.noaa.gov/products/outlook/day1otlk_cat.nolyr.geojson'); if(!r.ok) throw new Error('HTTP '+r.status); const j=await r.json();
      if(this.layer) map.removeLayer(this.layer);
      if(!map.getPane('outlook')){ const p=map.createPane('outlook'); p.style.zIndex=410; }
      this.layer=L.geoJSON(j,{pane:'outlook',style:f=>({color:f.properties.stroke||'#888',weight:1.6,fillColor:f.properties.fill||'#888',fillOpacity:.18,opacity:.9}),
        onEachFeature:(f,l)=>l.bindTooltip(`SPC Day 1 · ${f.properties.LABEL2||f.properties.LABEL||''}`,{sticky:true,className:'site-tip'})}).addTo(map);
      this.at=Date.now();
    }catch(e){ console.warn('SPC',e); toast('SPC outlook could not be loaded (the SPC server may block browser requests).',5000); state.spc=false; $('#lySPC').checked=false; store.set('spc',false); }
  },
  set(on){ state.spc=on; store.set('spc',on); if(on) this.load(true); else if(this.layer){ map.removeLayer(this.layer); this.layer=null; } }
};

/* ================= storm ETA from a SCIT cell ================= */
function trackFromCell(c,base){
  const kmh=c.kt*1.852, to=(c.dir+180)%360, tb=base||+curFrame().t, back=destLL(c.pos[0],c.pos[1],kmh/6,(to+180)%360);
  tools.clearTrack(); tools.track={a:back,b:c.pos,ta:tb-10*60e3,tb,brg:to,kmh}; tools.drawTrack();
  tools.hint(`Tracking <b>${c.id}</b>: ${compass16(to)} at ${Math.round(kmh*0.621371)} mph. Hover the map for ETA.`);
}

/* ================= point history (value over the loop) ================= */
function pointHistory(ll){
  const pts=state.frames.map(f=>{ const d=f.data; if(!d) return null; const p=sample(d,L.latLng(ll[0],ll[1])); if(!p||p.b<=0) return {t:+f.t,v:NaN}; const T=decodeTable(d.dec); return {t:+f.t,v:T.rf[p.b]?NaN:T.v[p.b]}; }).filter(Boolean);
  const ok=pts.filter(p=>!isNaN(p.v)), P_=PAL[famById(state.prod).pal];
  if(ok.length<2){ popup(ll,`<span class="tag" style="background:#ffb347;color:#1a1208">HISTORY</span><h4>Not enough data here</h4><div class="when">Load more frames or pick a point with echo.</div>`); return; }
  const W=280,H=110,pad=24, t0=pts[0].t, t1=pts[pts.length-1].t||t0+1, lo=Math.min(...ok.map(p=>p.v)), hi=Math.max(...ok.map(p=>p.v)), span=(hi-lo)||1;
  const X=t=>pad+(t-t0)/(t1-t0||1)*(W-pad-6), Y=v=>H-16-(v-lo)/span*(H-30);
  let path='',pen=false; for(const p of pts){ if(isNaN(p.v)){ pen=false; continue; } path+=(pen?'L':'M')+X(p.t).toFixed(1)+','+Y(p.v).toFixed(1); pen=true; }
  const cur=+curFrame().t, fmt=v=>P_.cat?(hcaClass(v)||[0,0,'—'])[2]:P_.fmt(v), last=ok[ok.length-1], mx=ok.reduce((a,b)=>b.v>a.v?b:a);
  const svg=`<svg width="${W}" height="${H}" style="display:block;margin-top:6px"><line x1="${pad}" x2="${W-6}" y1="${H-16}" y2="${H-16}" stroke="rgba(255,236,210,.18)"/><line x1="${X(cur)}" x2="${X(cur)}" y1="6" y2="${H-16}" stroke="rgba(255,179,71,.6)" stroke-dasharray="3 3"/><path d="${path}" fill="none" stroke="#ffb347" stroke-width="2"/>${ok.map(p=>`<circle cx="${X(p.t).toFixed(1)}" cy="${Y(p.v).toFixed(1)}" r="1.8" fill="#ffb347"/>`).join('')}<text x="2" y="${Y(hi)+4}" fill="#8d9099" font-size="9" font-family="IBM Plex Mono">${Math.round(hi)}</text><text x="2" y="${Y(lo)+4}" fill="#8d9099" font-size="9" font-family="IBM Plex Mono">${Math.round(lo)}</text><text x="${pad}" y="${H-3}" fill="#8d9099" font-size="9" font-family="IBM Plex Mono">${zTime(new Date(t0))}</text><text x="${W-6}" y="${H-3}" fill="#8d9099" font-size="9" text-anchor="end" font-family="IBM Plex Mono">${zTime(new Date(t1))}</text></svg>`;
  popup(ll,`<span class="tag" style="background:#ffb347;color:#1a1208">HISTORY</span><h4>${famById(state.prod).name}</h4><div class="when">${ok.length} of ${pts.length} frames with data</div>${dl([['Latest',`<b>${fmt(last.v)}</b>`],['Peak',`${fmt(mx.v)} at ${zTime(new Date(mx.t))}`]])}${svg}`);
}

/* ================= share link (#hash) ================= */
function shareLink(){
  const c=map.getCenter(), h=`s=${state.site?state.site.id:''}&p=${state.prod}&t=${state.tilt}&src=${state.src}&c=${c.lat.toFixed(4)},${c.lng.toFixed(4)}&z=${map.getZoom().toFixed(2)}${split.on?'&b='+split.ctx.prod:''}`;
  try{ history.replaceState(null,'','#'+h); }catch(e){ location.hash=h; }
  const url=location.href; (navigator.clipboard?navigator.clipboard.writeText(url):Promise.reject()).then(()=>toast('View link copied')).catch(()=>{ tools.hint(`<input id="shareIn" value="${url.replace(/"/g,'&quot;')}" readonly>`); const i=document.getElementById('shareIn'); i.select(); });
}
function readHash(){ const h=new URLSearchParams(location.hash.slice(1)); if(!h.get('s')) return null; const c=(h.get('c')||'').split(',').map(Number);
  return {site:h.get('s'),prod:h.get('p'),tilt:+h.get('t')||0,src:h.get('src'),c:c.length===2&&!c.some(isNaN)?c:null,z:+h.get('z')||null,b:h.get('b')}; }

/* ================= tool panel additions + settings wiring ================= */
TOOL_DEFS[1][1].push(['history','Point history','Y','mode'],['xsect','Cross section','\\','mode']);
TOOL_DEFS[5][1].push(['record','Record loop video','⇧V','action']);
TOOL_DEFS[5][1].unshift(['split','Split screen','2','toggle']);
TOOL_DEFS[5][1].push(['share','Copy view link','.','action']);
$('#toolsPanel').innerHTML=TOOL_DEFS.map(([g,items])=>`<h3>${g}</h3><div class="tg">${items.map(([id,lab,k])=>`<button data-x="${id}" aria-pressed="false"><span>${lab}</span><kbd>${k}</kbd></button>`).join('')}</div>`).join('');
tools.hints.history='Click a point to chart its value across the loop.';
tools.hints.xsect='Level II: click the start (A), then the end (B) of the cross section.';
const _runTool=runTool;
runTool=function(id){ if(id==='split') return split.toggle(); if(id==='share') return shareLink(); if(id==='record') return recordLoop(); return _runTool(id); };
const _extClick=tools.extClick.bind(tools);
tools.extClick=function(ll){
  if(this.active==='history') return pointHistory(ll);
  if(this.active==='xsect'){ if(!this.tmp.length){ this.clearTmp(); this.tmp=[ll]; this.dot(ll,'#fff',this.tmpG()); this.hint('Now click the end point (B).'); return; } const A=this.tmp[0]; this.clearTmp(); this.hint(this.hints.xsect); xsBuild(A,ll); return; }
  return _extClick(ll); };
document.addEventListener('keydown',e=>{ if(e.target.matches('input,select,textarea')||e.ctrlKey||e.metaKey||e.altKey) return;
  if(e.key==='2'){ e.preventDefault(); split.toggle(); } else if(e.key==='\\'){ e.preventDefault(); runTool('xsect'); } else if(e.key==='V'&&e.shiftKey){ e.preventDefault(); recordLoop(); } else if(e.key==='.'){ shareLink(); } else if(e.key.toLowerCase()==='y'){ runTool('history'); } });

state.minRef=store.get('minRef',-10); state.hideRF=store.get('hideRF',false); state.utc=store.get('utc',false); state.playMode=store.get('playMode','loop'); state.dwell=store.get('dwell',1); state.alarm=store.get('alarm',false); state.spc=store.get('spc',false);
for(const [sel,kind] of [['#palRef','REF'],['#palVel','VEL']]){ const el=$(sel), cur=store.get('pal'+kind,'Echo Desk'); el.innerHTML=Object.keys(PALETTES[kind]).map(n=>`<option ${n===cur?'selected':''}>${n}</option>`).join(''); if(cur!=='Echo Desk') applyPalette(kind,cur);
  el.onchange=()=>{ applyPalette(kind,el.value); store.set('pal'+kind,el.value); drawLegend(famById(state.prod).pal); radar.drawNow(); if(split.radar){ split.ui(); split.radar.drawNow(); } tools.refreshPins&&tools.refreshPins(); }; }
const refreshLUTs=()=>{ lutVer++; radar.drawNow(); if(split.radar) split.radar.drawNow(); };
$('#minRef').value=state.minRef; $('#minRefOut').textContent=state.minRef<=-10?'off':state.minRef+' dBZ';
$('#minRef').oninput=e=>{ state.minRef=+e.target.value; store.set('minRef',state.minRef); $('#minRefOut').textContent=state.minRef<=-10?'off':state.minRef+' dBZ'; refreshLUTs(); };
$('#lyHideRF').checked=state.hideRF; $('#lyHideRF').onchange=e=>{ state.hideRF=e.target.checked; store.set('hideRF',state.hideRF); refreshLUTs(); };
$('#lyUTC').checked=state.utc; $('#lyUTC').onchange=e=>{ state.utc=e.target.checked; store.set('utc',state.utc); showFrame(state.idx); };
$('#playMode').value=state.playMode; $('#playMode').onchange=e=>{ state.playMode=e.target.value; store.set('playMode',state.playMode); rockDir=1; };
$('#dwell').value=Math.round(state.dwell*2); $('#dwellOut').textContent=state.dwell+' s';
$('#dwell').oninput=e=>{ state.dwell=+e.target.value/2; store.set('dwell',state.dwell); $('#dwellOut').textContent=state.dwell+' s'; };
$('#lyAlarm').checked=state.alarm; $('#lyAlarm').onchange=e=>alarm.enable(e.target.checked);
$('#lySPC').checked=state.spc; $('#lySPC').onchange=e=>spc.set(e.target.checked); if(state.spc) spc.load(true);
setInterval(()=>{ if(state.spc&&!document.hidden) spc.load(false); },5*60e3);
const HASH0=readHash();
$('#lySat').checked=sat.on; $('#satCh').value=sat.ch;
$('#lySat').onchange=e=>{ sat.on=e.target.checked; store.set('sat',sat.on); satSet(); };
$('#satCh').onchange=e=>{ sat.ch=e.target.value; store.set('satCh',sat.ch); if(sat.on) satSet(); };
$('#lyOpera').checked=opera.on; $('#lyOpera').onchange=e=>operaSet(e.target.checked);
$('#lyMosaic').checked=mosaic.on; $('#lyMosaic').onchange=e=>{ mosaic.on=e.target.checked; store.set('mosaic',mosaic.on); mosaicSet(); };
document.getElementById('playBtn').addEventListener('click',()=>setTimeout(mosaicOnFrame,0));

/* ================= vertical cross section (Level II volume) ================= */
const xs={vol:null,volKey:'',a:null,b:null,group:null,busy:false};
function xsOpen(){ let p=document.getElementById('xsPanel'); if(p) return p;
  p=document.createElement('section'); p.id='xsPanel'; p.className='panel';
  p.innerHTML=`<header><b id="xsTitle">Cross section</b><button id="xsRefresh" title="Rebuild for the current frame">Current frame</button><button id="xsClose">Close</button></header><canvas id="xsCanvas" width="560" height="260"></canvas><div id="xsNote"></div>`;
  document.body.appendChild(p);
  document.getElementById('xsClose').onclick=()=>{ p.remove(); if(xs.group){ toolLayer.removeLayer(xs.group); xs.group=null; } };
  document.getElementById('xsRefresh').onclick=()=>xs.a&&xsBuild(xs.a,xs.b);
  return p; }
async function xsBuild(A,B){
  const f=curFrame(), s=state.site, fam=famById(state.prod);
  if(!f||!(f.type==='l2'||f.type==='l2c')){ toast('Cross sections need Level II volumes. Switch the data level to LVL II.',5000); return; }
  if(!fam.l2){ toast(`${fam.name} isn't in Level II. Use REF, VEL, SW, ZDR or CC.`); return; }
  xs.a=A; xs.b=B; xsOpen(); const note=document.getElementById('xsNote'), title=document.getElementById('xsTitle');
  title.textContent=`${fam.name} · ${s.id} · ${zTime(f.t)}`;
  if(xs.group) toolLayer.removeLayer(xs.group); xs.group=L.layerGroup().addTo(toolLayer);
  tools.line([A,B],'#fff',xs.group); tools.label(A,'A','','top',xs.group); tools.label(B,'B','','top',xs.group);
  const key=`${f.url}|${fam.l2}`;
  if(xs.volKey!==key){ note.textContent='Loading the full volume…'; xs.busy=true;
    try{ const v=await pool.run('l2vol',f.url,{block:fam.l2,urls:f.urls},120,'xs'); v.pal=fam.pal; xs.vol=v; xs.volKey=key; }
    catch(e){ note.textContent='Could not load the volume: '+e.message; xs.busy=false; return; }
    xs.busy=false; }
  xsRender();
}
function xsRender(){
  const v=xs.vol, A=xs.a, B=xs.b, cv=document.getElementById('xsCanvas'); if(!v||!cv) return;
  const x=cv.getContext('2d'), W=cv.width, H=cv.height, padL=44, padB=26, padT=8, padR=8, pw=W-padL-padR, ph=H-padB-padT;
  const totKm=distKm(A,B), mi=state.units==='mi', topKm=mi?50/3.28084:15, NX=Math.min(pw,320), NY=Math.min(ph,180);
  const o=[v.lat,v.lon], luts=v.tilts.map(t=>buildLUT(t.dec,v.pal)), bw=0.5*Math.PI/180;
  const img=x.createImageData(NX,NY), px=img.data;
  for(let i=0;i<NX;i++){
    const fr=i/(NX-1), P=[A[0]+(B[0]-A[0])*fr,A[1]+(B[1]-A[1])*fr], sKm=distKm(o,P); let az=bearingDeg(o,P);
    const row=((az/0.5)|0)%720;
    const hs=v.tilts.map(t=>{ const r=sKm/Math.cos(t.elev*Math.PI/180); return {r,h:beamHeightKm(r,t.elev),hw:r*Math.tan(bw)}; });
    const codeAt=k=>{ const t=v.tilts[k], g=((hs[k].r-t.firstKm)/t.gateKm)|0; return g<0||g>=t.nGates?0:t.grid[row*t.nGates+g]; };
    const codes=hs.map((_,k)=>codeAt(k)), nT=hs.length;
    for(let j=0;j<NY;j++){
      const h=topKm*(1-(j+.5)/NY); let k=-1; for(let q=0;q<nT;q++){ if(hs[q].h<=h) k=q; else break; }
      let c=0, li=0;
      if(k<0){ if(h>=hs[0].h-hs[0].hw){ c=codes[0]; li=0; } }
      else if(k>=nT-1){ if(h<=hs[nT-1].h+hs[nT-1].hw){ c=codes[nT-1]; li=nT-1; } }
      else { const c0=codes[k], c1=codes[k+1], f=(h-hs[k].h)/(hs[k+1].h-hs[k].h||1);
        if(c0>=2&&c1>=2&&v.pal!=='HCA'){ c=Math.round(c0+(c1-c0)*f); li=f<.5?k:k+1; }
        else if(c0&&(h-hs[k].h)<=hs[k].hw*1.2){ c=c0; li=k; } else if(c1&&(hs[k+1].h-h)<=hs[k+1].hw*1.2){ c=c1; li=k+1; } }
      if(!c) continue; const L4=luts[li], q=(j*NX+i)*4;
      px[q]=L4[c*4]; px[q+1]=L4[c*4+1]; px[q+2]=L4[c*4+2]; px[q+3]=L4[c*4+3];
    }
  }
  x.clearRect(0,0,W,H); x.fillStyle='#07090c'; x.fillRect(0,0,W,H);
  const tmp=document.createElement('canvas'); tmp.width=NX; tmp.height=NY; tmp.getContext('2d').putImageData(img,0,0);
  x.imageSmoothingEnabled=state.smooth; x.drawImage(tmp,padL,padT,pw,ph);
  // axes
  x.strokeStyle='rgba(255,236,210,.18)'; x.fillStyle='#8d9099'; x.font='500 10px "IBM Plex Mono",monospace'; x.lineWidth=1;
  const stepH=mi?10:3; for(let hv=0;hv<=(mi?50:15)+1e-6;hv+=stepH){ const y=padT+ph*(1-hv/(mi?50:15)); x.beginPath(); x.moveTo(padL,y); x.lineTo(W-padR,y); x.stroke(); x.textAlign='right'; x.fillText(`${hv}${mi?'k':''}`,padL-5,y+3); }
  x.save(); x.translate(10,padT+ph/2); x.rotate(-Math.PI/2); x.textAlign='center'; x.fillText(mi?'height ft':'height km',0,0); x.restore();
  const [k,u]=UNITS[state.units], totU=totKm*k, stepD=totU>150?50:totU>60?20:totU>25?10:5; x.textAlign='center';
  for(let dv=0;dv<=totU+1e-6;dv+=stepD){ const xx=padL+pw*dv/totU; x.beginPath(); x.moveTo(xx,padT+ph); x.lineTo(xx,padT+ph+4); x.stroke(); x.fillText(`${dv}`,xx,H-12); }
  x.fillText(u,W-padR-10,H-2); x.fillStyle='#fff'; x.font='700 11px "IBM Plex Mono",monospace'; x.textAlign='left'; x.fillText('A',padL+3,padT+12); x.textAlign='right'; x.fillText('B',W-padR-3,padT+12);
  document.getElementById('xsNote').textContent=`${v.tilts.length} tilts · ${v.tilts.map(t=>t.elev.toFixed(1)).join(' ')}° · ${fmtDist(totKm)} long · heights above radar (4/3 earth)`;
}

/* ================= storm cell trends (from every NST in the loop) ================= */
function cellTrend(id){
  if(!state.frames.length) return [];
  const t0=+state.frames[0].t-15*60e3, t1=+state.frames[state.frames.length-1].t+60e3, out=[];
  for(const e of cellIndex.nst){ if(e.t<t0||e.t>t1) continue; const c=cellCache.get(e.url);
    if(!c){ loadCellFile(e,8); continue; } if(!c.data) continue;
    const st=c.data.storms.find(q=>q.id===id); if(st&&st.dbzm!=null) out.push({t:c.data.time||e.t,dbz:st.dbzm,hgt:st.hgt}); }
  return out.sort((a,b)=>a.t-b.t);
}
function trendSVG(pts){
  if(pts.length<2) return `<div style="color:var(--mute);font-size:11.5px;margin-top:6px">Trend builds as more SCIT products load.</div>`;
  const W=260,H=84,pl=26,pr=26,t0=pts[0].t,t1=pts[pts.length-1].t||t0+1,X=t=>pl+(t-t0)/(t1-t0||1)*(W-pl-pr);
  const dz=pts.map(p=>p.dbz), hz=pts.map(p=>p.hgt).filter(v=>v!=null), dlo=Math.min(...dz)-2, dhi=Math.max(...dz)+2, hlo=hz.length?Math.min(...hz)-1:0, hhi=hz.length?Math.max(...hz)+1:1;
  const Yd=v=>H-14-(v-dlo)/(dhi-dlo||1)*(H-24), Yh=v=>H-14-(v-hlo)/(hhi-hlo||1)*(H-24);
  const path=(f,sel)=>pts.filter(p=>sel(p)!=null).map((p,i)=>(i?'L':'M')+X(p.t).toFixed(1)+','+f(sel(p)).toFixed(1)).join('');
  return `<svg width="${W}" height="${H}" style="display:block;margin-top:6px"><path d="${path(Yd,p=>p.dbz)}" fill="none" stroke="#ff7a7a" stroke-width="2"/><path d="${path(Yh,p=>p.hgt)}" fill="none" stroke="#5cc8ff" stroke-width="1.6" stroke-dasharray="4 3"/><text x="2" y="12" fill="#ff7a7a" font-size="9" font-family="IBM Plex Mono">${Math.round(dhi)}</text><text x="2" y="${H-14}" fill="#ff7a7a" font-size="9" font-family="IBM Plex Mono">${Math.round(dlo)}</text><text x="${W-2}" y="12" fill="#5cc8ff" font-size="9" text-anchor="end" font-family="IBM Plex Mono">${hhi.toFixed(0)}k</text><text x="${pl}" y="${H-2}" fill="#8d9099" font-size="9" font-family="IBM Plex Mono">${zTime(new Date(t0))}</text><text x="${W-pr}" y="${H-2}" fill="#8d9099" font-size="9" text-anchor="end" font-family="IBM Plex Mono">${zTime(new Date(t1))}</text></svg><div style="font:500 10px var(--mono);color:var(--mute)"><span style="color:#ff7a7a">━</span> max dBZ &nbsp; <span style="color:#5cc8ff">┅</span> height of max (kft)</div>`;
}

/* ================= GOES satellite + national radar mosaic (IEM WMS) ================= */
const sat={layer:null,ch:store.get('satCh','ch13'),on:store.get('sat',false),at:0};
const mosaic={layer:null,on:store.get('mosaic',false),time:null};
function satSet(){
  if(sat.layer){ map.removeLayer(sat.layer); sat.layer=null; } if(!sat.on) return;
  if(!map.getPane('sat')){ const p=map.createPane('sat'); p.style.zIndex=250; p.style.pointerEvents='none'; }
  const west=state.site&&state.site.lon<-105, url=`https://mesonet.agron.iastate.edu/cgi-bin/wms/${west?'goes_west':'goes_east'}.cgi`;
  sat.layer=L.tileLayer.wms(url,{layers:`conus_${sat.ch}`,format:'image/png',transparent:true,version:'1.1.1',tileSize:512,pane:'sat',opacity:sat.ch==='ch02'?.75:.65,crossOrigin:'anonymous',attribution:'GOES via IEM',_t:Math.floor(Date.now()/300e3)}).addTo(map);
  sat.at=Date.now();
}
function mosaicTime(){ const f=curFrame(), t=f?+f.t:Date.now(); return new Date(Math.floor(t/300e3)*300e3).toISOString().slice(0,16)+'Z'; }
function mosaicSet(){
  if(mosaic.layer){ map.removeLayer(mosaic.layer); mosaic.layer=null; } if(!mosaic.on) return;
  if(!map.getPane('mosaic')){ const p=map.createPane('mosaic'); p.style.zIndex=340; p.style.pointerEvents='none'; }
  mosaic.time=mosaicTime();
  mosaic.layer=L.tileLayer.wms('https://mesonet.agron.iastate.edu/cgi-bin/wms/nexrad/n0q-t.cgi',{layers:'nexrad-n0q-wmst',format:'image/png',transparent:true,version:'1.1.1',tileSize:512,pane:'mosaic',opacity:.5,time:mosaic.time,crossOrigin:'anonymous',attribution:'NEXRAD mosaic via IEM'}).addTo(map);
}
function mosaicOnFrame(){ if(!mosaic.on||!mosaic.layer||state.playing) return; const t=mosaicTime(); if(t!==mosaic.time){ mosaic.time=t; mosaic.layer.setParams({time:t}); } }
setInterval(()=>{ if(sat.on&&!document.hidden&&Date.now()-sat.at>5*60e3){ sat.at=Date.now(); sat.layer&&sat.layer.setParams({_t:Math.floor(Date.now()/300e3)}); } },60e3);

/* ================= OPERA European composite (ODIM HDF5 → Mercator image) ================= */
const opera={on:store.get('opera',false),layer:null,index:[],at:0,cache:new Map(),cur:null,busy:false};
async function operaIndex(){
  if(Date.now()-opera.at<5*60e3&&opera.index.length) return opera.index;
  const out=[];
  for(const d of [new Date(Date.now()-86400e3),new Date()]){ const [Y,M,D]=ymdU(d);
    const keys=await cachedList(OPERA_BUCKET,`${Y}/${M}/${D}/OPERA/COMP/`,120e3);
    for(const k of keys){ const m=k.match(/OPERA@(\d{8})T(\d{4})@0@DBZH\.h5$/); if(m) out.push({t:Date.UTC(+m[1].slice(0,4),m[1].slice(4,6)-1,+m[1].slice(6,8),+m[2].slice(0,2),+m[2].slice(2,4)),url:`${OPERA_BUCKET}/${k.split('/').map(encodeURIComponent).join('/')}`}); } }
  opera.index=out.sort((a,b)=>a.t-b.t); opera.at=Date.now(); return opera.index;
}
async function operaShow(force){
  if(!opera.on||opera.busy) return;
  opera.busy=true;
  try{
    const idx=await operaIndex(); if(!idx.length) throw new Error('no composites listed');
    const f=curFrame(), t=f&&!state.playing?+f.t:Date.now(); let e=idx[0]; for(const x of idx) if(x.t<=t+60e3) e=x;
    if(!force&&opera.cur===e.url) return;
    let c=opera.cache.get(e.url);
    if(!c){ c=await pool.run('operacomp',e.url,{w:1800},30,'opera'); opera.cache.set(e.url,c); if(opera.cache.size>6) opera.cache.delete(opera.cache.keys().next().value); }
    const cv=document.createElement('canvas'); cv.width=c.w; cv.height=c.h; const x=cv.getContext('2d'), img=x.createImageData(c.w,c.h), lut=buildLUT({t:'so',scale:2,offset:66,lead:2,mul:1,rf:false},'REF');
    for(let i=0;i<c.codes.length;i++){ const k=c.codes[i]; if(!k) continue; const q=i*4; img.data[q]=lut[k*4]; img.data[q+1]=lut[k*4+1]; img.data[q+2]=lut[k*4+2]; img.data[q+3]=lut[k*4+3]; }
    x.putImageData(img,0,0);
    if(!map.getPane('mosaic')){ const p=map.createPane('mosaic'); p.style.zIndex=340; p.style.pointerEvents='none'; }
    const url=cv.toDataURL('image/png');
    if(opera.layer){ opera.layer.setUrl(url); opera.layer.setBounds(L.latLngBounds(c.bounds)); }
    else opera.layer=L.imageOverlay(url,c.bounds,{pane:'mosaic',opacity:.75,attribution:'OPERA composite © EUMETNET'}).addTo(map);
    opera.cur=e.url; $('#operaSrc').textContent=zTime(new Date(e.t));
  }catch(err){ console.warn('OPERA',err); toast(`OPERA composite unavailable (${/fetch|network|cors/i.test(err.message)?'the server blocks browser requests':err.message}).`,6000); opera.on=false; $('#lyOpera').checked=false; store.set('opera',false); }
  finally{ opera.busy=false; }
}
function operaSet(on){ opera.on=on; store.set('opera',on); if(on) operaShow(true); else { if(opera.layer){ map.removeLayer(opera.layer); opera.layer=null; } opera.cur=null; $('#operaSrc').textContent='EUMETNET'; } }
setInterval(()=>{ if(opera.on&&!document.hidden) operaShow(false); },3*60e3);

/* ================= boot ================= */
setBasemap(store.get('bm','dark'));
if(!CATALOG.some(f=>f.id===state.prod)) state.prod='REF';
drawLegend(famById(state.prod).pal);
loadSites().then(()=>{ const H0=HASH0; if(H0){ if(H0.prod&&CATALOG.some(f=>f.id===H0.prod)) state.prod=H0.prod; if(H0.src) state.src=H0.src; state.tilt=H0.tilt; if(H0.site) store.set('site',H0.site); }
  const id=store.get('site','KTLX'); selectSite(SITES.find(s=>s.id===id)?id:'KTLX',false); const s=state.site; if(s) map.setView([s.lat,s.lon],s.type==='TDWR'?8:7);
  if(H0&&H0.c) map.setView(H0.c,H0.z||map.getZoom());
  if(sat.on) satSet(); if(mosaic.on) mosaicSet(); if(opera.on) operaShow(true);
  if((H0&&H0.b)||store.get('split',false)){ if(H0&&H0.b) store.set('splitProd',H0.b); split.toggle(true); } });
</script>
</body>
</html>
