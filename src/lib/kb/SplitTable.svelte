<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { CAUSES, causeOf, uniqueById, type Cause, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. causes.ts places a report by tenant; the reports the
  // desk read as a one-day move (desk-notes.json) are not a day/month swap, so they get their own row.
  let { ticket, issues }: { ticket: number; issues: Record<string, string> } = $props();

  const rows = $derived.by(() => {
    const merged = uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket);
    const by = new Map<string, Report[]>();
    for (const r of merged) {
      const key = notes.oneDayMove.includes(r.id) ? 'one-day shift' : (causeOf(r, tenantOf(r)) ?? 'unplaced');
      by.set(key, [...(by.get(key) ?? []), r]);
    }
    return [...by.entries()]
      .map(([problem, rs]) => ({
        problem,
        count: rs.length,
        tenants: new Set(rs.map((r) => r.tenant)).size,
        // Intake only knows the causes in causes.ts; for any other problem the column means nothing.
        tagged: CAUSES.includes(problem as Cause) ? rs.filter((r) => r.meta.intakeTag !== problem).length : '—',
      }))
      .sort((a, b) => b.count - a.count);
  });
  const total = $derived(rows.reduce((n, r) => n + r.count, 0));
</script>

<p class="sum"><strong>{total}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>intake tag differs</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.count}</td>
        <td>{r.tenants}</td>
        <td>{r.tagged}</td>
        <td>{#if issues[r.problem]}<a href={issues[r.problem]}>{issues[r.problem].replace(/.*github\.com\//, '').replace('/issues/', '#')}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
