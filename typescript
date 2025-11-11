import React, { useEffect, useMemo, useRef, useState } from "react";

/**
 * AmpJiew Digital Mixer — Web Bluetooth PWA (Pro UI)
 * -------------------------------------------------------------
 * Channel-strip style mixer with parametric EQ graph, dynamics, AUX sends, and master bus.
 * Works with an ESP32/MCU over BLE (GATT). UI writes normalized values; your firmware does DSP.
 *
 * CHANNELS
 *   • CH1 Mic, CH2 Mic, CH3 Music (stereo to mono mix with pan)
 *   • Per‑channel: Input Gain, HPF, 4‑band Parametric EQ (G,F,Q), Compressor (Thresh, Ratio, Attack, Release, Make‑up),
 *                  Gate (Thresh, Hold, Release), Pan, Fader, Mute, Solo, AUX1 (FX) send (pre/post)
 *   • Master: AUX1 return with Echo/Reverb controls, Limiter, Master fader, scene presets
 *   • Metering: Peak + RMS with peak‑hold (expects BLE notifications for levels; see UUIDs below)
 *
 * BLE PROTOCOL (example UUIDs — change to match your firmware)
 * Service UUID:  "8f04ebf3-5c4b-4b8e-9a9a-1a2b3c4d5e60"
 * Per‑channel base (CH1=0x10, CH2=0x20, CH3=0x30). Append offsets:
 *   +0x01 inputGain    (f32, 0..1 maps to -12..+48 dB)
 *   +0x02 pan          (f32 0=L, .5=C, 1=R)
 *   +0x03 fader        (f32 0..1 maps to -∞..+6 dB via taper)
 *   +0x04 mute         (u8 0/1)
 *   +0x05 solo         (u8 0/1)
 *   +0x06 aux1Send     (f32 0..1)
 *   +0x07 aux1PrePost  (u8 0=post,1=pre)
 *   +0x08 hpfFreqHz    (f32 20..400)
 *   +0x09 gateThresh   (f32 0..1)
 *   +0x0a gateHoldMs   (f32)
 *   +0x0b gateReleaseMs(f32)
 *   +0x0c compThresh   (f32 0..1)
 *   +0x0d compRatio    (f32 1..20)
 *   +0x0e compAttackMs (f32)
 *   +0x0f compReleaseMs(f32)
 *   +0x10 compMakeup   (f32 0..1)
 *   +0x20 eqBandData   (f32[12] = [g1,f1,q1,g2,f2,q2,g3,f3,q3,g4,f4,q4])
 *   +0xF0 meterRMS     (f32 notify 0..1)
 *   +0xF1 meterPeak    (f32 notify 0..1)
 * Master UUIDs:
 *   masterFader    "8f04ebf3-0001-4b8e-9a9a-1a2b3c4d5e60"
 *   limiterEnable  "8f04ebf3-0002-4b8e-9a9a-1a2b3c4d5e60"
 *   limiterThresh  "8f04ebf3-0003-4b8e-9a9a-1a2b3c4d5e60"
 *   aux1Level      "8f04ebf3-0004-4b8e-9a9a-1a2b3c4d5e60"
 *   aux1DelayMs    "8f04ebf3-0005-4b8e-9a9a-1a2b3c4d5e60"
 *   aux1Feedback   "8f04ebf3-0006-4b8e-9a9a-1a2b3c4d5e60"
 *   aux1ReverbTime "8f04ebf3-0007-4b8e-9a9a-1a2b3c4d5e60"
 *   sceneIndex     "8f04ebf3-0008-4b8e-9a9a-1a2b3c4d5e60" (u8 write=recall)
 *
 * NOTE: iOS requires PWA/Standalone for Web Bluetooth. Use Chrome/Edge on Android for best results.
 */

