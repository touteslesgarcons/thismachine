# thismachine
zine.
import React, { useEffect, useMemo, useRef, useState } from "react";

// Single‑file React app for uploading a zine (images or a PDF), arranging pages,
// styling metadata, previewing a reader view, printing to PDF, and exporting a
// standalone HTML of the zine (image‑based projects export best).
//
// Notes:
// • Works entirely client‑side; no server required.
// • Best with images (JPG/PNG/WebP). PDFs can be previewed/printed but are not
//   embedded into the exported HTML (object URLs are ephemeral). For a fully
//   portable export, use images.
// • Uses Tailwind classes for styling. If Tailwind isn’t present, it still looks
//   fine with basic CSS variables included below.

export default function ZineStudio() {
  const [title, setTitle] = useState("My Zine");
  const [subtitle, setSubtitle] = useState("Issue #1");
  const [creator, setCreator] = useState("");
  const [dateStr, setDateStr] = useState(new Date().toISOString().slice(0,10));
  const [description, setDescription] = useState("Write a short blurb about your zine.");
  const [accent, setAccent] = useState("#6b5cff");
  const [bg, setBg] = useState("#0b0b0c");
  const [fg, setFg] = useState("#f7f7f7");
  const [compact, setCompact] = useState(false);

  type ZineFile = {
    id: string;
    name: string;
    kind: "image" | "pdf";
    dataUrl?: string; // for images
    objectUrl?: string; // for pdfs
    width?: number;
    height?: number;
  };

  const [files, setFiles] = useState<ZineFile[]>([]);
  const [activeTab, setActiveTab] = useState<"edit" | "preview">("edit");
  const [readerMode, setReaderMode] = useState<"spread" | "single">("single");

  const dropRef = useRef<HTMLDivElement | null>(null);

  // Local storage persistence
  useEffect(() => {
    const saved = localStorage.getItem("zine_studio_state_v1");
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        setTitle(parsed.title ?? title);
        setSubtitle(parsed.subtitle ?? subtitle);
        setCreator(parsed.creator ?? creator);
        setDateStr(parsed.dateStr ?? dateStr);
        setDescription(parsed.description ?? description);
        setAccent(parsed.accent ?? accent);
        setBg(parsed.bg ?? bg);
        setFg(parsed.fg ?? fg);
        setCompact(!!parsed.compact);
        if (Array.isArray(parsed.files)) setFiles(parsed.files);
      } catch {}
    }
  }, []);

  useEffect(() => {
    const state = {
      title, subtitle, creator, dateStr, description, accent, bg, fg, compact, files
    };
    localStorage.setItem("zine_studio_state_v1", JSON.stringify(state));
  }, [title, subtitle, creator, dateStr, description, accent, bg, fg, compact, files]);

  // Drag & drop
  useEffect(() => {
    const el = dropRef.current;
    if (!el) return;
    const onPrevent = (e: DragEvent) => { e.preventDefault(); e.stopPropagation(); };
    const onDrop = async (e: DragEvent) => {
      e.preventDefault();
      e.stopPropagation();
      const dt = e.dataTransfer;
      if (!dt) return;
      const newFiles: ZineFile[] = [];
      for (const item of Array.from(dt.files)) {
        const zf = await toZineFile(item);
        if (zf) newFiles.push(zf);
      }
      if (newFiles.length) setFiles(prev => [...prev, ...newFiles]);
    };
    el.addEventListener("dragover", onPrevent);
    el.addEventListener("dragenter", onPrevent);
    el.addEventListener("drop", onDrop);
    return () => {
      el.removeEventListener("dragover", onPrevent);
      el.removeEventListener("dragenter", onPrevent);
      el.removeEventListener("drop", onDrop);
    };
  }, []);

  async function onPickFiles(e: React.ChangeEvent<HTMLInputElement>) {
    const fl = e.target.files;
    if (!fl) return;
    const arr = await Promise.all(Array.from(fl).map(f => toZineFile(f)));
    setFiles(prev => [...prev, ...arr.filter(Boolean) as ZineFile[]]);
    e.target.value = ""; // reset
  }

  function removeFile(id: string) {
    setFiles(prev => prev.filter(f => f.id !== id));
  }

  function move(idx: number, dir: -1 | 1) {
    setFiles(prev => {
      const a = [...prev];
      const j = idx + dir;
      if (j < 0 || j >= a.length) return a;
      const [item] = a.splice(idx, 1);
      a.splice(j, 0, item);
      return a;
    });
  }

  function onReorderDragStart(e: React.DragEvent, index: number) {
    e.dataTransfer.setData("text/plain", String(index));
  }
  function onReorderDrop(e: React.DragEvent, index: number) {
    const from = Number(e.dataTransfer.getData("text/plain"));
    if (Number.isNaN(from)) return;
    setFiles(prev => {
      const a = [...prev];
      const [item] = a.splice(from, 1);
      a.splice(index, 0, item);
      return a;
    });
  }

  // Build a standalone HTML file (image‑based only) for download
  function downloadStandaloneHTML() {
    const hasPdf = files.some(f => f.kind === "pdf");
    const doc = `<!doctype html>
<html lang="en">
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>${escapeHtml(title)} — ${escapeHtml(subtitle)}</title>
<style>
:root{--accent:${accent};--bg:${bg};--fg:${fg}}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--fg);font:16px/1.4 ui-sans-serif,system-ui,-apple-system,Segoe UI,Roboto,Inter,Arial}
.header{padding:2rem 1rem;border-bottom:1px solid #222}
.h1{font-size:clamp(1.8rem,4vw,3rem);font-weight:800;letter-spacing:-.02em}
.meta{opacity:.8;margin-top:.375rem}
.desc{max-width:70ch;margin-top:.75rem}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(280px,1fr));gap:12px;padding:1rem}
.card{background:#111;border:1px solid #222;border-radius:16px;overflow:hidden}
.card img{width:100%;display:block}
.footer{opacity:.7;padding:2rem 1rem;text-align:center;border-top:1px solid #222}
.badge{display:inline-block;padding:.25rem .6rem;border:1px solid #333;border-radius:999px;background:#0f0f10}
</style>
<body>
  <header class="header">
    <div class="badge">Standalone zine</div>
    <div class="h1">${escapeHtml(title)}</div>
    <div class="meta">${escapeHtml(subtitle)}${creator?" • "+escapeHtml(creator):""}${dateStr?" • "+escapeHtml(dateStr):""}</div>
    <p class="desc">${escapeHtml(description)}</p>
  </header>
  <main class="grid">
    ${files.filter(f=>f.kind==="image").map(f=>`<figure class="card"><img alt="${escapeHtml(f.name)}" src="${f.dataUrl ?? ""}"/></figure>`).join("")}
  </main>
  ${hasPdf?`<div class="footer">Note: PDFs can’t be embedded in this exported file. Re‑open your project in Zine Studio to view PDFs.</div>`:``}
</body></html>`;

    const blob = new Blob([doc], { type: "text/html" });
    const url = URL.createObjectURL(blob);
    triggerDownload(url, slugify(title) + "_zine.html");
    setTimeout(() => URL.revokeObjectURL(url), 2000);
  }

  function triggerDownload(url: string, filename: string) {
    const a = document.createElement("a");
    a.href = url; a.download = filename; a.rel = "noopener"; a.click();
  }

  function clearProject() {
    if (!confirm("This will remove all pages and metadata from this browser. Continue?")) return;
    localStorage.removeItem("zine_studio_state_v1");
    setFiles([]);
    setTitle("My Zine");
    setSubtitle("Issue #1");
    setCreator("");
    setDescription("");
  }

  // Print support: opens preview then calls print
  function printZine() {
    setActiveTab("preview");
    setTimeout(() => window.print(), 250);
  }

  // Serialize project to JSON
  function exportProject() {
    const blob = new Blob([JSON.stringify({
      title, subtitle, creator, dateStr, description, accent, bg, fg, compact, files
    }, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    triggerDownload(url, slugify(title) + "_project.json");
    setTimeout(() => URL.revokeObjectURL(url), 2000);
  }

  // Restore from JSON
  function handleImportProject(e: React.ChangeEvent<HTMLInputElement>) {
    const f = e.target.files?.[0];
    if (!f) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const parsed = JSON.parse(String(reader.result));
        setTitle(parsed.title ?? title);
        setSubtitle(parsed.subtitle ?? subtitle);
        setCreator(parsed.creator ?? creator);
        setDateStr(parsed.dateStr ?? dateStr);
        setDescription(parsed.description ?? description);
        setAccent(parsed.accent ?? accent);
        setBg(parsed.bg ?? bg);
        setFg(parsed.fg ?? fg);
        setCompact(!!parsed.compact);
        if (Array.isArray(parsed.files)) setFiles(parsed.files);
      } catch (err) {
        alert("Invalid project file");
      }
    };
    reader.readAsText(f);
    e.target.value = "";
  }

  return (
    <div className="min-h-dvh" style={{
      // Fallback CSS vars in case Tailwind isn’t present
      // (the app still renders with decent defaults)
      // @ts-ignore
      "--accent": accent,
      "--bg": bg,
      "--fg": fg,
      background: bg,
      color: fg,
      fontFamily: 'ui-sans-serif, system-ui, -apple-system, Segoe UI, Inter, Roboto, Arial'
    }}>
      <style>{baseCss}</style>

      <header className="sticky top-0 z-30 border-b border-neutral-800/60 backdrop-blur supports-[backdrop-filter]:bg-black/30">
        <div className="mx-auto max-w-6xl px-4 py-3 flex items-center gap-3 justify-between">
          <div className="flex items-center gap-3">
            <div className="w-8 h-8 rounded-xl" style={{background: accent}}/>
            <div>
              <div className="text-lg font-bold tracking-tight">Zine Studio</div>
              <div className="text-xs opacity-60">No‑code, local, printable</div>
            </div>
          </div>
          <nav className="flex items-center gap-1">
            <TabButton active={activeTab === "edit"} onClick={() => setActiveTab("edit")}>Editor</TabButton>
            <TabButton active={activeTab === "preview"} onClick={() => setActiveTab("preview")}>Reader</TabButton>
            <div className="ml-2 hidden sm:flex gap-1">
              <Button onClick={printZine}>Print / PDF</Button>
              <Button onClick={downloadStandaloneHTML}>Export HTML</Button>
              <Button onClick={exportProject} variant="ghost">Export JSON</Button>
              <label className="cursor-pointer"><span className="sr-only">Import</span>
                <input type="file" accept="application/json" hidden onChange={handleImportProject}/>
                <Button variant="ghost">Import</Button>
              </label>
            </div>
          </nav>
        </div>
      </header>

      {activeTab === "edit" ? (
        <main className="mx-auto max-w-6xl px-4 py-6 grid gap-6 lg:grid-cols-3">
          <section className="lg:col-span-2 space-y-4">
            <Card>
              <div className="flex items-start justify-between gap-4">
                <div className="space-y-2 w-full">
                  <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                    <Field label="Title">
                      <input value={title} onChange={e=>setTitle(e.target.value)} className="inp"/>
                    </Field>
                    <Field label="Subtitle / Issue">
                      <input value={subtitle} onChange={e=>setSubtitle(e.target.value)} className="inp"/>
                    </Field>
                    <Field label="Creator">
                      <input value={creator} onChange={e=>setCreator(e.target.value)} className="inp"/>
                    </Field>
                    <Field label="Date">
                      <input type="date" value={dateStr} onChange={e=>setDateStr(e.target.value)} className="inp"/>
                    </Field>
                  </div>
                  <Field label="Description">
                    <textarea rows={3} value={description} onChange={e=>setDescription(e.target.value)} className="inp"/>
                  </Field>
                </div>
              </div>
            </Card>

            <Card>
              <div className="flex items-center justify-between mb-3">
                <h3 className="font-semibold">Pages</h3>
                <div className="flex items-center gap-2">
                  <input type="file" accept="image/*,application/pdf" multiple onChange={onPickFiles} id="filepick" hidden/>
                  <label htmlFor="filepick"><Button>Add files</Button></label>
                  <Button variant="ghost" onClick={clearProject}>Reset</Button>
                </div>
              </div>

              <div ref={dropRef} className="rounded-2xl border border-dashed border-neutral-700 p-6 text-center mb-4 bg-neutral-900/40">
                <p className="font-medium">Drag & drop images or a PDF here</p>
                <p className="text-sm opacity-70">JPG, PNG, WebP, or one PDF. You can reorder pages below.</p>
              </div>

              {files.length === 0 ? (
                <EmptyState/>
              ) : (
                <ul className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
                  {files.map((f, i) => (
                    <li key={f.id}
                        draggable
                        onDragStart={(e)=>onReorderDragStart(e,i)}
                        onDragOver={(e)=>e.preventDefault()}
                        onDrop={(e)=>onReorderDrop(e,i)}
                        className="group">
                      <div className="rounded-xl overflow-hidden border border-neutral-800 bg-neutral-950">
                        <div className="aspect-[3/4] bg-neutral-900 flex items-center justify-center overflow-hidden">
                          {f.kind === "image" ? (
                            // eslint-disable-next-line @next/next/no-img-element
                            <img src={f.dataUrl} alt={f.name} className="w-full h-full object-cover"/>
                          ) : (
                            <iframe src={f.objectUrl} title={f.name} className="w-full h-full"></iframe>
                          )}
                        </div>
                        <div className="p-3 flex items-center justify-between text-sm">
                          <span className="truncate opacity-80" title={f.name}>{f.name}</span>
                          <div className="flex items-center gap-1 opacity-80">
                            <IconButton onClick={()=>move(i,-1)} title="Move up">↑</IconButton>
                            <IconButton onClick={()=>move(i, 1)} title="Move down">↓</IconButton>
                            <IconButton onClick={()=>removeFile(f.id)} title="Remove">✕</IconButton>
                          </div>
                        </div>
                      </div>
                    </li>
                  ))}
                </ul>
              )}
            </Card>
          </section>

          <aside className="space-y-4">
            <Card>
              <h3 className="font-semibold mb-3">Theme</h3>
              <div className="grid grid-cols-2 gap-3">
                <Field label="Accent">
                  <input type="color" value={accent} onChange={e=>setAccent(e.target.value)} className="inp p-1 h-10"/>
                </Field>
                <Field label="Background">
                  <input type="color" value={bg} onChange={e=>setBg(e.target.value)} className="inp p-1 h-10"/>
                </Field>
                <Field label="Foreground">
                  <input type="color" value={fg} onChange={e=>setFg(e.target.value)} className="inp p-1 h-10"/>
                </Field>
                <Field label="Layout">
                  <select value={readerMode} onChange={e=>setReaderMode(e.target.value as any)} className="inp">
                    <option value="single">Single page</option>
                    <option value="spread">Two‑page spread</option>
                  </select>
                </Field>
                <Field label="Dense layout">
                  <label className="inline-flex items-center gap-2 text-sm">
                    <input type="checkbox" checked={compact} onChange={e=>setCompact(e.target.checked)}/>
                    Compact spacing
                  </label>
                </Field>
              </div>
            </Card>

            <Card>
              <h3 className="font-semibold mb-2">Publish</h3>
              <div className="space-y-2">
                <Button onClick={()=>setActiveTab("preview")}>Open Reader</Button>
                <p className="text-xs opacity-70">Tip: Use <kbd>Cmd/Ctrl+P</kbd> in Reader to save a print‑ready PDF.</p>
                <Button onClick={downloadStandaloneHTML} variant="ghost">Download standalone HTML</Button>
                <Button onClick={exportProject} variant="ghost">Export project (JSON)</Button>
              </div>
            </Card>
          </aside>
        </main>
      ) : (
        <ReaderView files={files} title={title} subtitle={subtitle} creator={creator} dateStr={dateStr} description={description} readerMode={readerMode} accent={accent} compact={compact}/>
      )}

      <footer className="border-t border-neutral-800/60">
        <div className="mx-auto max-w-6xl px-4 py-10 text-sm opacity-70 text-center">
          Built with ❤️ — everything stays in your browser.
        </div>
      </footer>
    </div>
  );
}

