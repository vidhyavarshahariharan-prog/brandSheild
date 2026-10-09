import { useEffect, useState } from 'react';
import { ResponsiveContainer, BarChart, Bar, XAxis, YAxis, Tooltip, LineChart, Line, ScatterChart, Scatter, CartesianGrid, ZAxis, Cell } from 'recharts';
import { Shield, Sun, Moon, AlertTriangle, Radar, Network, Layers, ListOrdered, LayoutDashboard, Play } from 'lucide-react';

const api = async (p: string, o: any = {}) => {
  const r = await fetch('/api' + p, { ...o, headers: { 'Content-Type': 'application/json', Authorization: 'Bearer ' + localStorage.getItem('t') }, body: o.body && JSON.stringify(o.body) });
  if (r.status === 401 && localStorage.getItem('t')) { localStorage.removeItem('t'); location.reload(); }
  return r.json();
};
const COL: any = { Critical: '#dc2626', High: '#f97316', Medium: '#eab308', Low: '#22c55e' };
const card = 'rounded-lg border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-800 p-4';
const STATUSES = ['New', 'Under Review', 'Confirmed Fraud', 'False Positive', 'Reported', 'Monitoring', 'Resolved'];
const Badge = ({ l }: any) => <span className="rounded px-2 py-0.5 text-xs font-semibold text-white" style={{ background: COL[l] }}>{l}</span>;
const Bar2 = ({ label, v, detail }: any) => (
  <div className="mb-2"><div className="flex justify-between text-sm"><span>{label}</span><b>{v}%</b></div>
    <div className="h-2 rounded bg-slate-200 dark:bg-slate-700"><div className="h-2 rounded" style={{ width: v + '%', background: v >= 81 ? COL.Critical : v >= 61 ? COL.High : v >= 31 ? COL.Medium : COL.Low }} /></div>
    {detail && <div className="text-xs text-slate-500 dark:text-slate-400">{detail}</div>}</div>
);

function Login({ done }: any) {
  const demo = async () => { const r = await api('/auth/demo', { method: 'POST', body: {} }); localStorage.setItem('t', r.token); done(); };
  return (
    <div className="min-h-screen grid place-items-center bg-slate-900 text-white p-6">
      <div className="max-w-sm w-full text-center">
        <Shield className="mx-auto mb-3 text-cyan-400" size={48} />
        <h1 className="text-3xl font-bold">BrandShield X</h1>
        <p className="mt-1 text-slate-400">Detect. Verify. Connect. Prioritize. Protect.</p>
        <button onClick={demo} className="mt-6 w-full rounded bg-cyan-500 py-2 font-semibold text-slate-900">Try demo login</button>
      </div>
    </div>
  );
}