const SVC = "8f04ebf3-5c4b-4b8e-9a9a-1a2b3c4d5e60";
const CH = (base: number, off: number) => {
  // Turn numeric base+off into UUID suffix (for readability; replace with your real UUIDs)
  const hex = (base + off).toString(16).padStart(4, "0");
  return `8f04ebf3-00${hex}-4b8e-9a9a-1a2b3c4d5e60`;
};
const CHBASE = { ch1: 0x10, ch2: 0x20, ch3: 0x30 };
const UU = {
  masterFader: "8f04ebf3-0001-4b8e-9a9a-1a2b3c4d5e60",
  limiterEnable: "8f04ebf3-0002-4b8e-9a9a-1a2b3c4d5e60",
  limiterThresh: "8f04ebf3-0003-4b8e-9a9a-1a2b3c4d5e60",
  aux1Level: "8f04ebf3-0004-4b8e-9a9a-1a2b3c4d5e60",
  aux1DelayMs: "8f04ebf3-0005-4b8e-9a9a-1a2b3c4d5e60",
  aux1Feedback: "8f04ebf3-0006-4b8e-9a9a-1a2b3c4d5e60",
  aux1ReverbTime: "8f04ebf3-0007-4b8e-9a9a-1a2b3c4d5e60",
  sceneIndex: "8f04ebf3-0008-4b8e-9a9a-1a2b3c4d5e60",
} as const;

function f32(v: number) { const b = new ArrayBuffer(4); new DataView(b).setFloat32(0, v, true); return b; }
function u8(v: number) { return new Uint8Array([v & 0xff]).buffer; }

// dB mapping helpers
const linToDb = (x: number) => (x <= 0 ? -80 : 20 * Math.log10(x));
const faderCurve = (x: number) => {
  // Map [0..1] UI to approximate -80..+6 dB with audio taper
  const db = -80 + Math.pow(x, 2.2) * 86; // -80 to +6
  return Math.pow(10, db / 20);
};

function VSlider({ value, onChange, min=0, max=1, step=0.001, height=140 }:{ value:number; onChange:(v:number)=>void; min?:number; max?:number; step?:number; height?:number; }){
  return (
    <div className="flex flex-col items-center" style={{height}}>
      <input type="range" orient="vertical" className="h-full accent-blue-500" min={min} max={max} step={step} value={value}
             onChange={(e)=>onChange(parseFloat(e.target.value))} />
    </div>
  );
}

function Meter({ rms=0, peak=0 }:{rms?:number; peak?:number}){
  // simple dual bar meter 0..1
  const pct = Math.min(100, Math.max(0, rms*100));
  const pp = Math.min(100, Math.max(0, peak*100));
  return (
    <div className="w-3 h-40 bg-slate-200 rounded relative overflow-hidden">
      <div className="absolute bottom-0 left-0 right-0 bg-green-400" style={{height:`${pct}%`}} />
      <div className="absolute bottom-0 left-0 right-0 h-0.5 bg-rose-600" style={{bottom:`${pp}%`}} />
    </div>
  );
}

type EQBand = { g:number; f:number; q:number };

function EQGraph({ bands, onChange }:{ bands: EQBand[]; onChange:(i:number, b:EQBand)=>void }){
  // Minimal SVG curve preview; not an exact filter plot, but gives shape and draggable F
  const W=240, H=120; const mid=H/2;
  const points = new Array(W).fill(0).map((_,x)=>{
    const fx = 20 * Math.pow(1000, x/(W-1)); // fake log scale 20..20k
    let y=0;
    for (const b of bands){ y += b.g * Math.exp(-Math.pow(Math.log(fx/b.f)/(b.q*1.2),2)); }
    return [x, mid - y*40];
  });
  return (
    <svg width={W} height={H} className="rounded border border-slate-300 bg-white">
      <polyline fill="none" stroke="#0ea5e9" strokeWidth={2} points={points.map(p=>p.join(",")).join(" ")} />
      {bands.map((b,i)=>{
        const x = Math.log10(b.f/20)/Math.log10(1000); // 0..1
        const cx = x*W; const cy = mid - b.g*40;
        return (
          <g key={i}>
            <circle cx={cx} cy={cy} r={6} className="fill-blue-500/80 cursor-pointer" />
          </g>
        );
      })}
    </svg>
  );
}