function TabButton({active, onClick, children}:{active?:boolean; onClick?:()=>void; children:React.ReactNode}){
  return (
    <button onClick={onClick}
      className={`px-3 py-2 rounded-xl text-sm font-medium border ${active?"bg-[var(--accent)] text-white border-transparent":"border-neutral-700 hover:bg-neutral-800/50"}`}>
      {children}
    </button>
  );
}

function Button({children, onClick, variant}:{children:React.ReactNode; onClick?:()=>void; variant?:"ghost"|"solid"}){
  const ghost = variant === "ghost";
  return (
    <button onClick={onClick}
      className={`px-3 py-2 rounded-xl text-sm font-semibold transition border ${ghost?"bg-transparent border-neutral-700 hover:bg-neutral-800/60":"bg-[var(--accent)] text-white border-transparent hover:opacity-90"}`}>
      {children}
    </button>
  );
}

function IconButton({children, onClick, title}:{children:React.ReactNode; onClick?:()=>void; title?:string}){
  return (
    <button className="px-2 py-1 rounded-lg border border-neutral-700 hover:bg-neutral-800/60" title={title} onClick={onClick}>{children}</button>
  );
}

function Field({label, children}:{label:string; children:React.ReactNode}){
  return (
    <label className="text-sm grid gap-1">
      <span className="opacity-80">{label}</span>
      {children}
    </label>
  );
}