function Dashboard({ open }: any) {
  const [d, setD] = useState<any>(null); const [busy, setBusy] = useState(false);
  const load = () => api('/dashboard').then(setD); useEffect(() => { load(); }, []);
  const scan = async () => { setBusy(true); await api('/scans/' + d.brandId, { method: 'POST', body: {} }); await load(); setBusy(false); };
  if (!d) return <p>Loading…</p>;
  const stat = (k: string, v: any) => <div className={card}><div className="text-2xl font-bold">{v}</div><div className="text-sm text-slate-500">{k}</div></div>;
  return (
    <div className="space-y-4">
      <div className="flex flex-wrap items-center justify-between gap-2">
        <h2 className="text-xl font-bold">Command Center – {d.brandName}</h2>
        <button onClick={scan} disabled={busy} className="flex items-center gap-2 rounded bg-cyan-600 px-4 py-2 text-white"><Play size={16} />{busy ? 'Scanning…' : 'Run AI Intelligence Scan'}</button>
      </div>
      <div className="grid grid-cols-2 gap-3 md:grid-cols-4">
        {stat('Brand Health Score', d.brandHealth)}{stat('Monitored assets', d.monitored)}{stat('Official assets', d.official)}{stat('Fraud Suspect Alerts', d.alerts)}
        {stat('Suspicious social', d.suspiciousSocial)}{stat('Suspicious apps', d.suspiciousApps)}{stat('Avg Conformability', d.avgConformability + '%')}
        {stat('Confirmed / FP / Resolved', `${d.confirmed} / ${d.falsePositives} / ${d.resolved}`)}
      </div>
      <div className="grid gap-4 md:grid-cols-2">
        <div className={card}><h3 className="mb-2 font-semibold">Threats by level</h3><ResponsiveContainer height={200}><BarChart data={d.levels}><XAxis dataKey="level" /><YAxis allowDecimals={false} /><Tooltip /><Bar dataKey="count">{d.levels.map((l: any) => <Cell key={l.level} fill={COL[l.level]} />)}</Bar></BarChart></ResponsiveContainer></div>
        <div className={card}><h3 className="mb-2 font-semibold">Threat trend</h3><ResponsiveContainer height={200}><LineChart data={d.trend}><XAxis dataKey="date" /><YAxis allowDecimals={false} /><Tooltip /><Line dataKey="count" stroke="#06b6d4" /></LineChart></ResponsiveContainer></div>
      </div>
      <div className={card}><h3 className="mb-2 font-semibold">Latest AI alerts</h3>
        {d.latest.length === 0 && <p className="text-sm text-slate-500">No alerts yet. Run the AI Intelligence Scan.</p>}
        {d.latest.map((a: any) => <Row key={a.id} t={a.threat} open={open} />)}</div>
    </div>
  );
}
const Row = ({ t, open, rank }: any) => (
  <div onClick={() => open(t.id)} className="mb-2 flex cursor-pointer flex-wrap items-center justify-between gap-2 rounded border border-slate-200 p-3 hover:bg-slate-50 dark:border-slate-700 dark:hover:bg-slate-700" style={{ borderLeft: `4px solid ${COL[t.level]}` }}>
    <div><div className="font-semibold">{rank !== undefined && <span className="mr-2 text-cyan-500">#{rank + 1}{rank === 0 ? ' Investigate first' : ''}</span>}{t.assetName} <span className="text-xs text-slate-500">{t.platform}</span></div>
      <div className="text-xs text-slate-500">{t.status}</div></div>
    <div className="flex items-center gap-3 text-sm"><span>Fraud {t.fraudScore}%</span><span>Conf {t.conformability}%</span><Badge l={t.level} /></div>
  </div>
);

function Alerts({ open }: any) {
  const [list, setList] = useState<any[]>([]); const [f, setF] = useState({ level: '', kind: '', status: '' });
  useEffect(() => { api('/threats?' + new URLSearchParams(Object.entries(f).filter(([, v]) => v) as any)).then(setList); }, [f]);
  const sel = (k: string, opts: string[]) => <select className="rounded border bg-transparent p-1" value={(f as any)[k]} onChange={(e) => setF({ ...f, [k]: e.target.value })}><option value="">{k}: all</option>{opts.map((o) => <option key={o}>{o}</option>)}</select>;
  return (
    <div className="space-y-3"><h2 className="text-xl font-bold">Fraud Suspect Alerts</h2>
      <div className="flex flex-wrap gap-2">{sel('level', ['Critical', 'High', 'Medium', 'Low'])}{sel('kind', ['SOCIAL', 'APP'])}{sel('status', STATUSES)}</div>
      {list.map((t) => (
        <div key={t.id} className={card + ' cursor-pointer'} onClick={() => open(t.id)} style={{ borderLeft: `6px solid ${COL[t.level]}` }}>
          {t.fraudScore >= 60 && <div className="mb-1 flex items-center gap-1 font-bold text-red-600"><AlertTriangle size={16} />⚠ FRAUD SUSPECT ALERT – High Probability of Brand Impersonation</div>}
          <div className="font-semibold">{t.assetName} · {t.platform}</div>
          <div className="text-sm">Suspect {t.fraudScore}% · Conformability {t.conformability}% · <Badge l={t.level} /> · {t.status} · {new Date(t.createdAt).toLocaleString()}</div>
          <div className="mt-1 text-sm text-slate-500">{t.reason}</div><div className="text-sm">Recommended: {t.action}</div>
        </div>))}
      {!list.length && <p className="text-slate-500">No alerts match. Run a scan from the Command Center.</p>}
    </div>
  );
}

function Queue({ open }: any) {
  const [l, setL] = useState<any[]>([]); useEffect(() => { api('/threats').then(setL); }, []);
  return <div><h2 className="mb-3 text-xl font-bold">Priority Queue</h2>{l.filter((t) => !['Resolved', 'False Positive'].includes(t.status)).map((t, i) => <Row key={t.id} t={t} open={open} rank={i} />)}</div>;
}
function RadarPage({ open }: any) {
  const [l, setL] = useState<any[]>([]); useEffect(() => { api('/threats').then(setL); }, []);
  const rk: any = { Low: 25, Medium: 50, High: 75, Critical: 95 };
  const data = l.map((t) => ({ ...t, x: t.fraudScore, y: Math.round((rk[t.customerRisk] + rk[t.brandRisk] + rk[t.operationalRisk]) / 3) }));
  return (
    <div className={card}><h2 className="mb-2 text-xl font-bold">Threat Radar – Probability vs Impact</h2>
      <ResponsiveContainer height={420}><ScatterChart><CartesianGrid /><XAxis type="number" dataKey="x" name="Probability" domain={[0, 100]} /><YAxis type="number" dataKey="y" name="Impact" domain={[0, 100]} /><ZAxis range={[220, 220]} /><Tooltip formatter={(v: any) => v} labelFormatter={() => ''} />
        <Scatter data={data} onClick={(d: any) => open(d.id)}>{data.map((d) => <Cell key={d.id} fill={COL[d.level]} />)}</Scatter></ScatterChart></ResponsiveContainer>
      <p className="text-xs text-slate-500">Top-right = high probability and high impact. Click a point to investigate.</p></div>
  );
}
function Graph({ open }: any) {
  const [g, setG] = useState<any>({ nodes: [], edges: [] }); useEffect(() => { api('/graph').then(setG); }, []);
  const pos: any = {}; g.nodes.forEach((n: any, i: number) => { const a = (2 * Math.PI * i) / g.nodes.length, R = n.type === 'DOMAIN' || n.type === 'DEVELOPER' ? 90 : 180; pos[n.id] = [300 + R * Math.cos(a), 220 + R * Math.sin(a)]; });
  return (
    <div className={card}><h2 className="mb-2 text-xl font-bold">Threat Graph</h2>
      <svg viewBox="0 0 600 440" className="w-full">
        {g.edges.map((e: any, i: number) => pos[e.from] && pos[e.to] && <line key={i} x1={pos[e.from][0]} y1={pos[e.from][1]} x2={pos[e.to][0]} y2={pos[e.to][1]} stroke="#64748b" />)}
        {g.nodes.map((n: any) => <g key={n.id} onClick={() => (n.level ? open(n.id) : null)} className="cursor-pointer">
          <circle cx={pos[n.id][0]} cy={pos[n.id][1]} r={n.level ? 14 : 10} fill={n.level ? COL[n.level] : n.type === 'DOMAIN' ? '#6366f1' : '#06b6d4'} />
          <text x={pos[n.id][0]} y={pos[n.id][1] + 26} fontSize="9" textAnchor="middle" fill="currentColor">{n.label}</text></g>)}
      </svg>
      <p className="text-xs text-slate-500">Assets sharing a domain or developer are linked – a sign they may belong to one campaign.</p></div>
  );
}
function Campaigns() {
  const [l, setL] = useState<any[]>([]); useEffect(() => { api('/campaigns').then(setL); }, []);
  return <div className="space-y-3"><h2 className="text-xl font-bold">AI Campaign Detection</h2>{l.map((c) => <div key={c.id} className={card}><div className="flex justify-between"><b>{c.name}</b><Badge l={c.severity} /></div>
    <div className="text-sm">{c.count} connected threats · Avg suspect {c.avgFraud}% · Avg conformability {c.avgConformability}% · Since {new Date(c.createdAt).toLocaleDateString()}</div></div>)}{!l.length && <p className="text-slate-500">No campaigns detected yet.</p>}</div>;
}

function Detail({ id, back }: any) {
  const [t, setT] = useState<any>(null); const [note, setNote] = useState('');
  const load = () => api('/threats/' + id).then(setT); useEffect(() => { load(); }, [id]);
  if (!t) return <p>Loading…</p>;
  const post = async (p: string, m = 'POST', body: any = {}) => { await api(`/threats/${id}/${p}`, { method: m, body }); load(); };
  return (
    <div className="space-y-4"><button onClick={back} className="text-cyan-500">← Back</button>
      <div className={card} style={{ borderLeft: `6px solid ${COL[t.level]}` }}>
        <h2 className="text-xl font-bold">{t.assetName} <span className="text-sm text-slate-500">{t.platform} · {t.assetRef}</span></h2>
        <div className="mt-2 grid grid-cols-2 gap-3 md:grid-cols-4"><b>Fraud Suspect: {t.fraudScore}%</b><b>Conformability: {t.conformability}%</b><b>AI Confidence: {t.confidence}%</b><span>Threat: <Badge l={t.level} /></span></div>
        <div className="mt-2 text-sm">Customer <Badge l={t.customerRisk} /> Brand <Badge l={t.brandRisk} /> Operational <Badge l={t.operationalRisk} /></div>
        <p className="mt-2 text-sm"><b>Why flagged:</b> {t.reason}</p><p className="text-sm"><b>Recommended action:</b> {t.action}</p>
        {t.campaign && <p className="text-sm"><b>Campaign:</b> {t.campaign.name} ({t.campaign.threats.length} assets)</p>}
      </div>
      <div className="grid gap-4 md:grid-cols-2">
        <div className={card}><h3 className="mb-2 font-semibold">AI Explainability – weighted signals</h3>{t.signals.map((s: any) => <Bar2 key={s.id} label={`${s.name} (weight ${Math.round(s.weight * 100)}%)`} v={s.score} detail={s.detail} />)}</div>
        <div className={card}><h3 className="mb-2 font-semibold">Conformability Rate Analysis – {t.conformability}%</h3>
          {t.signals.filter((s: any) => s.name !== 'Developer Mismatch').map((s: any) => <Bar2 key={s.id} label={s.name} v={s.score} />)}<Bar2 label="Official Status Mismatch" v={100} />
          <p className="mt-2 text-xs text-slate-500">A high Conformability Rate means this asset strongly matches known brand-impersonation indicators. It is a risk indicator, not absolute proof of fraud – analyst verification is required.</p>
          {t.analyses[0] && <p className="mt-2 text-sm">{t.analyses[0].summary}</p>}</div>
      </div>
      <div className={card}><h3 className="mb-2 font-semibold">Action Center</h3>
        <div className="mb-2 flex flex-wrap gap-2">
          <select value={t.status} onChange={(e) => post('status', 'PATCH', { status: e.target.value })} className="rounded border bg-transparent p-1">{STATUSES.map((s) => <option key={s}>{s}</option>)}</select>
          <button className="rounded bg-slate-700 px-3 py-1 text-white" onClick={() => post('incident', 'POST', { assignee: 'Demo Analyst' })}>Create incident</button>
          <button className="rounded bg-green-600 px-3 py-1 text-white" onClick={() => post('legitimate')}>Mark as Legitimate</button>
          <button className="rounded bg-orange-600 px-3 py-1 text-white" onClick={() => post('actions', 'POST', { type: 'REPORT_SIMULATED', note: 'Simulated report to platform (no real report sent)' })}>Simulate report</button></div>
        <div className="flex gap-2"><input value={note} onChange={(e) => setNote(e.target.value)} placeholder="Add investigation note" className="flex-1 rounded border bg-transparent p-1" />
          <button className="rounded bg-cyan-600 px-3 text-white" onClick={() => { if (note) { post('actions', 'POST', { note }); setNote(''); } }}>Add note</button></div>
        <h4 className="mt-3 font-semibold">Timeline</h4>
        <ul className="text-sm">{t.actions.map((a: any) => <li key={a.id}>{new Date(a.createdAt).toLocaleString()} – {a.type}: {a.note}</li>)}<li>{new Date(t.createdAt).toLocaleString()} – First detected</li></ul></div>
    </div>
  );
}

export default function App() {
  const [authed, setAuthed] = useState(!!localStorage.getItem('t')); const [tab, setTab] = useState('dash'); const [sel, setSel] = useState<string | null>(null); const [dark, setDark] = useState(true);
  useEffect(() => { document.documentElement.classList.toggle('dark', dark); }, [dark]);
  if (!authed) return <Login done={() => setAuthed(true)} />;
  const open = (id: string) => setSel(id);
  const tabs: any[] = [['dash', 'Command Center', LayoutDashboard], ['alerts', 'Fraud Alerts', AlertTriangle], ['queue', 'Priority Queue', ListOrdered], ['radar', 'Threat Radar', Radar], ['graph', 'Threat Graph', Network], ['camp', 'Campaigns', Layers]];
  const pages: any = { dash: <Dashboard open={open} />, alerts: <Alerts open={open} />, queue: <Queue open={open} />, radar: <RadarPage open={open} />, graph: <Graph open={open} />, camp: <Campaigns /> };
  return (
    <div className="min-h-screen bg-slate-50 text-slate-900 dark:bg-slate-900 dark:text-slate-100 md:flex">
      <nav className="flex gap-1 overflow-x-auto bg-slate-900 p-3 text-slate-200 md:w-56 md:flex-col md:overflow-visible">
        <div className="mb-2 flex items-center gap-2 font-bold text-cyan-400"><Shield size={20} />BrandShield X</div>
        {tabs.map(([k, n, I]) => <button key={k} onClick={() => { setTab(k); setSel(null); }} className={`flex items-center gap-2 whitespace-nowrap rounded px-3 py-2 text-left ${tab === k && !sel ? 'bg-cyan-600 text-white' : 'hover:bg-slate-800'}`}><I size={16} />{n}</button>)}
        <button onClick={() => setDark(!dark)} className="mt-auto flex items-center gap-2 px-3 py-2">{dark ? <Sun size={16} /> : <Moon size={16} />}Theme</button>
        <button onClick={() => { localStorage.removeItem('t'); location.reload(); }} className="px-3 py-2 text-left">Log out</button>
      </nav>
      <main className="flex-1 p-4 md:p-6">
        <p className="mb-3 text-xs text-slate-500">AI assists analysts in prioritising suspicious activity. Scores are risk indicators, not proof of fraud.</p>
        {sel ? <Detail id={sel} back={() => setSel(null)} /> : pages[tab]}
      </main>
    </div>
  );
}