function Toggle({checked, onChange, label}:{checked:boolean; onChange:(v:boolean)=>void; label:string}){
  return (
    <label className="flex items-center gap-2 text-xs select-none">
      <input type="checkbox" checked={checked} onChange={(e)=>onChange(e.target.checked)} />
      {label}
    </label>
  );
}

function Button({children,onClick,variant="default"}:{children:React.ReactNode; onClick:()=>void; variant?:"default"|"ghost"|"danger"}){
  const cls = variant==="danger" ? "bg-rose-600 text-white hover:bg-rose-700" : variant==="ghost" ? "bg-white border border-slate-300 hover:bg-slate-50" : "bg-blue-600 text-white hover:bg-blue-700";
  return <button onClick={onClick} className={`px-3 py-2 rounded-xl text-sm shadow ${cls}`}>{children}</button>;
}

function ChannelStrip({
  title,
  base,
  connected,
  server,
}:{ title:string; base:number; connected:boolean; server:BluetoothRemoteGATTServer|null }){
  // Local UI state
  const [inputGain, setInputGain] = useState(0.4);
  const [hpf, setHpf] = useState(100);
  const [eq, setEq] = useState<EQBand[]>([
    {g:-0.2,f:80,q:1.0},
    {g: 0.0,f:750,q:1.2},
    {g: 0.0,f:2500,q:1.2},
    {g: 0.1,f:8000,q:1.0},
  ]);
  const [comp, setComp] = useState({ th:0.6, ratio:3, att:10, rel:120, mu:0.2 });
  const [gate, setGate] = useState({ th:0.2, hold:60, rel:120 });
  const [aux, setAux] = useState({ send:0.2, pre:true });
  const [pan, setPan] = useState(0.5);
  const [fader, setFader] = useState(0.7);
  const [mute, setMute] = useState(false);
  const [solo, setSolo] = useState(false);
  const [rms, setRms]   = useState(0);
  const [peak, setPeak] = useState(0);

  async function getChar(uuid:string){ if(!server) throw new Error("Not connected"); const svc = await server.getPrimaryService(SVC); return await svc.getCharacteristic(uuid); }
  async function w32(off:number, v:number){ const c = await getChar(CH(base,off)); await c.writeValue(f32(v)); }
  async function w8(off:number, v:number){ const c = await getChar(CH(base,off)); await c.writeValue(u8(v)); }
  async function wEQ(){ const buf = new ArrayBuffer(4*12); const dv=new DataView(buf); const arr=[eq[0].g,eq[0].f,eq[0].q, eq[1].g,eq[1].f,eq[1].q, eq[2].g,eq[2].f,eq[2].q, eq[3].g,eq[3].f,eq[3].q]; arr.forEach((v,i)=>dv.setFloat32(i*4,v,true)); const c=await getChar(CH(base,0x20)); await c.writeValue(buf); }

  // writers
  useEffect(()=>{ if(connected) w32(0x01,inputGain); },[connected,inputGain]);
  useEffect(()=>{ if(connected) w32(0x08,hpf); },[connected,hpf]);
  useEffect(()=>{ if(connected) wEQ(); },[connected,eq]);
  useEffect(()=>{ if(connected){ w32(0x0c,comp.th); w32(0x0d,comp.ratio); w32(0x0e,comp.att); w32(0x0f,comp.rel); w32(0x10,comp.mu);} },[connected,comp]);
  useEffect(()=>{ if(connected){ w32(0x09,gate.th); w32(0x0a,gate.hold); w32(0x0b,gate.rel);} },[connected,gate]);
  useEffect(()=>{ if(connected){ w32(0x02,pan); } },[connected,pan]);
  useEffect(()=>{ if(connected){ w32(0x03,fader); } },[connected,fader]);
  useEffect(()=>{ if(connected){ w8(0x04, mute?1:0); } },[connected,mute]);
  useEffect(()=>{ if(connected){ w8(0x05, solo?1:0); } },[connected,solo]);
  useEffect(()=>{ if(connected){ w32(0x06, aux.send); w8(0x07, aux.pre?1:0);} },[connected,aux]);

  // meters (subscribe when connected)
  useEffect(()=>{
    if(!connected||!server) return;
    let peakC:BluetoothRemoteGATTCharacteristic|undefined, rmsC:BluetoothRemoteGATTCharacteristic|undefined;
    (async()=>{
      const svc = await server.getPrimaryService(SVC);
      rmsC = await svc.getCharacteristic(CH(base,0xF0));
      peakC = await svc.getCharacteristic(CH(base,0xF1));
      const onR = (e:any)=>{ const dv=new DataView((e.target as BluetoothRemoteGATTCharacteristic).value!.buffer); setRms(Math.max(0,Math.min(1,dv.getFloat32(0,true)))); };
      const onP = (e:any)=>{ const dv=new DataView((e.target as BluetoothRemoteGATTCharacteristic).value!.buffer); setPeak(Math.max(0,Math.min(1,dv.getFloat32(0,true)))); };
      await rmsC.startNotifications(); rmsC.addEventListener("characteristicvaluechanged", onR);
      await peakC.startNotifications(); peakC.addEventListener("characteristicvaluechanged", onP);
    })();
    return ()=>{
      rmsC?.stopNotifications(); peakC?.stopNotifications();
    };
  },[connected,server,base]);

  const faderDb = (():string=>{
    const lin = faderCurve(fader); const db = linToDb(lin); return db<=-79?"-∞ dB":`${db.toFixed(1)} dB`;
  })();

  return (
    <div className="flex flex-col items-stretch w-56 rounded-2xl bg-white/80 border border-slate-200 shadow p-3">
      <div className="text-sm font-semibold text-slate-700 mb-2">{title}</div>
      <div className="grid grid-cols-2 gap-2 text-[11px] text-slate-600">
        <div>
          <div className="font-medium">Gain</div>
          <input type="range" min={0} max={1} step={0.001} value={inputGain} onChange={e=>setInputGain(parseFloat(e.target.value))} className="w-full accent-blue-500" />
          <div className="mt-2 font-medium">HPF</div>
          <input type="range" min={20} max={400} step={1} value={hpf} onChange={e=>setHpf(parseFloat(e.target.value))} className="w-full accent-blue-500" />
          <div className="mt-2 font-medium">Pan</div>
          <input type="range" min={0} max={1} step={0.001} value={pan} onChange={e=>setPan(parseFloat(e.target.value))} className="w-full accent-blue-500" />
          <div className="mt-2 font-medium">AUX1 Send</div>
          <input type="range" min={0} max={1} step={0.001} value={aux.send} onChange={e=>setAux(a=>({...a,send:parseFloat(e.target.value)}))} className="w-full accent-blue-500" />
          <div className="mt-1"><Toggle checked={aux.pre} onChange={(v)=>setAux(a=>({...a,pre:v}))} label="Pre‑Fader" /></div>
        </div>
        <div>
          <div className="font-medium mb-1">EQ (4‑band P.EQ)</div>
          <EQGraph bands={eq} onChange={(i,b)=>{ const cp=[...eq]; cp[i]=b; setEq(cp); }} />
          {eq.map((b, i)=> (
            <div key={i} className="grid grid-cols-3 gap-1 mt-1">
              <input type="range" min={-1} max={1} step={0.001} value={b.g} onChange={e=>{const g=parseFloat(e.target.value); const cp=[...eq]; cp[i]={...cp[i],g}; setEq(cp);}} className="accent-blue-500" />
              <input type="range" min={40} max={14000} step={1} value={b.f} onChange={e=>{const f=parseFloat(e.target.value); const cp=[...eq]; cp[i]={...cp[i],f}; setEq(cp);}} className="accent-blue-500" />
              <input type="range" min={0.4} max={8} step={0.001} value={b.q} onChange={e=>{const q=parseFloat(e.target.value); const cp=[...eq]; cp[i]={...cp[i],q}; setEq(cp);}} className="accent-blue-500" />
            </div>
          ))}
          <div className="mt-2 grid grid-cols-5 gap-1 items-center">
            <div className="col-span-2 text-[11px] font-medium">Comp</div>
            <input title="Thresh" type="range" min={0} max={1} step={0.001} value={comp.th} onChange={e=>setComp(c=>({...c,th:parseFloat(e.target.value)}))} className="accent-blue-500" />
            <input title="Ratio" type="range" min={1} max={20} step={0.1} value={comp.ratio} onChange={e=>setComp(c=>({...c,ratio:parseFloat(e.target.value)}))} className="accent-blue-500" />
            <input title="Make‑up" type="range" min={0} max={1} step={0.001} value={comp.mu} onChange={e=>setComp(c=>({...c,mu:parseFloat(e.target.value)}))} className="accent-blue-500" />
          </div>
          <div className="mt-1 grid grid-cols-5 gap-1 items-center">
            <div className="col-span-2 text-[11px] font-medium">Gate</div>
            <input title="Thresh" type="range" min={0} max={1} step={0.001} value={gate.th} onChange={e=>setGate(g=>({...g,th:parseFloat(e.target.value)}))} className="accent-blue-500" />
            <input title="Hold" type="range" min={10} max={500} step={1} value={gate.hold} onChange={e=>setGate(g=>({...g,hold:parseFloat(e.target.value)}))} className="accent-blue-500" />
            <input title="Release" type="range" min={20} max={1000} step={1} value={gate.rel} onChange={e=>setGate(g=>({...g,rel:parseFloat(e.target.value)}))} className="accent-blue-500" />
          </div>
        </div>
      </div>

      <div className="mt-3 flex items-end gap-2">
        <Meter rms={rms} peak={peak} />
        <div className="flex-1 flex flex-col items-center">
          <div className="text-[11px] text-slate-500 h-4">{faderDb}</div>
          <VSlider value={fader} onChange={setFader} />
          <div className="mt-1 flex gap-1">
            <Button onClick={()=>setMute(m=>!m)} variant="ghost">{mute?"Unmute":"Mute"}</Button>
            <Button onClick={()=>setSolo(s=>!s)} variant="ghost">{solo?"Unsolo":"Solo"}</Button>
          </div>
        </div>
      </div>
    </div>
  );
}

