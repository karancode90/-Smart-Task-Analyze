# -Smart-Task-Analyze

import React, { useState, useEffect, useMemo } from "react";

// Helper utilities
const clamp = (v, a = 0, b = 1) => Math.max(a, Math.min(b, v));
const daysBetween = (d1, d2 = new Date()) => Math.ceil((new Date(d1) - new Date(d2)) / (1000 * 60 * 60 * 24));

// Default sample tasks
const SAMPLE_TASKS = [
  {
    id: "t1",
    title: "Finish client report",
    notes: "Include Q4 numbers and recommendations",
    dueDate: new Date(Date.now() + 2 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10), // in 2 days
    importance: 5,
    impact: 5,
    effortHours: 4,
    blocked: false,
    tags: ["work", "report"],
    createdAt: new Date().toISOString(),
  },
  {
    id: "t2",
    title: "Refactor old payment module",
    notes: "Tech debt — estimate larger effort",
    dueDate: null,
    importance: 3,
    impact: 4,
    effortHours: 16,
    blocked: false,
    tags: ["engineering", "tech-debt"],
    createdAt: new Date(Date.now() - 2 * 24 * 60 * 60 * 1000).toISOString(),
  },
  {
    id: "t3",
    title: "Book dentist appointment",
    notes: "Prefer morning slots",
    dueDate: new Date(Date.now() + 10 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10),
    importance: 2,
    impact: 1,
    effortHours: 0.5,
    blocked: false,
    tags: ["personal"],
    createdAt: new Date().toISOString(),
  },
  {
    id: "t4",
    title: "Deploy v2.1 to staging",
    notes: "Blocked by infra issue",
    dueDate: new Date(Date.now() + 1 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10),
    importance: 4,
    impact: 5,
    effortHours: 1,
    blocked: true,
    tags: ["release", "blocking"],
    createdAt: new Date().toISOString(),
  },
];

// Scoring configuration defaults
const DEFAULT_WEIGHTS = {
  urgency: 0.30,
  importance: 0.25,
  impact: 0.20,
  effort: 0.15,
  blockedPenalty: 0.10, // how much to subtract when blocked (as fraction of total possible score)
  preferenceBoost: 0.0, // user preference boost for starred/favorite
};

// Normalize a value between min..max to 0..1
const normalize = (value, min, max) => {
  if (value == null || isNaN(value)) return 0;
  if (max === min) return 0;
  return clamp((value - min) / (max - min));
};

function computeScores(tasks, weights, options = {}) {
  // Determine normalization ranges for effort and days
  const now = new Date();

  // effort normalization: cap the maximum effort (e.g. 40h) to reduce skew
  const EFFORT_CAP = 40;
  const efforts = tasks.map((t) => Math.min(t.effortHours || 0, EFFORT_CAP));
  const minEffort = Math.min(...efforts, 0);
  const maxEffort = Math.max(...efforts, EFFORT_CAP);

  // urgency normalization: convert due date to daysUntilDue; far future -> low urgency; past due -> high urgency
  const daysArray = tasks.map((t) => {
    if (!t.dueDate) return Infinity;
    const d = daysBetween(t.dueDate, now);
    return d; // positive if dueDate in future, negative if past
  });

  // We'll map daysUntilDue to urgency score: closer date -> higher score.
  // For tasks with no dueDate, set days to large value.
  const finiteDays = daysArray.filter((d) => isFinite(d));
  const minDays = finiteDays.length ? Math.min(...finiteDays) : -30;
  const maxDays = finiteDays.length ? Math.max(...finiteDays) : 365;

  // For each task compute normalized components
  const scored = tasks.map((t) => {
    // urgency: if no dueDate -> 0. If past due (d < 0) -> treat as highest urgency (1)
    let urgency = 0;
    if (t.dueDate) {
      const days = daysBetween(t.dueDate, now);
      if (days <= 0) urgency = 1.0; // due today or past due -> max urgency
      else {
        // we invert: fewer days -> closer to 1
        const norm = normalize(days, minDays, Math.max(maxDays, 30));
        urgency = 1 - norm; // smaller days -> higher urgency
      }
    }

    // importance normalized: importance values expected 1..5
    const importance = normalize(t.importance || 0, 1, 5);

    // impact normalized 1..5
    const impact = normalize(t.impact || 0, 1, 5);

    // effort: we want smaller effort to produce higher priority. Normalize and invert.
    const effortVal = Math.min(t.effortHours || 0, EFFORT_CAP);
    const effortNorm = normalize(effortVal, minEffort, maxEffort); // 0 small, 1 large
    const effortScore = 1 - effortNorm;

    // base weighted score
    const baseScore =
      urgency * weights.urgency +
      importance * weights.importance +
      impact * weights.impact +
      effortScore * weights.effort;

    // blocked penalty reduces score
    const blockedPenalty = t.blocked ? weights.blockedPenalty : 0;

    // preference boost (e.g., starred tasks). use t.favorite boolean
    const preferenceBonus = t.favorite ? weights.preferenceBoost : 0;

    // final raw score (clamped 0..1)
    let rawScore = clamp(baseScore - blockedPenalty + preferenceBonus, 0, 1);

    // small tie-breaker: newest tasks slightly favored if scores equal
    const tieBreaker = 1 - normalize(new Date(t.createdAt).getTime(), 0, Date.now());

    return {
      ...t,
      __components: { urgency, importance, impact, effortScore, baseScore, blockedPenalty, preferenceBonus },
      score: rawScore,
      tieBreaker,
    };
  });

  return scored;
}

