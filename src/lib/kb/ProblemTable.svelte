<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem. causes.ts places a report by its tenant; the desk's one-day-move reports
  // (desk-notes.json) are not a month/day flip, so they leave their tenant's group and stand alone.
  type Problem = 'parser' | 'hint' | 'one-day move' | 'unplaced';
  let { ticket, tickets }: { ticket: number; tickets: Partial<Record<Problem, string>> } = $props();

  const rows = $derived(uniqueById(reports as Report[]));
  const merged = $derived(rows.filter((r) => r.mergedInto === ticket));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const flipped = $derived(merged.filter((r) => !notes.oneDayMove.includes(r.id)));
  const problems = $derived(
    [
      ...groupByCause(flipped, tenantOf).map((g) => ({ problem: g.cause as Problem, reports: g.reports })),
      ...(moved.length ? [{ problem: 'one-day move' as Problem, reports: moved }] : []),
    ].map((p) => {
      const days = p.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        ...p,
        tenants: new Set(p.reports.map((r) => r.tenant)).size,
        intakeAgrees: p.reports.filter((r) => r.meta.intakeTag === p.problem).length,
        first: days[0],
        last: days[days.length - 1],
      };
    }),
  );
  const exportRows = (reports as Report[]).filter((r) => r.mergedInto === ticket).length;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{problems.length}</strong> {problems.length === 1 ? 'problem' : 'problems'} ·
  {problems.map((p) => p.reports.length).join(' + ')} = {problems.reduce((n, p) => n + p.reports.length, 0)}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake agreed</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each problems as p (p.problem)}
      <tr>
        <td><code>{p.problem}</code></td>
        <td>{p.reports.length}</td>
        <td>{p.problem === 'one-day move' ? '—' : `${p.intakeAgrees} of ${p.reports.length}`}</td>
        <td>{p.tenants}</td>
        <td>{p.first} → {p.last}</td>
        <td>{#if tickets[p.problem]}<a href={tickets[p.problem]}>{tickets[p.problem]?.replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if exportRows !== merged.length}
  <p class="note">The export lists {exportRows} rows for #{ticket}; {exportRows - merged.length} repeat an id and are counted once.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