export default function AmpJiewDigitalMixer(){
  const [device, setDevice] = useState<BluetoothDevice|null>(null);
  const [server, setServer] = useState<BluetoothRemoteGATTServer|null>(null);
  const [connected, setConnected] = useState(false);

  const connect = async ()=>{
    try{
      const dev = await navigator.bluetooth.requestDevice({ filters:[{ namePrefix:"AmpJiew" }], optionalServices:[SVC] });
      setDevice(dev); const srv = await dev.gatt!.connect(); setServer(srv); setConnected(true);
      dev.addEventListener("gattserverdisconnected", ()=> setConnected(false));
    }catch(e){ alert((e as Error).message || String(e)); }
  };
  const disconnect = ()=>{ if(device?.gatt?.connected) device.gatt.disconnect(); setConnected(false); };

  // Master state
  const [master, setMaster] = useState(0.8);
  const [lim, setLim] = useState({ en:true, th:0.7 });
  const [aux, setAux] = useState({ level:0.25, delay:180, fb:0.32, rv:1.2 });

  async function getChar(uuid:string){ if(!server) throw new Error("Not connected"); const svc = await server.getPrimaryService(SVC); return await svc.getCharacteristic(uuid); }
  useEffect(()=>{ if(!connected) return; (async()=>{ (await getChar(UU.masterFader)).writeValue(f32(master)); })(); },[connected,master]);
  useEffect(()=>{ if(!connected) return; (async()=>{ (await getChar(UU.limiterEnable)).writeValue(u8(lim.en?1:0)); (await getChar(UU.limiterThresh)).writeValue(f32(lim.th)); })(); },[connected,lim]);
  useEffect(()=>{ if(!connected) return; (async()=>{ (await getChar(UU.aux1Level)).writeValue(f32(aux.level)); (await getChar(UU.aux1DelayMs)).writeValue(f32(aux.delay)); (await getChar(UU.aux1Feedback)).writeValue(f32(aux.fb)); (await getChar(UU.aux1ReverbTime)).writeValue(f32(aux.rv)); })(); },[connected,aux]);

  const saveScene = (i:number)=>{ const snapshot = { master, lim, aux }; localStorage.setItem(`ampjiew_scene_${i}`, JSON.stringify(snapshot)); alert(`บันทึกฉาก ${i}`); };
  const loadScene = (i:number)=>{ const raw = localStorage.getItem(`ampjiew_scene_${i}`); if(!raw) return alert("ไม่มีฉากนี้"); try{ const s = JSON.parse(raw); setMaster(s.master??master); setLim(s.lim??lim); setAux(s.aux??aux);}catch{ alert("ข้อมูลฉากเสียหาย"); } };

  // XR-Style Dark Theme + Tabs
  const [tab, setTab] = useState<'mixer'|'channel'|'input'|'gate'|'eq'|'comp'|'sends'|'main'|'fx'|'meter'>('gate');
  const [selected, setSelected] = useState<number>(0); // index of channel strip

  // Build channels (8 mono + 4 stereo + 4 FX returns visual)
  const strips = Array.from({length:12}, (_,i)=>({title: i<8?`MIC${i+1}`: i<10?`ST${i-7}`:`AUX${i-9}`, base: (i<8?0x10: i<10?0x30:0x40)+i*0x10 }));

  return (
    <div className="min-h-screen bg-[#121418] text-slate-200 p-3">
      <div className="max-w-[1400px] mx-auto">
        {/* Top Bar Tabs */}
        <div className="flex items-center gap-1 text-xs mb-2">
          {['Mixer','Channel','Input','Gate','EQ','Comp','Sends','Main','FX','Meter'].map((t,i)=>{
            const key = t.toLowerCase() as typeof tab;
            const active = tab===key;
            return (
              <button key={t} onClick={()=>setTab(key)} className={`px-3 py-2 rounded bg-[#1b1e24] border ${active?'border-cyan-500 text-cyan-300':'border-[#2a2f3a] text-slate-300 hover:text-white'}`}>{t}</button>
            );
          })}
          <div className="ml-auto flex gap-2">
            {!connected ? <button onClick={connect} className="px-3 py-2 rounded bg-cyan-600 text-white">เชื่อมต่อ</button> : <button onClick={disconnect} className="px-3 py-2 rounded bg-rose-600 text-white">ตัดการเชื่อมต่อ</button>}
          </div>
        </div>

        {/* Upper module panel (selected channel) */}
        <div className="rounded-xl bg-[#1b1e24] border border-[#2a2f3a] p-3 mb-3">
          <div className="text-sm font-semibold mb-2">{strips[selected].title} — {tab.toUpperCase()}</div>
          {tab==='gate' && (
            <div className="grid grid-cols-4 gap-4">
              <div>
                <div className="text-xs mb-1">Noise Gate</div>
                <div className="h-24 rounded bg-[#121418] border border-[#2a2f3a] flex items-center justify-center">(Curve)</div>
                <div className="grid grid-cols-3 gap-2 mt-2 text-[11px]">
                  <div>
                    <div>Threshold</div>
                    <input type="range" min={0} max={1} step={0.001} className="w-full" />
                  </div>
                  <div>
                    <div>GR</div>
                    <input type="range" min={0} max={1} step={0.001} className="w-full" />
                  </div>
                  <div>
                    <div>Range</div>
                    <input type="range" min={0} max={1} step={0.001} className="w-full" />
                  </div>
                </div>
              </div>
              <div>
                <div className="text-xs mb-1">Gain Envelope</div>
                <div className="h-24 rounded bg-[#121418] border border-[#2a2f3a] flex items-center justify-center">(Env)</div>
                <div className="grid grid-cols-3 gap-2 mt-2 text-[11px]">
                  <div><div>Attack</div><input type="range" min={0} max={100} className="w-full" /></div>
                  <div><div>Hold</div><input type="range" min={0} max={500} className="w-full" /></div>
                  <div><div>Release</div><input type="range" min={0} max={1000} className="w-full" /></div>
                </div>
              </div>
              <div>
                <div className="text-xs mb-1">Side Chain Filter</div>
                <div className="h-24 rounded bg-[#121418] border border-[#2a2f3a] flex items-center justify-center">(Filter)</div>
                <div className="grid grid-cols-2 gap-2 mt-2 text-[11px]">
                  <div><div>Type</div><input type="range" min={0} max={1} step={0.001} className="w-full" /></div>
                  <div><div>Frequency</div><input type="range" min={20} max={8000} className="w-full" /></div>
                </div>
              </div>
              <div className="flex flex-col gap-2">
                <div className="text-xs">Presets</div>
                {['Kick','Snare','Acoustic','Vocal'].map(p=> <button key={p} className="px-2 py-1 rounded bg-[#121418] border border-[#2a2f3a] text-xs text-slate-300 hover:text-white">{p}</button>)}
              </div>
            </div>
          )}
          {tab==='eq' && (
            <div className="grid grid-cols-3 gap-4">
              <div className="col-span-2">
                <div className="text-xs mb-1">Parametric EQ</div>
                <div className="bg-[#121418] border border-[#2a2f3a] rounded p-2"><EQGraph bands={[{g:-0.2,f:80,q:1},{g:0,f:750,q:1.2},{g:0,f:2500,q:1.2},{g:0.1,f:8000,q:1}]} onChange={()=>{}} /></div>
              </div>
              <div className="grid grid-cols-3 gap-2 text-[11px]">
                {[0,1,2,3].map(i=> (
                  <div key={i} className="bg-[#121418] border border-[#2a2f3a] rounded p-2">
                    <div className="font-semibold mb-1">Band {i+1}</div>
                    <div>Gain</div><input type="range" min={-1} max={1} step={0.001} className="w-full" />
                    <div>Freq</div><input type="range" min={40} max={14000} step={1} className="w-full" />
                    <div>Q</div><input type="range" min={0.4} max={8} step={0.001} className="w-full" />
                  </div>
                ))}
              </div>
            </div>
          )}
          {tab==='comp' && (
            <div className="grid grid-cols-4 gap-4">
              <div>
                <div className="text-xs mb-1">Compressor Curve</div>
                <div className="h-24 rounded bg-[#121418] border border-[#2a2f3a] flex items-center justify-center">(Knee)</div>
              </div>
              <div className="grid grid-cols-2 gap-2 text-[11px] col-span-3">
                {['Thresh','Ratio','Attack','Release','Make‑up'].map(lbl=> (
                  <div key={lbl}><div>{lbl}</div><input type="range" min={0} max={1} step={0.001} className="w-full" /></div>
                ))}
              </div>
            </div>
          )}
          {tab==='sends' && (
            <div className="grid grid-cols-4 gap-3 text-[11px]">
              {[1,2,3,4].map(b=> (
                <div key={b} className="bg-[#121418] border border-[#2a2f3a] rounded p-2">
                  <div className="font-semibold mb-1">Bus {b}</div>
                  <input type="range" min={0} max={1} step={0.001} className="w-full" />
                  <div className="mt-1"><label className="flex items-center gap-2"><input type="checkbox" /> Pre‑Fader</label></div>
                </div>
              ))}
            </div>
          )}
        </div>

        {/* Mixer strips bottom */}
        <div className="rounded-xl bg-[#1b1e24] border border-[#2a2f3a] p-3 overflow-x-auto">
          <div className="flex gap-3 min-w-[1100px]">
            {strips.map((s, i)=> (
              <div key={i} className={`w-20 p-2 rounded bg-[#111318] border ${i===selected?'border-cyan-500':'border-[#2a2f3a]'} text-[11px]`}>
                <div className="h-6 mb-1 rounded text-center font-semibold" style={{background:i<8?"#ef444433": i<10?"#22c55e33":"#a78bfa33"}}>{s.title}</div>
                <div className="h-24 mb-1 bg-[#0b0e12] rounded relative overflow-hidden">
                  {/* meter */}
                  <div className="absolute bottom-0 left-1 w-3 bg-green-500/70" style={{height:`${Math.random()*100}%`}} />
                  <div className="absolute bottom-0 left-1 w-3 h-0.5 bg-rose-500" style={{bottom:`${Math.random()*100}%`}} />
                </div>
                <div className="flex gap-1 mb-1">
                  <button onClick={()=>setSelected(i)} className="flex-1 rounded bg-[#1f2430] border border-[#2a2f3a]">Sel</button>
                  <button className="flex-1 rounded bg-[#1f2430] border border-[#2a2f3a]">Solo</button>
                </div>
                <div className="flex items-center justify-center mb-1"><div className="w-2/3"><VSlider value={0.7} onChange={()=>{}} /></div></div>
                <div className="flex gap-1">
                  <button className="flex-1 rounded bg-[#1f2430] border border-[#2a2f3a]">Mute</button>
                  <button className="flex-1 rounded bg-[#1f2430] border border-[#2a2f3a]">FX</button>
                </div>
              </div>
            ))}
            {/* Right master section */}
            <div className="w-28 p-2 rounded bg-[#111318] border border-[#2a2f3a] text-[11px] ml-2">
              <div className="text-center font-semibold mb-2">Main LR</div>
              <div className="flex items-end gap-2">
                <Meter rms={0.4} peak={0.7} />
                <div className="flex-1">
                  <div className="text-center text-[10px] mb-1">Master</div>
                  <VSlider value={master} onChange={setMaster} />
                  <div className="mt-1 text-center text-[10px]">{(()=>{ const lin=faderCurve(master); const db=linToDb(lin); return db<=-79?"-∞":`${db.toFixed(1)} dB`; })()}</div>
                </div>
              </div>
              <div className="mt-2">
                <label className="flex items-center gap-2"><input type="checkbox" checked={lim.en} onChange={e=>setLim(p=>({...p,en:e.target.checked}))} /> Limiter</label>
                <input type="range" min={0} max={1} step={0.001} value={lim.th} onChange={e=>setLim(p=>({...p,th:parseFloat(e.target.value)}))} className="w-full" />
              </div>
              <div className="mt-2 text-center">
                <div className="font-semibold mb-1">FX Returns</div>
                {[1,2,3,4].map(n=> (
                  <div key={n} className="mb-1 flex items-center gap-2">
                    <div className="text-[10px] w-8">FX {n}</div>
                    <div className="flex-1"><input type="range" min={0} max={1} step={0.001} className="w-full" /></div>
                  </div>
                ))}
              </div>
            </div>
          </div>
        </div>

        <div className="text-[10px] text-slate-400 mt-2">สกิน XR‑style: แท็บด้านบน + แผงโมดูล + แถวชานเนลสไตล์มิกดิจิตอล พร้อม Master/FX ด้านขวา (ค่าบางส่วนเป็นตัวอย่าง/placeholder — ผูก BLE ทีละพารามิเตอร์ได้ทันทีเมื่อพร้อม)</div>
      </div>
    </div>
  );
}