export default function SmartTaskAnalyzer() {
  const [tasks, setTasks] = useState(() => {
    try {
      const raw = localStorage.getItem("sta_tasks_v1");
      if (raw) return JSON.parse(raw);
    } catch (e) {}
    return SAMPLE_TASKS;
  });

  const [weights, setWeights] = useState(() => {
    try {
      const raw = localStorage.getItem("sta_weights_v1");
      if (raw) return JSON.parse(raw);
    } catch (e) {}
    return DEFAULT_WEIGHTS;
  });

  const [query, setQuery] = useState("");
  const [filterTag, setFilterTag] = useState("");
  const [sortBy, setSortBy] = useState("score");
  const [editing, setEditing] = useState(null);

  useEffect(() => {
    localStorage.setItem("sta_tasks_v1", JSON.stringify(tasks));
  }, [tasks]);

  useEffect(() => {
    localStorage.setItem("sta_weights_v1", JSON.stringify(weights));
  }, [weights]);

  const scored = useMemo(() => computeScores(tasks, weights), [tasks, weights]);

  // Filtering + searching
  const visible = scored
    .filter((t) => (!query || t.title.toLowerCase().includes(query.toLowerCase())))
    .filter((t) => (!filterTag || (t.tags || []).includes(filterTag)));

  // Sorting
  const sorted = [...visible].sort((a, b) => {
    if (sortBy === "score") {
      if (b.score === a.score) return b.tieBreaker - a.tieBreaker;
      return b.score - a.score;
    }
    if (sortBy === "due") {
      if (!a.dueDate && !b.dueDate) return b.createdAt.localeCompare(a.createdAt);
      if (!a.dueDate) return 1;
      if (!b.dueDate) return -1;
      return new Date(a.dueDate) - new Date(b.dueDate);
    }
    if (sortBy === "effort") return a.effortHours - b.effortHours;
    if (sortBy === "importance") return b.importance - a.importance;
    return 0;
  });

  // Add / Edit handlers
  function upsertTask(task) {
    setTasks((prev) => {
      if (task.id) return prev.map((p) => (p.id === task.id ? { ...p, ...task } : p));
      const id = "t" + Math.random().toString(36).slice(2, 9);
      return [{ ...task, id, createdAt: new Date().toISOString() }, ...prev];
    });
    setEditing(null);
  }

  function removeTask(id) {
    if (!confirm("Delete this task?")) return;
    setTasks((prev) => prev.filter((t) => t.id !== id));
  }

  function toggleFavorite(id) {
    setTasks((prev) => prev.map((t) => (t.id === id ? { ...t, favorite: !t.favorite } : t)));
  }

  // Small UI components inside the file
  const WeightSlider = ({ label, keyName, min = 0, max = 1, step = 0.01 }) => (
    <div className="flex items-center gap-3">
      <div className="w-28 text-sm">{label}</div>
      <input
        type="range"
        min={min}
        max={max}
        step={step}
        value={weights[keyName]}
        onChange={(e) => setWeights({ ...weights, [keyName]: parseFloat(e.target.value) })}
      />
      <div className="w-12 text-right text-xs">{weights[keyName].toFixed(2)}</div>
    </div>
  );

  // Task editor form
  function TaskForm({ initial = {}, onCancel, onSave }) {
    const [local, setLocal] = useState({
      title: "",
      dueDate: "",
      importance: 3,
      impact: 3,
      effortHours: 1,
      blocked: false,
      tags: "",
      notes: "",
      favorite: false,
      ...initial,
    });

    return (
      <div className="p-4 bg-white rounded shadow">
        <div className="flex flex-col gap-2">
          <input className="border p-2" placeholder="Title" value={local.title} onChange={(e) => setLocal({ ...local, title: e.target.value })} />
          <input
            type="date"
            className="border p-2"
            value={local.dueDate || ""}
            onChange={(e) => setLocal({ ...local, dueDate: e.target.value })}
          />
          <div className="flex gap-2">
            <label className="flex-1">Importance
              <input type="range" min={1} max={5} value={local.importance} onChange={(e) => setLocal({ ...local, importance: parseInt(e.target.value) })} />
            </label>
            <label className="flex-1">Impact
              <input type="range" min={1} max={5} value={local.impact} onChange={(e) => setLocal({ ...local, impact: parseInt(e.target.value) })} />
            </label>
          </div>
          <div className="flex gap-2">
            <input className="border p-2 flex-1" type="number" min={0} step={0.25} value={local.effortHours} onChange={(e) => setLocal({ ...local, effortHours: parseFloat(e.target.value) })} />
            <label className="flex items-center gap-2"><input type="checkbox" checked={local.blocked} onChange={(e) => setLocal({ ...local, blocked: e.target.checked })} /> Blocked</label>
            <label className="flex items-center gap-2"><input type="checkbox" checked={local.favorite} onChange={(e) => setLocal({ ...local, favorite: e.target.checked })} /> Favorite</label>
          </div>
          <input className="border p-2" placeholder="Tags (comma separated)" value={local.tags} onChange={(e) => setLocal({ ...local, tags: e.target.value })} />
          <textarea className="border p-2" rows={3} placeholder="Notes" value={local.notes} onChange={(e) => setLocal({ ...local, notes: e.target.value })} />

          <div className="flex gap-2 justify-end">
            <button className="px-3 py-1 border rounded" onClick={onCancel}>Cancel</button>
            <button
              className="px-3 py-1 bg-blue-600 text-white rounded"
              onClick={() => {
                // sanitize and pass
                const clean = {
                  ...local,
                  tags: typeof local.tags === "string" ? local.tags.split(",").map((s) => s.trim()).filter(Boolean) : local.tags,
                };
                onSave(clean);
              }}
            >
              Save
            </button>
          </div>
        </div>
      </div>
    );
  }

  // quick stats
  const avgScore = (scored.reduce((s, t) => s + t.score, 0) / Math.max(scored.length, 1)).toFixed(2);

  return (
    <div className="p-6 min-h-screen bg-slate-50">
      <div className="max-w-6xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <h1 className="text-2xl font-bold">Smart Task Analyzer</h1>
          <div className="text-sm text-gray-600">Avg score: <strong>{avgScore}</strong></div>
        </header>

        <section className="grid grid-cols-3 gap-6">
          {/* Left: Controls */}
          <div className="col-span-1 bg-white p-4 rounded shadow">
            <h2 className="font-semibold">Controls</h2>
            <div className="mt-3 space-y-3">
              <div className="flex gap-2">
                <input className="flex-1 border p-2" placeholder="Search tasks..." value={query} onChange={(e) => setQuery(e.target.value)} />
                <button className="px-3 py-1 border rounded" onClick={() => setEditing({})}>+ New</button>
              </div>
              <div className="flex gap-2">
                <select className="flex-1 border p-2" value={filterTag} onChange={(e) => setFilterTag(e.target.value)}>
                  <option value="">All tags</option>
                  {[...new Set(tasks.flatMap((t) => t.tags || []))].map((tg) => (
                    <option key={tg} value={tg}>{tg}</option>
                  ))}
                </select>
                <select className="border p-2" value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
                  <option value="score">Smart score</option>
                  <option value="due">Due date</option>
                  <option value="effort">Effort</option>
                  <option value="importance">Importance</option>
                </select>
              </div>

              <div className="pt-2">
                <h3 className="text-sm font-medium">Weights</h3>
                <div className="space-y-2 mt-2">
                  <WeightSlider label="Urgency" keyName="urgency" />
                  <WeightSlider label="Importance" keyName="importance" />
                  <WeightSlider label="Impact" keyName="impact" />
                  <WeightSlider label="Effort" keyName="effort" />
                  <div className="text-xs text-gray-600 mt-2">Blocked penalty: {weights.blockedPenalty.toFixed(2)}</div>
                  <input type="range" min={0} max={0.5} step={0.01} value={weights.blockedPenalty} onChange={(e) => setWeights({ ...weights, blockedPenalty: parseFloat(e.target.value) })} />
                </div>

                <div className="mt-4 text-xs text-gray-500">Tip: adjust weights to reflect your workflow (e.g., increase urgency close to deadlines).</div>
              </div>

              <div className="mt-4">
                <button className="px-3 py-1 rounded border" onClick={() => { setWeights(DEFAULT_WEIGHTS); }}>Reset weights</button>
                <button className="ml-2 px-3 py-1 rounded border" onClick={() => { setTasks(SAMPLE_TASKS); }}>Load sample tasks</button>
              </div>
            </div>
          </div>

          {/* Middle: Task list */}
          <div className="col-span-2 bg-white p-4 rounded shadow">
            <h2 className="font-semibold">Tasks ({sorted.length})</h2>
            <div className="mt-3 space-y-3">
              {editing && (
                <div className="mb-3">
                  <TaskForm
                    initial={editing.id ? editing : {}}
                    onCancel={() => setEditing(null)}
                    onSave={(t) => upsertTask({ ...editing, ...t })}
                  />
                </div>
              )}

              <div className="space-y-2">
                {sorted.map((t) => (
                  <div key={t.id} className="flex items-start gap-3 border p-3 rounded">
                    <div className="w-8 text-sm">
                      <button onClick={() => toggleFavorite(t.id)} title="Star">
                        {t.favorite ? "★" : "☆"}
                      </button>
                    </div>
                    <div className="flex-1">
                      <div className="flex justify-between items-start">
                        <div>
                          <div className="font-medium">{t.title}</div>
                          <div className="text-xs text-gray-500">{(t.tags || []).join(", ")} • {t.notes}</div>
                        </div>
                        <div className="text-right">
                          <div className="text-sm font-semibold">{(t.score * 100).toFixed(0)}%</div>
                          <div className="text-xs text-gray-500">Score</div>
                        </div>
                      </div>

                      <div className="mt-2 flex gap-2 text-xs">
                        <div className="px-2 py-1 border rounded">Due: {t.dueDate || "—"}</div>
                        <div className="px-2 py-1 border rounded">Importance: {t.importance}</div>
                        <div className="px-2 py-1 border rounded">Impact: {t.impact}</div>
                        <div className="px-2 py-1 border rounded">Effort: {t.effortHours}h</div>
                        {t.blocked && <div className="px-2 py-1 border rounded text-red-600">Blocked</div>}
                      </div>

                      <div className="mt-2 flex gap-2">
                        <button className="px-2 py-1 border rounded text-sm" onClick={() => setEditing(t)}>Edit</button>
                        <button className="px-2 py-1 border rounded text-sm" onClick={() => removeTask(t.id)}>Delete</button>
                        <button className="px-2 py-1 border rounded text-sm" onClick={() => alert(JSON.stringify(t.__components, null, 2))}>Inspect components</button>
                      </div>
                    </div>
                  </div>
                ))}
              </div>

              {sorted.length === 0 && <div className="text-gray-500">No tasks match the filters.</div>}
            </div>
          </div>
        </section>

        <footer className="mt-6 text-sm text-gray-600">
          Algorithm: normalized urgency (closer due date higher) + importance + impact + inverted effort (smaller effort higher). Blocked tasks get a penalty. Weights are adjustable.
        </footer>
      </div>
    </div>
  );
}