function Card({children}:{children:React.ReactNode}){
  return (
    <div className="rounded-2xl border border-neutral-800 bg-neutral-950 p-4">{children}</div>
  );
}

function EmptyState(){
  return (
    <div className="rounded-2xl border border-neutral-800 p-10 text-center bg-neutral-950">
      <div className="text-2xl font-extrabold tracking-tight mb-2">No pages yet</div>
      <p className="opacity-70">Add JPG/PNG/WebP images or a PDF. Drag to reorder. Use Reader to preview and print.</p>
    </div>
  );
}

function ReaderView({files, title, subtitle, creator, dateStr, description, readerMode, accent, compact}:{
  files: {id:string; name:string; kind:"image"|"pdf"; dataUrl?:string; objectUrl?:string;}[];
  title:string; subtitle:string; creator:string; dateStr:string; description:string; readerMode:"single"|"spread"; accent:string; compact:boolean;
}){
  const onlyImages = files.filter(f=>f.kind==="image");
  const anyPdf = files.some(f=>f.kind==="pdf");
  const gridCols = readerMode === "spread" ? "md:grid-cols-2" : "md:grid-cols-1";
  return (
    <div>
      <div className="print:hidden mx-auto max-w-6xl px-4 pt-8">
        <div className="mb-6">
          <div className="text-3xl font-black tracking-tight" style={{color: accent}}>{title}</div>
          <div className="opacity-80">{subtitle}{creator?" • "+creator:"")}{dateStr?" • "+dateStr:"")}</div>
          {description && <p className="mt-2 max-w-2xl opacity-90">{description}</p>}
        </div>
      </div>

      {/* Printable content */}
      <div className={`mx-auto ${compact?"max-w-5xl":"max-w-6xl"} px-4 pb-12 print:px-0`}>
        <div className={`grid ${gridCols} gap-4 print:grid-cols-1`}>
          {files.map((f) => (
            <div key={f.id} className="rounded-xl overflow-hidden border border-neutral-800 bg-neutral-950 page-break-inside-avoid">
              {f.kind === "image" ? (
                // eslint-disable-next-line @next/next/no-img-element
                <img src={f.dataUrl} alt={f.name} className="w-full h-auto block"/>
              ) : (
                <iframe src={f.objectUrl} title={f.name} className="w-full aspect-[3/4] bg-white"></iframe>
              )}
            </div>
          ))}
        </div>
        {onlyImages.length === 0 && anyPdf && (
          <p className="text-sm opacity-70 mt-4">Tip: For best export results, consider adding images instead of PDF pages.</p>
        )}
      </div>
    </div>
  );
}

async function toZineFile(file: File): Promise<any | null> {
  const id = crypto.randomUUID?.() ?? Math.random().toString(36).slice(2);
  if (file.type.startsWith("image/")) {
    const dataUrl = await readAsDataURL(file);
    // Try to parse intrinsic size for nicer layout (optional)
    const size = await getImageSize(dataUrl).catch(()=>({width:0,height:0}));
    return { id, name: file.name, kind: "image", dataUrl, width: size.width, height: size.height };
  }
  if (file.type === "application/pdf") {
    const objectUrl = URL.createObjectURL(file);
    return { id, name: file.name, kind: "pdf", objectUrl };
  }
  alert(`Unsupported file type: ${file.type}`);
  return null;
}

function readAsDataURL(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const fr = new FileReader();
    fr.onerror = () => reject(fr.error);
    fr.onload = () => resolve(String(fr.result));
    fr.readAsDataURL(file);
  });
}

function getImageSize(src: string): Promise<{width:number;height:number}> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve({width: img.naturalWidth, height: img.naturalHeight});
    img.onerror = reject;
    img.src = src;
  });
}

function slugify(s: string){
  return s.toLowerCase().replace(/[^a-z0-9]+/g,"-").replace(/(^-|-$)/g,"");
}

function escapeHtml(s: string){
  return s.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/\"/g,"&quot;").replace(/'/g,"&#39;");
}

const baseCss = `
:root{--accent:#6b5cff;--bg:#0b0b0c;--fg:#f7f7f7}
.inp{background:#0f0f12;border:1px solid #2a2a2e;border-radius:12px;padding:.6rem .75rem}
.page-break-inside-avoid{break-inside:avoid}
@media print{
  body{background:white;color:black}
  header, footer, .print\:hidden{display:none !important}
  .print\:grid-cols-1{grid-template-columns:1fr !important}
}
`;
